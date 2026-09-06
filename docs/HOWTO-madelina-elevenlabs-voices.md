# Madelina Paradis — ElevenLabs voices, workflows, and server restarts

Session log + reference for the two custom voices and two ComfyUI workflows
built for the Madelina Paradis salon project (2026-09-06). Read this before
touching either workflow or either voice again.

## The two workflows

| Workflow (ComfyUI library name) | Purpose | Nodes |
|---|---|---|
| `Madelina.json` | Marie's on-camera answer — photo → lip-synced HeyGen video | `LoadImage` → `ElevenLabs Voice Selector` (Marie) → `ElevenLabs Text to Speech` (script box) → `HeyGenTalkingPhotoNode` → `SaveVideo` |
| `Madelina-Interviewer.json` | Off-camera customer question — audio only, no photo | `ElevenLabs Voice Selector` (CiCi) → `ElevenLabs Text to Speech` (question box) → `Save Audio (MP3)` |

Both are edited in the ComfyUI canvas: open the workflow from the sidebar,
click the `text` field on the `ElevenLabs Text to Speech` node, type the
line, hit Run. No file edits, no asking an agent to change it.

**Output paths:** `ComfyUI/output/madelina-paradis/...` (renamed from
`madelina-preview/` earlier this session — see `RULES-output-naming.md`).

## The two voices — what they are, and which model to use

Both are added to ComfyUI's `ElevenLabs Voice Selector` dropdown via a local
edit to `comfy_api_nodes/nodes_elevenlabs.py`'s `ELEVENLABS_VOICES` list
(stock ComfyUI only ships ~20 generic voices — custom account voices have to
be added by hand, there's no "enter a voice_id" widget). **Any edit to that
list requires a full ComfyUI restart** to take effect — see the restart
section below.

### Marie (salon owner, on-camera)

- **voice_id:** `CiEU7xqSY4lTz8jxOJhz`
- **Model to use:** `eleven_v3` (NOT `multilingual_v2`)
- **Created via:** ElevenLabs Voice Design (`POST /v1/text-to-voice/design`
  → `POST /v1/text-to-voice` — note: **no `/create` suffix**, that 404s).
  Description used: *"Middle-aged West African woman, about 50 years old,
  warm and slow-paced, Togolese francophone West African accent speaking
  English, friendly salon-owner tone."*
- **Full history, including two superseded attempts, in**
  `shot-builder/workspace/projects/madelina-paradis/people/MARIE.md`
  (gitignored — read it directly, not via git log).

**Do not regenerate this voice from the description again expecting the
same result.** Voice Design is NOT deterministic — re-running `/design`
with an identical description produces a *different* voice each time. If
you need this exact voice, use `voice_id CiEU7xqSY4lTz8jxOJhz` directly,
never re-derive it from the description text.

An earlier attempt cloned Marie's voice with Instant Voice Clone
(`wqYEydQOTCDXabZcTuzP`, from 4 real audio samples) instead of Voice
Design. **Don't reuse that approach for accented voices** — cloning from a
short sample lost both the accent and the original slower pace. Voice
Design (description-driven) held onto both; Instant Voice Clone
(sample-driven) did not, at least at ~25s of reference audio.

### CiCi (off-camera interviewer, audio only)

- **voice_id:** `fLQhkOW7F9KVKAjYCbhr`
- **Model to use:** `eleven_multilingual_v2` (NOT `eleven_v3`)
- **Source:** an existing ElevenLabs Voice Library voice ("CiCi - Sweet,
  Loving and Accepting", category `professional`), picked by ear directly
  from ElevenLabs — not cloned, not designed, already lives on the
  account. No transfer/clone step was ever needed for this one.
- **Full history in**
  `shot-builder/workspace/projects/madelina-paradis/people/INTERVIEWER.md`.

**The model choice matters and is not interchangeable between these two
voices.** The first version of `Madelina-Interviewer.json` used `eleven_v3`
(copied from Marie's node without checking) and it audibly flattened
CiCi's character — confirmed by cross-referencing Bench Studio's own
generation history (`bench-studio-public/data/bench.db`, `generations`
table, rows with `fLQhkOW7F9KVKAjYCbhr` in `payload_json`), which showed
every approved take used `multilingual_v2`. When adding a new custom voice
to a workflow, **check what model actually produced the accepted output
before assuming a default** — don't copy settings from a different voice's
node.

## Restarting ComfyUI after editing `nodes_elevenlabs.py`

The `comfyui` MCP's own `restart_comfyui` tool **does not work reliably in
this environment** — it relaunched with the wrong Python interpreter
(`/opt/local/bin/python3`, MacPorts) instead of ComfyUI's actual conda env,
which is missing `sqlalchemy` and crashes on boot. Confirmed dead on
2026-09-06. Use this instead:

```bash
pkill -f "condaenv/bin/python main.py"
# wait for the port to free (poll `pgrep -f "condaenv/bin/python main.py"`)
cd "/Users/shahramsedehi/Documents/Github Local/ComfyUI"
nohup "/Users/shahramsedehi/Documents/Github Local/ComfyUI/condaenv/bin/python" main.py --cpu > /tmp/comfyui_restart.log 2>&1 &
disown
# poll curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8188/system_stats
# until it returns 200
```

The correct interpreter is always
`ComfyUI/condaenv/bin/python` — never bare `python3`.

## Auto-generated layout can visually hide nodes

`save_workflow`'s API→UI auto-conversion lays out nodes using only their
declared `size` in the JSON — but `LoadImage` (and similar preview-heavy
nodes) renders a large image thumbnail in the canvas that isn't reflected
in that declared size. A node placed directly below a `LoadImage` node by
the auto-layout can end up visually hidden underneath the thumbnail, even
though the underlying workflow JSON is completely correct (confirmed
2026-09-06 — `ElevenLabsVoiceSelector` was "missing" from the canvas but
present and correctly wired in every read of the saved file; it was
rendering underneath the photo preview). If a node seems to have vanished
after a save, check the actual saved JSON via `get_workflow` before
assuming anything is broken — it's very likely just an overlap, fixable by
giving `LoadImage`-type nodes more vertical room (e.g. `size: [280, 320]`)
or moving nodes below them further down.

## Adding another custom account voice — the pattern

1. Confirm the voice_id actually resolves against the account's own key
   (`GET https://api.elevenlabs.io/v1/voices/<id>` with the `xi-api-key`
   header from `custom_nodes/comfy-api-liberation/api_keys.json`) — don't
   assume a voice_id from another tool/pipeline (e.g. fal.ai) is reachable
   directly; fal's ElevenLabs calls go through fal's OWN backend credentials,
   not the user's personal account, and a fal-only voice_id will 404
   against the direct API.
2. Add a `(voice_id, display_name, gender, accent)` tuple to
   `ELEVENLABS_VOICES` in `comfy_api_nodes/nodes_elevenlabs.py`, right after
   the existing custom-voices comment block.
3. Restart ComfyUI (see above).
4. Confirm it via `node_info` on `ElevenLabsVoiceSelector` (or the canvas
   dropdown directly) before wiring anything to it.
5. Check what TTS *model* actually produced the reference audio for this
   voice before picking one for the new node — see the CiCi lesson above.

## Related docs in this folder

- `RULES-comfyui-mcp-composite-inputs.md` — how to author DynamicCombo /
  Autogrow composite inputs (`speech.audio`, `files.audio0`, etc.) via the
  `comfyui` MCP without guessing JSON shapes.
- `RULES-output-naming.md` — the `<client-slug>/<file>` output folder
  convention that lets Bench Studio's importer auto-detect clients.
- `HOWTO-fix-madelina-heygen-audio-wiring.md` — the original HeyGen
  DynamicCombo wiring fix (superseded by direct authoring once the correct
  shape was confirmed, but still useful background).
