# n8n Workflow: Video-Transkription aus Google Drive (chunked)

Dieser Workflow überwacht einen Google-Drive-Ordner, extrahiert und splittet das Audio großer Videos (40–200 MB+) mit `ffmpeg`, transkribiert jeden Chunk mit OpenAI Whisper, fügt die Teile zusammen und legt das Transkript als `.txt` im selben Ordner ab.

## Warum chunked?

Die OpenAI-Whisper-API akzeptiert max. **25 MB pro Request**. Darum:

1. Wir extrahieren nur das Audio (Video weglassen).
2. Wir kodieren zu Mono 16 kHz MP3 @ 64 kbit/s (≈ 0,5 MB pro Minute → 1 Stunde ≈ 30 MB).
3. Wir splitten zusätzlich in **10-Minuten-Segmente**, damit selbst mehrstündige Videos sicher unter dem Limit bleiben.

## Ablauf (12 Nodes)

1. **Google Drive Trigger** – pollt den Ordner jede Minute auf neue Dateien.
2. **Only Videos (IF)** – filtert `mimeType` auf `video/*` (so lösen hochgeladene `.txt`-Transkripte den Trigger nicht aus).
3. **Download Video** – lädt die Datei als Binary.
4. **Prepare Paths (Code)** – baut Arbeitsverzeichnis `/tmp/n8n-transcribe-<id>-<ts>/` und Pfade.
5. **Write Video to Disk** – schreibt das Binary nach `inputPath`.
6. **Extract & Split Audio (ffmpeg)** – `ffmpeg -vn -ac 1 -ar 16000 -b:a 64k -f segment -segment_time 600 ...` erzeugt `chunk_000.mp3`, `chunk_001.mp3`, …
7. **Parse Chunk List (Code)** – wandelt die `ls`-Ausgabe in n8n-Items (einen pro Chunk).
8. **Read Chunk** – lädt jede Chunk-Datei als Binary.
9. **Transcribe (Whisper)** – ruft OpenAI Whisper pro Chunk.
10. **Join Transcripts (Code)** – konkateniert die Texte in Reihenfolge und erzeugt die Ziel-`.txt`.
11. **Upload Transcript** – lädt die `.txt` in den ursprünglichen Drive-Ordner hoch.
12. **Cleanup Tmp** – `rm -rf` des Arbeitsverzeichnisses.

## Voraussetzungen

- **Selbst gehostetes n8n** (Execute-Command-Node ist in n8n Cloud nicht verfügbar).
- `ffmpeg` im n8n-Container installiert.
  - Offizielles Image (`n8nio/n8n`, Alpine-basiert):
    ```bash
    docker exec -u root -it <n8n-container> apk add --no-cache ffmpeg
    ```
    oder dauerhaft per Custom-Dockerfile:
    ```dockerfile
    FROM n8nio/n8n
    USER root
    RUN apk add --no-cache ffmpeg
    USER node
    ```
- Env-Var `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` (Default ist ok) und **kein** Eintrag `executeCommand` in `NODES_EXCLUDE`.
- Schreibrechte auf `/tmp` für den n8n-Prozess.
- Google-Drive-OAuth2-Credential (Scope: `drive` oder mindestens `drive.file`).
- OpenAI-API-Credential mit Zugriff auf `whisper-1`.

## Import & Setup

1. In n8n: *Workflows → Import from File* → `video-transcription-workflow.json`.
2. Platzhalter ersetzen:
   - `REPLACE_WITH_DRIVE_FOLDER_ID` (2×: Trigger + Upload) → Ordner-ID aus der Drive-URL.
   - Google-Drive- und OpenAI-Credentials verknüpfen.
3. Workflow aktivieren.

## Tuning

- **Chunk-Länge** (Node *Extract & Split Audio*): `-segment_time 600` = 600 s = 10 min. Bei sehr gesprächigem Audio ggf. auf 300 s reduzieren, um auch bei dichter Sprache unter 25 MB zu bleiben.
- **Audio-Qualität**: `-b:a 64k` reicht für Whisper. Noch kleiner geht mit `-b:a 48k`.
- **Sprache vorgeben** im Whisper-Node unter *Options → Language* (z. B. `de`) – spart Erkennungszeit und verbessert die Genauigkeit.
- **Größere Videos parallelisieren**: Im Node *Transcribe (Whisper)* unter *Settings → Execution* „Continue on fail" + Batch-Größen anpassen.

## Fehlerbehandlung

- **`ffmpeg: not found`** → ffmpeg im Container installieren (siehe oben).
- **`EACCES` beim Schreiben** → Arbeitsverzeichnis auf einen von n8n beschreibbaren Pfad ändern (in *Prepare Paths* die `workDir`-Konstante anpassen).
- **Leere Transkript-Datei** → *Parse Chunk List* wirft eine Exception mit dem `stderr` von ffmpeg, dort steht normalerweise die Ursache.
- **Dauer-Trigger durch hochgeladene `.txt`** → durch den IF-Node auf `video/*` bereits abgefangen.
