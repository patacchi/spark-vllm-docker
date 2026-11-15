# Research — sess-glm47-stray-tool-start-20261115

Explorer delegation c3aa2abc (map fit + Task 1 groundwork), merged by caller 2026-11-15.
Full verbatim quotes were in the scratch dir (deleted after merge); essentials preserved here.

## Map (core) verdict
- Was headerless (no Rule 3 headers); layout spot-checks all PASS except: map claims
  `recipes/backup/` which does not exist (docs/AGENT_DEVELOPMENT.md also references
  `recipes/backups/`), and map lacks coverage of .github/ submodules, .mcp.json, .serena config.
- Retrofit: full-replacement write_core_map applied by caller (apply_core_delta refused the
  header-only patch against the headerless map: "stale base — source commit: None"). Body kept
  byte-identical, four headers added (source commit 9c5b26e). check_core_map: healthy.
- REMAINING for next delegation: full-replacement restructure (## sections, staleness fixes,
  .github/.mcp.json/.serena coverage) + this session's mod/test/recipe additions. Author =
  next Explorer (Task 2 prep), caller applies.

## Template mod mechanics (mods/fix-glm47-unknown-tool-name)
- Files: run.sh, README.md, glm47_unknown_tool_name.patch.
- run.sh: set -euo pipefail; VLLM_PACKAGE_ROOT ladder (env → VLLM_SITE_PACKAGES/vllm →
  PYTHON_ROOT/vllm → python3 find_spec heredoc, no vLLM import); target
  $VLLM_PACKAGE_ROOT/parser/glm47_moe.py; idempotency = grep the POST-patch token; apply =
  `patch -p1 -d "$VLLM_PACKAGE_ROOT" < "$PATCH"`. No --include filters in any mod.
- Patch diff paths are relative to the package root: `a/parser/glm47_moe.py` (NOT a/vllm/...).
- Older mods hardcode /usr/local/lib/python3.12/dist-packages with `|| echo` fallbacks —
  the template is the modern strict pattern to mirror.

## Decisions (blocker resolutions)
1. Patch header → `a/parser/glm47_moe.py` (template-identical); hunk content unchanged from
   the previous session's container-copy verification.
2. Context hunk kept (NOT context-free): an insertion has no changed line of its own to anchor
   on; the surrounding transition entries are the anchor and the apply fails closed on drift.
   GNU patch default fuzz=2 noted in the test's negative case.
3. Patch REGENERATED from the live container file (diff -U4) instead of trusting the memory
   transcription (which had inconsistent context indentation/counts). Verified: clean apply
   with the exact run.sh invocation, only the 4-line insertion (190a191,194), ast.parse OK,
   idempotency token appears exactly once.

## Container facts (build 8244d1c44, /usr/local/lib/python3.12/dist-packages/vllm)
- vllm/parser/glm47_moe.py: 280 lines. Main transitions dict: `**reasoning_transitions` splice
  at L178, main entries L179-202, `**arg_tag_transitions` splice at L203; separate
  arg_tag_transitions dict at L91 — this is why upstream #60842/#60813 hunks do not apply.
- Insertion point: between the (CONTENT, "TOOL_START") block (L187-190) and the
  (TOOL_NAME, "TOOL_END") block (L191-194); dict entries indented 12/16 spaces.
- `(ParserState.TOOL_NAME, "TOOL_START")`: 0 occurrences pre-patch → idempotency grep token.

## Recipe resolution fact (learned during dry-run validation)
- Bare name `./run-recipe.sh glm-5.3-flash` resolves to the STANDARD recipes/glm-5.3-flash.yaml
  (500k ctx, 1 mod: fix-glm47-unknown-tool-name). The ops variant needs the explicit path:
  `./run-recipe.sh recipes/modified/glm-5.3-flash.yaml` (800k ctx, DEV_MODE, both mods).
  Recipe `mods:` entries expand to repeated `--apply-mod` flags in list order (run-recipe.py
  L1422 region); recipe mods run before CLI mods.

## Conventions (mem:development, mem:conventions, dev guide)
- Mods: runtime-only, fresh-container; no migration/legacy/reversal logic; repo-relative mod
  paths; preserve exec bits; tests use fixtures/mocks (never real deployments); test invocation
  = direct exec of tests/test_*_mod.sh; bash -n for edited scripts; commit style imperative,
  no prefix.