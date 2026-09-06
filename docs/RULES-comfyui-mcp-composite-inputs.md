# Project rules: composite/DynamicCombo inputs and the comfyui MCP

Adopted 2026-09-02, after the `HeyGenTalkingPhotoNode.speech` wiring bug;
**corrected the same day** once the actual root cause was found (see
`PRD-comfyui-mcp-composite-input-handling.md`'s "CORRECTION" section for the
full story). The rule below reflects the corrected understanding — the
original version of this file wrongly said composite inputs can't be authored
via `create_workflow` at all. They can; you just need the right shape.

## The rule

**A composite input (`DynamicCombo`, `Autogrow`, nested `MatchType`) is
represented in API-format JSON as a FLAT top-level dotted key for any linked
sub-field — never a nested dict.** For `HeyGenTalkingPhotoNode.speech` in
`"audio"` mode, the correct shape is:

```json
{ "speech": "audio", "speech.audio": ["<source_node_id>", <output_index>] }
```

**NOT** `{"speech": {"speech": "audio", "audio": [...]}}` (nested dict) —
that shape looks more intuitive for a dict-typed input but is wrong and
fails at execution with a bare `TypeError` (`unexpected keyword argument
'speech.audio'` or `missing required positional argument 'speech'`),
**even though `create_workflow (action:"validate")` reports it as valid.**
Validation passing is still not proof of correctness for a composite input —
that part of the original finding holds.

**When authoring via `create_workflow (action:"modify")`:** use the `connect`
operation with the dotted key as a literal `input_name` string, e.g.
`{"op":"connect", "source_id":"2", "output_index":0, "target_id":"5",
"input_name":"speech.audio"}`. This works — confirmed by direct test,
producing byte-identical JSON to a real ComfyUI canvas export and executing
successfully. Composite inputs are NOT off-limits for direct authoring; only
the nested-dict shape is wrong.

## How to recognize a composite input before you wire it

Before wiring any input on a node you haven't wired before (especially
partner/API nodes like HeyGen, or any node with a "mode"-style dropdown that
reveals different sub-fields):

1. Check the node's Python schema — for local ComfyUI, that's usually
   `comfy_api_nodes/nodes_<vendor>.py` in the ComfyUI repo. Look for
   `IO.DynamicCombo.Input(...)` or `IO.Autogrow.Input(...)` in
   `define_schema()`.
2. If present, that input is composite — use the flat-dotted-key shape below,
   not a nested dict.
3. If you can't check the source (a hosted/cloud node), treat any input whose
   description mentions "drive with X or Y" / a mode selector as a
   composite-input suspect.

## The correct path, in order of preference

1. **If you already know the composite input's flat-dotted-key pattern**
   (documented here, or found previously) — author it directly with
   `create_workflow (action:"modify")`'s `connect` op, using the dotted key
   as the literal `input_name` (e.g. `input_name: "speech.audio"`), plus a
   `set_input` for the mode-selector literal (e.g. `speech: "audio"`). No
   canvas needed. This is now the default — reach for it first.
2. **If you DON'T yet know the pattern for a new composite input** (a
   different `DynamicCombo`/`Autogrow` you haven't wired before) — build it
   once in ComfyUI's own free local canvas (`http://127.0.0.1:8188`, the
   plain node-graph editor, no subscription needed) and export via
   **Workflow → Export (API Format)**. Read the resulting JSON to learn the
   correct flat-dotted-key shape for that specific input, then use path 1 for
   every future run of that same input. This is a one-time discovery step per
   composite input type, not a permanent requirement.
3. Either way, **always validate AND still be skeptical of a passing
   validation** for composite inputs — `create_workflow (action:"validate")`
   reports both the correct and incorrect shapes as valid, so validation
   alone doesn't confirm you got the shape right. If in doubt, do a cheap
   real test run before a full/expensive one (or check for a ComfyUI
   node-cache hit — matching a previously-successful run's exact input
   signature is strong independent confirmation, as it was here).

## What Claude should do differently going forward

- Composite inputs are normal to author directly via `create_workflow` —
  don't default to telling the user they need the canvas unless the specific
  flat-dotted-key pattern for that input is genuinely unknown yet.
- Never use a nested-dict shape (`{"speech": {"audio": [...]}}`) for a
  composite input under any circumstances — always flat top-level dotted
  keys for the linked sub-field, plus a plain literal for the mode selector.
- Document each new composite input's correct pattern here (in the Reference
  section below) once discovered, so it doesn't need re-discovering via the
  canvas next time.
- If in doubt and about to run something billed (HeyGen Avatar tiers are
  billed per second), ask before enqueueing — a wrong guess costs a real
  render attempt, not just time.

## Reference

- Full technical background and phased roadmap for actually fixing this in
  `comfyui-mcp` itself (not just working around it): see
  `PRD-comfyui-mcp-composite-input-handling.md` in this same folder.
- The exact schema for `HeyGenTalkingPhotoNode.speech` (as of this writing,
  `comfy_api_nodes/nodes_heygen.py`): two options, `"script"` (widget-only:
  `text`, `voice`, `custom_voice_id`, `voice_speed` — no links, safe to author
  via `create_workflow`) and `"audio"` (composite: exactly one sub-field,
  `audio`, an `IO.Audio.Input` requiring the flat-dotted-key `speech.audio`
  pattern documented above).
- A ready-to-run, frozen template for the `"audio"` mode case:
  `templates/heygen-talking-photo-audio-template.json` (see
  `templates/README.md` for usage) — swap the two filenames and run, no
  canvas or `create_workflow` authoring needed.
- This exact pattern (flat dotted keys, no nested dict, server-computed
  `dynamic_paths` from the node's own schema) is **not documented anywhere
  else** — not `docs.comfy.org`, not the relevant `comfy-cli` GitHub issue,
  not any third-party converter's README, not even ComfyUI's own source code
  comments. It was reverse-engineered here via a live canvas export +
  independent confirmation. Worth contributing upstream (PRD §6) since
  apparently nobody else has written it down either.
