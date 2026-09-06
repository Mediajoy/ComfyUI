# PRD: Composite/Dynamic Input Handling for `comfyui-mcp`

**Status:** Draft for review — **root cause corrected 2026-09-02, see below**
**Owner:** TBD
**Target repo:** `artokun/comfyui-mcp`
**Related upstream:** `Comfy-Org/ComfyUI` (V3 schema), `Comfy-Org/ComfyUI_frontend`
**Trigger bug:** `create_workflow` / `enqueue_workflow` fail to wire `HeyGenTalkingPhotoNode.speech` (an `IO.DynamicCombo`), with `TypeError: unexpected keyword argument 'speech.audio'` or `TypeError: missing required positional argument 'speech'`

---

## CORRECTION (2026-09-02) — the tool is NOT structurally incapable; the original attempts used the wrong shape

The original diagnosis in §1/§2 below claimed the API-format wire representation is "structurally incapable" of expressing a composite input. **This was wrong**, discovered empirically:

1. `Madelina.json` was rebuilt by hand in ComfyUI's own live canvas (not the paid AI panel — the plain free node editor) and exported via "Export (API Format)". The exported JSON for `HeyGenTalkingPhotoNode` was:
   ```json
   { "speech": "audio", "speech.audio": ["2", 0], ... }
   ```
   — a **flat string** for the mode selector, plus a **separate top-level dotted key** for the link. No nested dict at all.
2. This ran successfully via `enqueue_workflow` on the first try — no `TypeError`.
3. As a direct test, the *exact same shape* was then authored from scratch using `create_workflow (action:"modify")`'s own `connect` operation — `{"op":"connect", "source_id":"2", "output_index":0, "target_id":"5", "input_name":"speech.audio"}` — passing `"speech.audio"` as a literal `input_name` string. **This produced the identical correct JSON and also ran successfully** (confirmed via ComfyUI's node-level cache reusing the prior run's output — same input signature).

**Actual root cause:** every prior failing attempt used a **nested dict** shape — `{"speech": {"speech": "audio", "audio": [link]}}`, or a nested dict plus a redundant top-level dotted key together. That shape is genuinely wrong and produces the `TypeError`s. The **correct** shape — flat top-level dotted key, no nesting — was representable by `create_workflow` all along; nobody had tried it because the nested-dict shape looked more intuitively "correct" for a dict-typed input.

**What stays true from the original investigation:**
- `create_workflow (action:"validate")` still does not catch a wrong (nested-dict) shape as invalid — it validated both the broken and correct shapes as "valid," so validation still isn't proof of correctness for composite inputs.
- The live-canvas-export path is still how the correct shape was *discovered* — it remains a good way to find the right pattern for a composite input you haven't wired before.
- Once the correct flat-dotted-key pattern for a given composite input is known, however, **it does not need to be re-derived from the canvas every time** — `create_workflow`'s own authoring operations (specifically `connect` with a dotted `input_name`) can reproduce it directly. The "canonical compiler" work in Phase 2 below is really about *recording and reusing known-correct dotted-key patterns per composite input*, not building new JSON-serialization logic from scratch.

**Practical implication:** the rest of this PRD's phased plan is still worth doing (a registry of which inputs are composite, validation that actually catches the nested-dict mistake, structured authoring so nobody has to know the dotted-key convention) — but the risk profile is much lower than originally assessed. This is a known, learnable pattern (flat dotted keys, no nesting), not a wire-format dead end requiring frontend-parity reverse-engineering.

---

## 1. Purpose

Fix the immediate DynamicCombo wiring failure, but do it by closing the *class* of bug, not the *instance*. Today, `comfyui-mcp` treats all node inputs as either a literal or a single link. That model is correct for the majority of ComfyUI nodes, but it is structurally incapable of representing V3 **composite input types** — `DynamicCombo`, `Autogrow`, nested `MatchType` groups — where a single input is a dict that may mix literals and links. Any future custom node using these V3 patterns will reproduce this exact failure mode. This PRD defines the architecture and phased plan to make composite inputs a first-class, tested, and validated concept in the MCP rather than a recurring support ticket.

## 2. Background (what we confirmed)

- `IO.DynamicCombo` is a real, current V3 schema feature: a dropdown that shows/hides sub-inputs based on the selected option. At execution time, the whole thing arrives at `execute()` as **one dict**, keyed by the combo's own input ID (e.g. `mode["mode"]`, `mode["width"]`) — never as separate top-level kwargs.
- The ComfyUI "API format" prompt JSON has exactly two shapes for an input value: a **literal**, or a **single link** `[node_id, output_index]`. There is no representation for "part of this dict is a literal, part is a link." This is a real limitation of the wire format itself, not an oversight in any one MCP.
- This is a known-enough problem that ComfyUI's own frontend has an internal marker for it — `COMFY_DYNAMICCOMBO_V3` — that dedicated UI-format→API-format converters (e.g. `SethRobinson/comfyui-workflow-to-api-converter-endpoint`) explicitly special-case, alongside reroute nodes and bypassed nodes.
- `comfyui-mcp`'s `validate_workflow` only checks JSON shape, not runtime signature compatibility — so it reports "valid" on graphs that will fail at execution.
- `comfyui-mcp` already ships a relevant primitive: `/comfy:convert <file>` converts between UI-format and API-format workflows. Building via the live canvas (or the panel's `panel_*` live-graph tools) and converting is a working manual escape hatch today — but it's not integrated into `create_workflow` / `modify_workflow`, so agents don't know to reach for it.
- A comparable project, `dreamrec/ComfyPilot`, shipped "strict live-catalog workflow validation for modern V3 widgets, DynamicCombo/Autogrow inputs" as of v1.9.0 — i.e., a competing tool has already built at least the validation half of what's proposed here. Worth treating as prior art, not just a fallback option (see §6).

## 3. Goals

1. An agent (or human) using `comfyui-mcp` can wire any V3 composite input — DynamicCombo today, Autogrow/MatchType by extension — through the standard tool surface, without hand-authoring nested dicts or dotted keys.
2. Invalid composite wiring is caught **before** the job is enqueued, with an error that names the node, the input, and what shape was expected — not a bare `TypeError` from deep in the executor.
3. The fix generalizes: adding support for a new composite pattern (e.g. `Autogrow`) should not require re-deriving this from scratch.
4. No regression to the large surface of plain-typed nodes that already work via literal/link JSON.

## 4. Non-Goals

- Rewriting ComfyUI's own wire format (that's upstream `Comfy-Org/ComfyUI` territory; we work within it).
- Full parity with the live canvas for every possible V3 pattern in one release — we scope to DynamicCombo first, generalize the mechanism, then extend coverage.
- Building a competing product to ComfyPilot — see §6 for how we relate to it.

## 5. Guiding Architectural Principles

These are the principles the phased plan is built to enforce. Treat this section as the thing to check new code against, not just this one bug.

**5.1 — One canonical serializer, not N ad hoc ones.**
Today, "build correct JSON" logic is implicitly duplicated across `create_workflow`, `modify_workflow`, and whatever a template author assumed. There should be exactly one function that takes a "logical" node input value (literal, link, or composite-with-links) and emits valid API-format JSON for it. Every tool that writes workflow JSON calls this function. This mirrors the compiler front-end/back-end split: authoring tools produce an intermediate representation (IR); one back-end compiles IR to wire format. If the compiler needs to special-case `COMFY_DYNAMICCOMBO_V3`, it does it once.

**5.2 — Classify before you author.**
Before any tool tries to set a value on a node input, look up that input's type from a live `/object_info` scan and classify it: `primitive` (literal-or-link representable) vs `composite` (DynamicCombo/Autogrow/nested MatchType — needs the compiler path). Composite classification should be cached but re-validated whenever a custom node pack is installed or updated, since dynamic nodes can change their schema at runtime.

**5.3 — Fail closed, with a semantic error, before enqueue.**
`validate_workflow` should stop being purely structural. For every input, resolve the *actual* backend argument shape via `inspect.signature` on the node's Python class (or the V3 `Schema` object) and diff it against what the JSON provides. If a composite input is missing a required sub-key, or a primitive input received a dotted key, reject with a specific message (`"speech" expects a dict with keys {mode, audio}; got flat key "speech.audio"`) rather than letting it reach the queue.

**5.4 — Prefer driving the real frontend over re-deriving its logic, where practical.**
`COMFY_DYNAMICCOMBO_V3` serialization is genuinely UI-owned logic. Rather than reverse-engineering and re-implementing every nuance of it, the most robust source of truth is the actual frontend running against the actual node. That's what `panel_*` live-canvas tools and `/comfy:convert` already give us — the plan leans on formalizing that path into the standard authoring flow instead of treating it as a separate manual workaround.

**5.5 — Golden-file regression tests per composite type.**
Once a DynamicCombo (or Autogrow, etc.) workflow is known to serialize and execute correctly, freeze its API-format JSON as a fixture. Any change to the compiler, to ComfyUI core, or to the frontend converter dependency gets tested against these fixtures in CI, so a regression is caught by a test failure, not a user's failed job weeks later.

**5.6 — Composite-input authoring should be structured, not stringly-typed.**
Long-term, an agent shouldn't need to know that `speech` is secretly a dict at all. The tool surface should expose composite inputs as their own structured parameter (mirroring `io.DynamicCombo.Option` shape: `{option: str, values: {...}}`), and the compiler resolves *that* into the dict-with-possible-links the executor expects. This removes the dotted-key/nested-dict guessing game entirely.

## 6. Phased Roadmap

### Phase 0 — Immediate mitigation (unblock now, ~1 day)
- Document the working manual path as an interim fix: build/verify the DynamicCombo wiring via the live canvas or `panel_*` tools, export, run `/comfy:convert`, feed the resulting JSON straight to `enqueue_workflow`.
- Add this as a documented troubleshooting entry (and to the `/comfy:debug` skill's suggestions) so agents surface it automatically instead of retrying dotted keys.
- **Exit criteria:** the HeyGen workflow runs end-to-end at least once via this path.

### Phase 1 — Composite type registry (diagnostics foundation)
- Build a scanner over live `/object_info` that flags every input across every installed node as `primitive` or `composite`, tagging composite inputs with their sub-schema (from the `DynamicCombo`/`Autogrow` definition).
- Re-run this scan on custom-node install/update (hook into existing `install_custom_node` / node-pack lifecycle).
- Expose it as a read tool (`list_composite_inputs` or similar) so agents/humans can query "does this node have anything weird before I try to wire it."
- **Exit criteria:** registry correctly flags `HeyGenTalkingPhotoNode.speech` as composite, and correctly flags a sample of 20+ known-plain nodes as primitive (no false positives).

### Phase 2 — Canonical compiler / serialization layer
- Implement the single "logical input → API JSON" function described in 5.1.
- For composite inputs, route through **one** of: (a) a maintained fork/vendoring of `SethRobinson/comfyui-workflow-to-api-converter-endpoint`'s conversion logic, or (b) driving the live frontend via `panel_*` and reading back the result, or (c) both, with (b) as the source of truth used to validate (a) in tests.
- Refactor `create_workflow` and `modify_workflow` to call this compiler for any input the Phase 1 registry marks composite, instead of writing raw dict/dotted keys.
- **Exit criteria:** `create_workflow`/`modify_workflow` can wire the HeyGen `speech` input correctly without touching the canvas.

### Phase 3 — Semantic pre-flight validation
- Extend `validate_workflow` to perform the `inspect.signature`-vs-JSON diff described in 5.3, for both primitive and composite inputs.
- Error messages name node, input, expected shape, and (for composite) which sub-key is missing/malformed.
- **Exit criteria:** the original dotted-key and bare-dict attempts from the bug report both get rejected by `validate_workflow` *before* enqueue, with an actionable message — not a queue-time `TypeError`.

### Phase 4 — Structured composite-input authoring API
- Add a first-class parameter shape for composite inputs in `create_workflow`/`modify_workflow` (e.g. `set_input(node_id, input_id, {"option": "...", "values": {...}}`), matching the `io.DynamicCombo.Option` structure instead of asking the caller to know the flattening rules.
- Update the "node authoring" skill and any agent-facing docs to describe this shape so agents stop guessing dotted-key vs nested-dict.
- **Exit criteria:** an agent given only the node's `/object_info` schema can wire a DynamicCombo correctly on the first attempt, without prior knowledge of this bug.

### Phase 5 — Regression harness + generalize beyond DynamicCombo
- Add golden-file tests (5.5) for at least: DynamicCombo, Autogrow, one nested-DynamicCombo-inside-DynamicCombo case.
- Extend the Phase 1 registry classification and Phase 2 compiler to cover `Autogrow` and `MatchType` groups using the same mechanism (they share the "frontend-computed, wire-format-incompatible" shape).
- Add these fixtures to CI so any dependency bump (ComfyUI core, frontend, converter library) that breaks composite serialization fails a test, not a user's job.
- **Exit criteria:** adding a new composite pattern is a matter of adding a classifier rule + a golden fixture, not new bespoke plumbing.

### Phase 6 — Upstream / community contribution
- File the gap as an issue against `artokun/comfyui-mcp`, referencing this PRD and the `dreamrec/ComfyPilot` precedent (§6.1) so reviewers have a concrete comparison point.
- Where the compiler in Phase 2 depends on a third-party converter (`SethRobinson`'s), consider contributing improvements upstream rather than forking silently, to keep it maintained as ComfyUI's V3 schema evolves.
- Consider whether the composite-type registry (Phase 1) is generally useful enough to propose as a `/object_info` extension upstream in `Comfy-Org/ComfyUI`, so every MCP/automation tool in the ecosystem benefits, not just this one.

## 6.1 Alternative / Complementary Tooling — Future Considerations

Don't build this in a vacuum. At least three other projects touch the same problem surface:

| Project | Relevance | Consideration |
|---|---|---|
| **`dreamrec/ComfyPilot`** | Already ships "strict live-catalog validation for V3 widgets, DynamicCombo/Autogrow inputs" (v1.9.0). This is effectively Phase 3 of this PRD, already built by someone else. | Before building Phase 3 from scratch, evaluate whether ComfyPilot's validator can be studied, vendored, or wrapped instead of reimplemented. At minimum, benchmark our error messages and coverage against theirs once built. If it's meaningfully ahead, it may be worth recommending as the near-term tool for anyone hitting this class of bug today, while `comfyui-mcp` catches up. |
| **`SethRobinson/comfyui-workflow-to-api-converter-endpoint`** | A custom node exposing a `/workflow/convert` endpoint that reimplements the frontend's "Save (API)" JS logic in Python, explicitly handling `COMFY_DYNAMICCOMBO_V3`, reroutes, and bypassed nodes. | Candidate dependency for Phase 2's compiler rather than reimplementing frontend serialization logic from scratch. Needs a maintenance-risk check (bus factor, update cadence vs. ComfyUI core/frontend release pace) before depending on it in production. |
| **`lalanikarim/comfyui-mcp`, `IO-AtelierTech/comfyui-mcp`** | Other MCP servers reported to have explicit UI-format/API-format detection and `convert_workflow_to_ui`/format-aware `save_workflow` tools. | Worth a short spike to see whether their detection/conversion approach differs meaningfully from `/comfy:convert`'s, and whether either has already solved Phase 2/3 in a way that's portable. |

**Recommendation:** treat Phase 2 and Phase 3 as "build vs. adopt vs. wrap" decisions, not default-build. A half-day spike against ComfyPilot's validator and the SethRobinson converter before writing new code could cut significant scope from this PRD.

## 7. Success Metrics

- Zero DynamicCombo/Autogrow related `TypeError`s reaching ComfyUI's execution queue from `comfyui-mcp`-authored workflows (caught earlier by Phase 3 validation instead).
- Time-to-wire a new composite-input custom node drops from "requires canvas + manual convert + trial and error" to a single structured tool call (Phase 4).
- At least one golden-file regression test exists per composite input pattern supported (Phase 5), and CI catches at least one real upstream breakage during the rollout window (proves the harness works).

## 8. Risks & Open Questions

- **Upstream churn:** V3 schema is explicitly marked as still evolving (`comfy_api.latest` vs versioned `v0_0_2`). The compiler and registry need a compatibility strategy (pin to a stable versioned API, re-test on `latest` in CI) rather than chasing `latest` reactively.
- **Third-party dependency risk:** if Phase 2 leans on `SethRobinson`'s converter, that's an external maintenance dependency for a core code path. Mitigate with the golden-file tests (Phase 5) so a break is visible immediately, and keep a documented fallback to the "drive the live frontend" path (5.4) if the converter falls behind.
- **Scope creep into Autogrow/MatchType:** Phase 5 assumes these generalize cleanly from the DynamicCombo work. That should be validated with a spike early (during Phase 1) rather than assumed.
- **Open question:** should the composite-type registry (Phase 1) be proposed upstream to `Comfy-Org/ComfyUI` as a first-class `/object_info` field, rather than something every downstream MCP re-derives independently? Worth raising in the Phase 6 upstream conversation.

## 9. References

- `docs.comfy.org/custom-nodes/v3_migration` — DynamicCombo schema definition and `execute()` argument shape
- `Comfy-Org/ComfyUI` issue #8580 — V3 Custom Node Schema introduction
- `Comfy-Org/ComfyUI` issue #11623 — DynamicCombo nested `display_name` bug (evidence of active, evolving schema)
- `Comfy-Org/comfy-cli` issue #446 — documents `COMFY_DYNAMICCOMBO_V3` marker and lists `SethRobinson/comfyui-workflow-to-api-converter-endpoint`, `lalanikarim/comfyui-mcp`, `IO-AtelierTech/comfyui-mcp`
- `dreamrec/ComfyPilot` release notes v1.9.0 — "strict live-catalog workflow validation for modern V3 widgets, DynamicCombo/Autogrow inputs"
- `artokun/comfyui-mcp` README/wiki — `create_workflow`, `enqueue_workflow`, `modify_workflow`, `validate_workflow`, `/comfy:convert`, `panel_*` live-canvas tools
