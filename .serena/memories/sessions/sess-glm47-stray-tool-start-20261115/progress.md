# Progress — sess-glm47-stray-tool-start-20261115

## Task 1 [mod-recovery-transition] — COMPLETE (implemented, locally verified, LIVE VERIFIED)
- Delegation: Explorer c3aa2abc (map-fit + groundwork) — gates OK, no shell used, no genuine
  Serena failures. Scratch merged into entry research.md and deleted; its core-delta.md was
  superseded by the caller's full-replacement retrofit (apply_core_delta refused with
  "stale base — source commit: None" against the headerless map).
- Implementation (all verified):
  - mods/fix-glm47-stray-tool-start/{run.sh, README.md, glm47_stray_tool_start.patch}
  - tests/test_fix_glm47_stray_tool_start_mod.sh (apply + ast.parse + idempotent re-run +
    negative drift fail-closed) — OK
  - recipes/modified/glm-5.3-flash.yaml: second mods entry (list order preserved)
  - README.md root mod list entry (alphabetical position)
- Validation: new mod test OK; template mod test OK (no regression); bash -n OK; recipe
  dry-runs OK (standard: 1 mod / modified via explicit path: both mods in order, 800k ctx,
  DEV_MODE); tests/test_recipes.sh 68/68 PASS.
- Patch provenance: regenerated from the live container file (diff -U4), header
  a/parser/glm47_moe.py, hunk @@ -187,8 +187,12 @@; clean-apply verified with the exact
  run.sh invocation on a container-file copy; ast.parse OK.
- Body commit: "Add fix-glm47-stray-tool-start mod" (see git log for hash).
- map_visible: YES (new mod dir + test + recipe/README changes) — Task 2's Explorer authors
  the full-replacement core delta (restructure + these additions) after the body commit.
- LIVE VERIFIED 2026-11-15: user restarted the 2-node cluster manually (launcher skips launch
  while containers are RUNNING; stopped containers are recreated fine). Both patches confirmed
  in container (recovery-transition grep=1, validate_tool_names=False=1); server healthy, 800k ctx.
  Regression: (a) 65k replay + unknown names → finish=tool_calls, both hallucinated calls
  intact, prompt_tokens 65057; (b) known-name tool → correct tool_calls; (c) no-tools → prose
  + reasoning, finish=stop; (d) forced stray-opener probe (opener built via chr(), never raw
  in chat): model emitted opener after a prose note → content ends at the emission point,
  phantom name created, then the REAL call still parsed (finish=tool_calls, no junk phantom
  call in output) — the fix path vs pre-fix truncation; streaming variant OK; engine log clean.
  Documented limitation (in mod README, upstream #60703): prose tail with no subsequent tool
  call is still swallowed.

## Open items
- Task 2 [mod-phantom-fallback-eval]: prototype phantom-name → content fallback as
  Glm47MoeParser overrides (NO parser_engine.py changes); decide adopt-vs-document.
  Its Explorer first authors the core full-replacement delta.
- Task 3 [ops-convenience]: document final recipe + plain startup command — note the recipe
  resolution fact in research.md (bare name = standard 500k recipe; ops variant = explicit
  path). DECIDED: no mod consolidation.
- docs/AGENT_DEVELOPMENT.md references nonexistent recipes/backups/ (docs fix candidate).