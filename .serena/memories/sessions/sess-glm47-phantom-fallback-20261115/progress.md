# Progress — sess-glm47-phantom-fallback-20261115

## Task 2 [mod-phantom-fallback-eval] status

- Delegation: Explorer e744a510 (parser fallback design research) — gates OK; study of
  /tmp/task2-src/{glm47_moe.py, parser_engine.py} (container-extracted, 1-based line refs);
  NO core map mismatch (map fresh, source commit 5f8aa0f). Scratch merged into entry
  research.md (design + corrections + pitfalls) and deleted.
- Design confirmed + 3 plan corrections (unclosed phantom needs finish_streaming sweep;
  deferred gate per-delta; _build_extracted_result override required because parse()
  bypasses finish_streaming). See entry research.md.

## Open questions RESOLVED (2026-11-15 live + offline validation)

1. StreamingParserEngine.finish() for open TOOL_NAME slots — RESOLVED empirically: live Probe A
   (unclosed opener + prose tail, tools present) → prose returned as content, no calls; finish()
   does NOT synthesize a TOOL_CALL_END for the open slot (otherwise a bogus prose-name call would
   have appeared even under mod1+mod2 without the rescue). Design robust either way.
2. tool_index sequencing — no anomalies in live probes; sparse index after phantom accepted
   (documented). Not a blocker.
3. TOOL_NAME event.value fidelity — verbatim confirmed (rescued content matches the model's text
   exactly in all probes).
4. Serving-layer entry — both covered: streaming (Probes A/B) AND non-streaming (Probe D) verified
   live; offline tests additionally covered parse() and extract_tool_calls() paths.
5. Phantom-with-args rendering — RESOLVED by v2: strip ARG_KEY/VALUE structural markers, join as
   'key value' text (raw markers leaked into content in v1; found in boundary tests).
6. Heuristic boundary — RESOLVED via offline matrix: CJK/mixed-unicode/emoji/spaced/129-char names
   → phantom → rescued; 128-char + dotted/dashed → valid → call; single-word prose name still a
   call (documented blind spot, acceptable per OpenAI name spec). Model self-correction observed
   in live probes E/F (could not force a raw hallucinated call via prompts — offline covers it).

## Verification log (2026-11-15)

- LIVE (user-restarted 2-node, 3-mod recipe): 3 patches in container; (a) 65k replay unknown names
  → tool_calls both intact, prompt 65057; (b) known-name streaming OK; (c) no-tools prose OK;
  Probe A unclosed+tail → content restored (previously swallowed); Probe B phantom+real call in one
  message → both; Probe D non-streaming unclosed → content restored; engine log clean.
- MOD ORDER/COMBINATION MATRIX (user-requested): 15/15 apply+ast OK (6 permutations + subsets);
  behavioral: all m3-containing combos OK in ANY order; m3-less combos lack the feature (expected).
  Conclusion: 3 mods fully order-independent. v2 patch (marker-stripping rescue) deployed to repo
  (live container still runs v1 — cosmetic-only difference, restart optional).
- Test-harness lessons recorded: (i) markers must be built from module constants (visual glyph copy
  was wrong strings); (ii) find_tool_name needs Pydantic tool objects (plain dicts fail validation);
  (iii) after in-place edits, verify the edit token exists in EVERY copy used downstream (v2 edit
  lost twice through interrupted applies / stale source for patch regeneration).

## V2 LIVE VERIFICATION (2026-11-15, second user restart) — ALL PASS

- Container confirms v2: rescue body strips ARG_KEY_START/END + ARG_VALUE_START/END (grep 6 refs);
  3 patches present; health 200; 800k ctx.
- Task 1 regressions: (a) 65k replay unknown names → finish=tool_calls both intact, prompt 65057;
  (b) known-name streaming → correct call; (c) no-tools prose → finish=stop.
- mod3 core (v2 live): Probe A unclosed opener + prose tail (streaming) → content restored, no
  calls; Probe B echo opener + real call → content restored + get_weather parsed; Probe D
  non-streaming unclosed → content restored. Engine log clean.
- Probe G2 (forcing the model to echo a CJK hallucinated call with real arg markers) — INCONCLUSIVE
  for v2-vs-v1 discrimination: the model did not reproduce the exact marker structure reliably
  (re-quoted a simplified form; second attempts partially consumed as structure and rescued).
  Deterministic v2 verification remains the OFFLINE boundary tests (17/17 + B1-B8 matrix);
  live v2-vs-v1 difference is cosmetic-only (marker leakage in rescued args). Lesson: do not rely
  on model-echo prompts for exact marker-structure paths — use offline module tests.

## REMAINING

- ADOPT decision: recommend ADOPT (user said "タスク 3 は何をするタスクだったか" — decision pending
  their confirmation; live verification passed).
- Task 3 [ops-convenience]: document final recipe + plain startup command.

## Implementation status (2026-11-15) — prototype COMPLETE, locally verified

- Explorer's draft revised mid-implementation (see research.md "REVISED design"): the
  _build_extracted_result override could not cover parse() (single-pass returns delta.content,
  ignores tool_call_info.content) → replaced with a _single_pass_parse override; import dropped;
  final patch = 2 hunks (regex block + class body), 62 added lines, header a/parser/glm47_moe.py.
- Offline behavioral verification IN THE CONTAINER (module loaded from patched file copy, no
  server involvement): 17/17 PASS across streaming/parse/extract paths + regressions (see
  research.md for scenario list). First fixture attempt failed because visually-copied marker
  glyphs were not the real ASCII marker strings — tests must build markers from module constants.
- Files: mods/fix-glm47-phantom-fallback/{run.sh (grep _rescue_phantom_slot idempotency),
  README.md (prototype status + limitations), glm47_phantom_fallback.patch};
  tests/test_fix_glm47_phantom_fallback_mod.sh (apply + ast + idempotent re-run (token count 4)
  + negative drift fail-closed).
- Wired: recipes/modified/glm-5.3-flash.yaml 3rd mods entry; root README mod list entry.
- Validation: 3 mod tests OK; test_recipes.sh 68/68; dry-run shows all 3 --apply-mod flags in
  order; full 3-mod sequence on pristine container file copy → ast OK (verified before the
  design revision; revised patch re-verified on container-state copy).
- Mod ORDER: the 3 mods are order-independent (disjoint regions; mod1 context-free) — user asked;
  documented in mod README. Rule for future mods: keep patches in disjoint regions or pin the
  recipe order deliberately.
- PENDING: live 2-node restart + regression (needs user approval). ADOPT-vs-DOCUMENT decision
  after live validation.
