# n8n Workflow: Video-Transkription aus Google Drive

Dieser Workflow überwacht einen Google-Drive-Ordner, transkribiert neu hochgeladene Videos mit OpenAI Whisper und legt das Transkript als `.txt` im selben Ordner ab.

## Ablauf

1. **Google Drive Trigger** – pollt den Zielordner jede Minute auf neu erstellte Dateien.
2. **Only Videos (IF)** – filtert auf `mimeType` beginnend mit `video/`.
3. **Download Video** – lädt die Datei als Binary herunter.
4. **Transcribe (Whisper)** – sendet das Binary an OpenAI Whisper (`audio/transcribe`).
5. **Prepare Transcript File (Code)** – baut aus dem Transkript eine Textdatei mit dem Namen des Originalvideos (`<video>.txt`).
6. **Upload Transcript** – lädt die `.txt` in denselben Drive-Ordner hoch.

## Import

1. In n8n: *Workflows → Import from File* → `video-transcription-workflow.json` auswählen.
2. Platzhalter ersetzen:
   - `REPLACE_WITH_DRIVE_FOLDER_ID` (2×: Trigger + Upload) → Ordner-ID aus der Drive-URL (`https://drive.google.com/drive/folders/<ID>`).
   - Credentials für **Google Drive** und **OpenAI** verknüpfen (werden nach dem Import automatisch abgefragt).
3. Workflow aktivieren.

## Voraussetzungen

- Google-Drive-OAuth2-Credential in n8n (Scope: `drive` oder mindestens `drive.file` + `drive.readonly` für den Zielordner).
- OpenAI-API-Credential mit Zugriff auf das Whisper-Modell.
- Video-Dateigröße ≤ 25 MB (Whisper-Limit). Für längere Videos zusätzlich einen Split-Schritt (z. B. `ffmpeg`) vorschalten.

## Hinweise

- Der Trigger reagiert nur auf **neu angelegte** Dateien – bestehende Videos werden nicht nachträglich verarbeitet.
- Falls der Transkript-Upload den Trigger erneut auslösen würde: Der `IF`-Filter blockt alles, was nicht `video/*` ist, daher werden `.txt`-Uploads ignoriert.
- Sprache/Prompt/Response-Format lassen sich im Whisper-Node unter *Options* anpassen.
