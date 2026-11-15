# Progress — sess-glm47-phantom-fallback-20261115

## Task 2 [mod-phantom-fallback-eval] status

- Delegation: Explorer e744a510 (parser fallback design research) — gates OK; study of
  /tmp/task2-src/{glm47_moe.py, parser_engine.py} (container-extracted, 1-based line refs);
  NO core map mismatch (map fresh, source commit 5f8aa0f). Scratch merged into entry
  research.md (design + corrections + pitfalls) and deleted.
- Design confirmed + 3 plan corrections (unclosed phantom needs finish_streaming sweep;
  deferred gate per-delta; _build_extracted_result override required because parse()
  bypasses finish_streaming). See entry research.md.

## Open questions (from Explorer, carried into implementation/validation)

1. StreamingParserEngine.finish() behavior for open TOOL_NAME slots — verify empirically
   (forced unclosed-opener probe on live; needs user approval for restart/probes).
2. tool_index sequencing — verify if probes show index anomalies (streaming_parser_engine.py
   was not in the study set).
3. TOOL_NAME event.value fidelity (verbatim chunks) — low risk; confirm via probe output.
4. Serving-layer non-streaming entry (parse() vs extract_tool_calls()) — both covered by design.
5. Phantom-with-args rendering — chose raw args with space join; revisit if probes look bad.
6. Heuristic tuning (identifier regex) — false positives possible (single-word prose names);
   the 65k replay corpus + forced probes are the test bed.

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
