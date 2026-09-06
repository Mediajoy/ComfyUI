# Project rule: ComfyUI output folder naming = Bench Studio client slug

Adopted 2026-09-02, when setting up import of ComfyUI outputs into
[Bench Studio](../../bench-studio-public/) for review (starring, notes,
outcome tracking) — capabilities ComfyUI's own Assets panel doesn't have.

## The rule

**The first path segment of every `SaveVideo`/`SaveImage` node's
`filename_prefix` must be the exact Bench Studio client slug for that
project.** Optionally add a second segment for a series/sub-category:

```
<client-slug>/<filename>                  → client only
<client-slug>/<series-slug>/<filename>    → client + series
```

Example (Madelina Paradis, no series): `filename_prefix: "madelina-paradis/marie-a1-newavatar"`
→ writes to `ComfyUI/output/madelina-paradis/marie-a1-newavatar_00001_.mp4`.

**Why this exact string and not something readable-but-different** (like the
`madelina-preview` folder used before this rule): Bench Studio's importer
detects which client a ComfyUI file belongs to purely from this folder name —
there is no separate mapping file to keep in sync. Get the slug wrong and the
file imports as untagged (still importable, just not auto-associated with
the right client).

## How to find the correct slug

Query Bench Studio's own client list rather than guessing or reusing an old
folder name:
```bash
sqlite3 "/Users/shahramsedehi/Documents/Github Local/bench-studio-public/data/bench.db" \
  "SELECT DISTINCT client FROM generations WHERE client IS NOT NULL;"
```
As of this writing, the only client is `madelina-paradis`. If a project has
never had a Bench Studio generation yet, use the same slug format
Bench itself would generate (lowercase, spaces/underscores → hyphens, e.g.
`normalizeSlug()` in `bench-studio-public/server/db.mjs:22`) — matching that
function's output exactly is what keeps a brand-new client from becoming
`"Grace Church"` / `"grace-church"` / `"gracechurch"` as three different tags.

## What this doesn't cover

- Files with **no** client segment (loose in `output/`, or from before this
  rule existed) are still importable into Bench Studio — they just show up
  untagged, and you assign a client manually at import time.
- This rule is about the **output** folder only. Input files (photos, audio)
  in `ComfyUI/input/` follow no such convention — name them however's useful
  for finding them again.

## Reference

- The importer that reads this convention: see the
  "Import ComfyUI outputs into Bench Studio" plan/implementation in
  `bench-studio-public/server/server.mjs` (`/api/comfyui/pending` and
  `/api/comfyui/import` routes).
- `RULES-comfyui-mcp-composite-inputs.md` in this same folder — a different,
  unrelated concern (composite input wiring), kept separate on purpose.
