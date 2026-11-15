# Session anchor (Agent mode)

- session_key: sess-glm47-phantom-fallback-20261115 (Task 2 ADOPTED; Task 3 issues POSTED (#463/#464/#465), PRs NEXT)
- map_visible: YES (mod + test + recipe + README changes, commits 505d45c/f7d284d)
- workspace: fork /home/punipower/gitrepos/patacchi/spark-vllm-docker, branch ops-glm53-local
  (16 commits ahead of origin, push disabled; latest a043368)
- Previous session: sess-glm47-stray-tool-start-20261115 — CLOSED (Task 1: mods/fix-glm47-stray-tool-start, live-verified, commits 5f8aa0f..27131c6). Plan + full debug history: read `mem:troubleshooting`.

## CURRENT POSITION (compact-safe checkpoint, 2026-11-15 23:00)

Issues POSTED: #464 (C stray-tool-start), #465 (A phantom-fallback experimental, incl.
"How to apply" section), #463 retitled + body edit (comparison table, rationale, nvidia/).
Standing rules added by user (BINDING): reference ONLY officially git-tracked recipes —
NEVER recipes/modified/... (local-only) in public posts; model names with org prefix
(nvidia/GLM-5.3-Flash-FP4-Local); titles = model first, purpose, (glm47 parser; ...);
companion-mod requirement stated in READMEs + PR bodies.
PRs NEXT: one dedicated branch per PR, fresh from upstream/main, PR files only (no
.serena/.mcp.json/recipes-modified); C PR wires the official recipes/glm-5.3-flash.yaml,
A PR has no recipe wiring (experimental). origin = patacchi fork with a NORMAL push URL
(old no_push memory is STALE). PR body pattern: #462 (Overview/Claims/Validation/disclosure).
Doc pass folds into the PRs (README fixes: local-recipe refs, Status wording, companion
mentions). Details: mem:troubleshooting "POSTED" + "NEW STANDING RULES" + "Task 3 doc-pass".

## Task 2 [mod-phantom-fallback-eval] — from mem:troubleshooting "Current plan"

Prototype phantom-name→content fallback as Glm47MoeParser overrides (NO parser_engine.py
changes): streaming path override `_handle_tool_end` → put rejected/unclosed phantom
name(+args) into `self._deferred_content` (engine emits deferred as content at TOOL_CALL_END
when no tool deltas; engine parser_engine.py L843-852 verified); non-streaming via
`_build_extracted_result` override; reconstruct opener literal via config
token_id_terminals. ~60-100 lines in glm47_moe.py patch. Decide adopt-vs-document after
prototype validation.

Key facts already verified (prior session + this session's container extraction):
- Engine file in container: vllm/parser/engine/parser_engine.py (1268 lines) — extracted to
  /tmp/task2-src/parser_engine.py for read-only study (same content as container build 8244d1c44).
- glm47_moe.py (280 lines) extracted to /tmp/task2-src/glm47_moe.py. Glm47MoeParser(ParserEngine)
  at L238 ALREADY overrides _emit_name_delta (L261) and _handle_tool_end (L266) — the override
  point exists in glm47_moe.py itself.
- parser_engine.py landmarks: _deferred_content init L122; finalize emit gate L207
  ("if events or self._deferred_content"); streaming deferred emit L843-852 (emitted as
  content when no tool_call_deltas); _handle_tool_end L953 (rejected name → slot dies
  silently); _build_extracted_result L1180.
- With mod1 (validate_tool_names=False) active, unknown names are ACCEPTED as calls; the
  fallback's remaining target is the UNCLOSED phantom (opener seen, never TOOL_END →
  stream ends in TOOL_NAME state) and any genuinely rejected names.

## Per-task session entries (user instruction 2026-11-15)

1. `sess-glm47-stray-tool-start-20261115` — CLOSED (Task 1).
2. `sess-glm47-phantom-fallback-20261115` — ACTIVE (Task 2, this entry).
3. `sess-glm53-recipe-docs-20261115` — Task 3 pending.

## Ops rules (standing)

- GitHub posts/issues/PRs/comments/edits: show user the final FULL text and get explicit
  approval BEFORE sending.
- Live server restarts need prior user confirmation.
- Raw marker sequences (token ids 154843-154846) NEVER in chat — token ids or chr() only.
- Push disabled on this repo (origin push = no_push).
- Serena `core` map: FRESH (restructured + validated this session, source commit 5f8aa0f,
  check_core_map healthy) — Task 2 needs no fit re-check unless work hits a mismatch.