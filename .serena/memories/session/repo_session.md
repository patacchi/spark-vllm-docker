# Session anchor (Agent mode)

- session_key: sess-glm47-stray-tool-start-20261115 (Task 1 COMPLETE incl. live verification)
- map_visible: YES (Task 1 added mods/fix-glm47-stray-tool-start, tests/test_fix_glm47_stray_tool_start_mod.sh, recipe/README changes — body committed; Task 2's Explorer authors the core delta)
- workspace: fork /home/punipower/gitrepos/patacchi/spark-vllm-docker, branch ops-glm53-local
  (ff of fix-glm47-unknown-tool-name; includes PR eugr/spark-vllm-docker#462 commits)

## Per-task session entries (user instruction 2026-11-15)

ONE session entry PER TASK in this continuation. Create a fresh entry at each task start,
close (delete) it when that task completes. Keys:

1. `sess-glm47-stray-tool-start-20261115` — mods/fix-glm47-stray-tool-start (4-line
   (TOOL_NAME, "TOOL_START") recovery transition), recipe wiring, test. ACTIVE — entry files
   sessions/<key>/{research,progress}.md hold the merged delegation findings + status.
2. `sess-glm47-phantom-fallback-20261115` — phantom-name → content fallback prototype eval.
3. `sess-glm53-recipe-docs-20261115` — final recipe + startup command documentation.

Work plan / patch details / decisions: read `mem:troubleshooting` (sections "Current plan",
"Related upstream issues", "Task 1 status"). Delegation findings for Task 1 are merged in the
ACTIVE entry's research.md (template mod mechanics, container file facts, recipe resolution
fact, conventions).

## Ops rules (standing)

- GitHub posts/issues/PRs/comments/edits: show user the final FULL text and get explicit
  approval BEFORE sending. Applies to edits of already-posted content too.
- Live server restarts (2-node regression) need prior user confirmation.
- Raw marker token sequences (ids 154843-154846) must NEVER appear in assistant chat —
  refer by token id or escaped form.
- Push is disabled on this repo (origin push URL = no_push).