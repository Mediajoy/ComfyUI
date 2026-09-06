# Frozen composite-input workflow templates

Each `.json` here is a verified-working ComfyUI API-format workflow, kept
frozen so a composite (`DynamicCombo`/`Autogrow`) input's tricky wiring never
needs to be re-derived from the canvas twice. See
`../RULES-comfyui-mcp-composite-inputs.md` for the general rule these follow.

## `heygen-talking-photo-audio-template.json`

Drives `HeyGenTalkingPhotoNode` with your own audio (not HeyGen's
script/TTS). Confirmed working 2026-09-02 — both as a canvas export and as an
independent `create_workflow`-authored version, each executed successfully
via `enqueue_workflow`.

**Frozen — do not change the shape of these two keys on node `"5"`:**
```json
"speech": "audio",
"speech.audio": ["2", 0]
```
This is the one verified-correct representation of the `speech` composite
input in `"audio"` mode: a flat top-level string for the mode selector, plus
a separate flat top-level dotted key for the link — never a nested dict.
Changing this back to a nested dict (`{"speech": {"audio": [...]}}`) will
silently reproduce the original bug (a `TypeError` at execution, even though
`create_workflow (action:"validate")` will still report it as valid).

**Safe to change for every run** — all plain literals, no composite
structure involved:
- Node `"2"` (`LoadAudio`) → `inputs.audio`: your audio filename
- Node `"6"` (`LoadImage`) → `inputs.image`: your photo filename
- Node `"4"` (`SaveVideo`) → `inputs.filename_prefix`: your output path (see
  naming convention below)
- Node `"5"`'s `resolution` / `aspect_ratio` / `expressiveness` / `seed`

## Output naming convention

`filename_prefix` on the `SaveVideo` node doubles as a folder path — anything
before a `/` becomes a subfolder under `ComfyUI/output/`. Use your **Bench
Studio client slug** as that first path segment (e.g. `madelina-paradis`, not
a made-up name like `madelina-preview`) — this is what lets Bench Studio's
importer auto-detect which client a render belongs to with zero manual
mapping. See `../RULES-output-naming.md` for the full convention, including
how to add a series subfolder.

## How to use

1. Read this template's JSON.
2. Replace the `REPLACE_ME_*` placeholder values with your actual filenames
   (both files must already exist in ComfyUI's `input/` folder — confirm with
   a filesystem check, not just by referencing the name) and your Bench
   client slug for `filename_prefix`.
3. Feed the result straight to `enqueue_workflow (action:"enqueue")`. No
   canvas, no `create_workflow` authoring needed — this is a complete,
   ready-to-run graph.
4. This is a billed HeyGen call — confirm cost with the user before
   enqueueing (Avatar tier is ~$0.0715/second of audio).
