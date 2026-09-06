# HOWTO: manually fix the HeyGen audio wiring in `Madelina.json`

This is the concrete, one-time manual fix for the broken `speech` input on
`Madelina.json`, following the rule in
`RULES-comfyui-mcp-composite-inputs.md`: composite inputs (this node's
`speech` `DynamicCombo`) must be wired in ComfyUI's own free local canvas, not
authored via `create_workflow`/hand-edited JSON.

**No Comfy Cloud subscription needed for this** — this is the plain node-graph
canvas, not the paid AI sidebar-panel agent. ComfyUI's local server is at
`http://127.0.0.1:8188`.

## What's actually wrong (confirmed by screenshot, 2026-09-02)

The file at
`/Users/shahramsedehi/Documents/Github Local/ComfyUI/user/default/workflows/Madelina.json`
was built by an earlier Claude Code agent hand-editing the workflow JSON
directly, not by clicking/dragging in the canvas. When opened, the `HeyGen
Talking Photo` node's `speech` row renders as the literal text **"[object
Object]"** instead of a real dropdown — that's the canvas telling you it
received a raw JS object it doesn't know how to unpack into its normal
interactive widget. There is also no separate `Load Audio` box visible on the
canvas at all (it should have one, feeding the HeyGen node's audio input).

**This means the node is genuinely broken, not just missing one wire** — the
fix is to delete that HeyGen node and re-add a fresh one by clicking, which
gives you a real, working dropdown, then wire it up from scratch. Reconnecting
a wire to the existing broken node will not work.

## Steps

1. **Open ComfyUI's local web UI**: `http://127.0.0.1:8188` in your browser.

2. **Open the `Madelina` workflow.** Small folder/menu icon top-left (or
   "Workflow" in a top menu bar) → click "Madelina" in the list.

3. **Delete the broken HeyGen box.** Click once on the "HeyGen Talking Photo"
   box's title bar to select it (it gets a highlight border), then press
   **Delete** (or Backspace). This only removes that one box — `Load Image`
   and `Save Video` stay; their wires to it will go dangling for a moment,
   that's expected.

4. **Add a fresh HeyGen Talking Photo node.** Right-click empty canvas where
   the old box was. In the menu that appears, either browse **partner →
   video → HeyGen → HeyGen Talking Photo**, or type `HeyGen Talking Photo`
   into the search box at top and click the match.

5. **Wire the image in.** Drag from `Load Image`'s `IMAGE` output dot
   (top-right of that box) to the new HeyGen box's `image` input dot.

6. **Set the speech source.** On the new box you should see a real, working
   dropdown this time — click it and choose **"audio"** (not "script"). A
   new empty input dot labeled `audio` appears on the box once you do.

7. **Add an audio loader.** Right-click empty canvas again, search for **Load
   Audio**, click it to add that box.

8. **Load the file into it.** On the new Load Audio box, click its file field
   and select **`marie-a1-newavatar-warm.mp3`**. It should already be in
   ComfyUI's input folder (generated earlier this session — the fal.ai
   ElevenLabs TTS reconstruction of the A1 line for the new avatar). If it's
   not in the list, flag it — the exact path needs tracking down.

9. **Wire audio in.** Drag from Load Audio's `AUDIO` output dot to the new
   HeyGen box's `audio` input dot (the one that appeared in step 6).

10. **Reconnect the output.** Drag from the new HeyGen box's `VIDEO` output
    dot to `Save Video`'s `video` input dot (broken when you deleted the old
    box in step 3).

11. **Re-set the other widgets** on the new HeyGen box to match the original:
    resolution `720p`, aspect ratio `1:1`, expressiveness `low` (these reset
    to node defaults on a fresh node — set them back).

12. **Export in API format.** Find **Workflow → Export (API Format)** in the
    top menu (look specifically for wording that says "API" — there's
    usually a plain "Export"/"Save" too, which produces the wrong, UI-only
    format). Save the resulting `.json` somewhere findable, e.g.
    `~/Documents/Github Local/ComfyUI/user/default/workflows/Madelina-api.json`.

13. **Tell me the file path** (or paste its contents) once exported. I'll
    read it, confirm the `speech.audio` link is present and well-formed, then
    feed it directly to `enqueue_workflow` — no further authoring needed on my
    end, since the canvas already did the correct serialization.

Tip: send a screenshot after step 6 (once the audio dropdown/socket appears)
if you want a sanity check before continuing.

## Why this is the right way (not a workaround)

The canvas's own JavaScript is the actual code that defines what a
`DynamicCombo` looks like and how it should be exported — it can't get this
wrong, because it's the same code path a real user always uses. Any
JSON-authoring tool (this MCP's `create_workflow`, or me hand-editing the
file) has to *reverse-engineer* that logic, which is exactly where the
original bug came from. This one-time manual step sidesteps the whole
problem for this specific run, while the PRD
(`PRD-comfyui-mcp-composite-input-handling.md`) tracks the longer-term fix so
this stops requiring a manual step at all.

## Turning this into a permanent template (do once, reuse forever)

Once the export in step 12 is confirmed working (a real render succeeds),
save that exported API-format JSON as a frozen template:
`ComfyUI/docs/templates/heygen-talking-photo-audio-template.json`. The only
two things that ever change between future avatar runs are the `Load Image`
and `Load Audio` filenames — both are plain literal widget values, not part
of the composite `speech` structure, so they are 100% safe to swap
programmatically (via `create_workflow (action:"modify")` or direct JSON
edit) without touching the frozen composite wiring at all. Future runs should
start from this template and swap only those two filenames — no canvas, no
manual dragging, ever again for this node type.
