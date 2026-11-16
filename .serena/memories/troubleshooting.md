# Troubleshooting Notes (live debug)

This host (thinkstationpgx-1cec) serves: client → caddy :443 (`*.patamon.jp`, LAN DNS resolves to 192.168.0.151, no Cloudflare for LAN clients) → OpenWebUI :3000 (docker, open-webui-cuda) → vLLM :8000 (container `vllm_node`, host network).

## Debug surfaces

- vLLM engine logs: tmux window `MainWIndow-48:1` (`tmux capture-pane -p -t MainWIndow-48:1 -S -N`). docker logs empty (process exec'd in container).
- Metrics: `curl http://localhost:8000/metrics` (num_requests_running, generation_tokens_total, request_success_total by finished_reason).
- Dev endpoints (recipe env `VLLM_SERVER_DEV_MODE: "1"`): `/server_info`, `/tokenize`, `/detokenize`, `/reset_prefix_cache`, `/sleep`, `/wake_up`, `/is_sleeping`, `/weight_info`. Note `/reset_prefix_cache` does NOT clear SSM/MTP internal state.
- Packet capture: needs sudo password (agent shell has no passwordless sudo) — ask user to run `sudo timeout N tcpdump -i any -w /tmp/x.pcap 'host 172.23.0.2 and tcp port 8000'`. pcap linktype 276 (LINUX_SLL2): ethertype at bytes [0:2], IP header at offset 20. veth+bridge duplicate packets → dedup by TCP seq.
- OpenWebUI logs: `docker logs open-webui`. DB: `docker exec open-webui python3` + sqlite3 `/app/backend/data/webui.db` (model params, function/pipe code in `function.content`).

## Known issues (2026-11-14)

1. **GLM-5.3-Flash-FP4-Local empty-stop loop** (Windows vllm-copilot: "model returned no output ... finish_reason: stop"): model emits an instant tool call for a tool NOT in the request's tools list (earlier suspect: `mcp_serena_list_memories`) then EOS; glm47 tool-call parser discards it → content=None, tool_calls=None, finish=stop. RESOLVED 2026-11-15 by server-side replays (temp=0, deterministic): the hallucinated names in the 75-msg replay were actually `mcp_session-close_list_backup_dirs` + `mcp_session-close_list_backup_base_dirs`, synthesized from the plan-backup-sweep SKILL.md text in context (references session-close/list_backup_dirs etc.); EOS emission itself is NORMAL (stop_reason=154829 also appears in successful tool-call turns). Root cause = glm47 parser (`validate_tool_names=True`) silently dropping unknown-name tool calls, returning a fully empty message. PROOF: same replay + the two names added to `tools` → finish=tool_calls, both calls parsed perfectly. FIX APPLIED & VERIFIED 2026-11-15: mod `mods/fix-glm47-unknown-tool-name` (patch flips validate_tool_names=True→False in vllm/parser/glm47_moe.py), wired into recipes/modified/glm-5.3-flash.yaml. Live 2-node restart verified: same 65k replay now returns finish=tool_calls with both hallucinated calls intact (streaming emits name+args deltas; no-tools & known-name paths unchanged; unknown-name call now reaches the client). PRODUCTION CONFIRMED 2026-11-15: Windows empty-stop session recovered — client received the unknown-name tool call, reported tool-not-found, fell back to shell and continued (no more retry loop). NOTE: container recreation reverts manual in-container edits — always apply via recipe mods. UPSTREAM ISSUE FILED 2026-11-15: vllm-project/vllm#67684 (author patacchi, drafted with AI assistance + human review disclosure, CPU-only minimal repro verified against released vllm==0.33.0 aarch64 wheel; references #63581/#63582 as partial). minimax_m2.py shares the same flag; kimi_k2/inkling set False. Local draft: ~/.copilot/session-state/b783146f-56d0-459d-9453-9e5d0bf6015c/files/issue-draft-glm47-unknown-tool.md.
PR FILED 2026-11-15: eugr/spark-vllm-docker#462 "Add fix-glm47-unknown-tool-name mod" from fork patacchi/spark-vllm-docker (branch fix-glm47-unknown-tool-name; clone at ~/gitrepos/patacchi/spark-vllm-docker). Contents: mod dir (context-free hunk patch + idempotent run.sh with VLLM_PACKAGE_ROOT support + README), glm-5.3-flash.yaml standard recipe mods entry, README mod list entry, tests/test_fix_glm47_unknown_tool_name_mod.sh (68 recipe tests still pass). Note: local eugr checkout has local commit f77b9c5 not on origin; fork was based on origin/main 6430ce1. Also in that dir: replay-req.json, replay-resp-nostream.json, w0/good-token-ids.json, tok-stop-tests*.json, win-fail*.pcap.
2. **This chat itself** (same vLLM) — mechanism CONFIRMED 2026-11-15 via server-side tests (no chat involved). NOTE: this is NOT fixed by the glm47 mod (needs engine-level fallback for unclosed/unknown tool-call blocks); mitigations remain the same. Three separate causes: (a) the tool-call marker characters (U+25C1/25E5-style ASCII strings defined in vllm/parser/glm47_moe.py) written literally in assistant content are consumed as parser structure — a closed pair vanishes from content, an UNCLOSED opening marker swallows the entire rest of the message (unknown name → dropped); (b) emitting the end-of-turn special token (id 154829) mid-text ends generation (legit EOS semantics); (c) non-streaming responses return reasoning under the `reasoning` key, NOT `reasoning_content` (streaming uses `reasoning_content`/`reasoning` in deltas) — earlier "empty response" reads were partly this key mismatch. Rule: never emit raw marker character sequences or EOS-style tokens in replies on this host; write them escaped/described or via files built with placeholder substitution.
3. Earlier orphan generation loop (10 tok/s, MTP acceptance 0%, 40+ min): client disconnect left open-webui→vLLM generation running. Fixed by: OpenWebUI env `AIOHTTP_CLIENT_STREAM_IDLE_TIMEOUT=300` + `AIOHTTP_CLIENT_TIMEOUT=3600` (requires `docker compose up -d` recreation, NOT restart/stop+start). vLLM-side defense: `repetition_detection` param (see ~/Templates/vllm-repetition-detection.md).
4. Windows-side VS Code settings: `%APPDATA%\Code\User\settings.json` shared copy at `/home/punipower/ComfyUI/output/ShareDocument/` (logs + settings.json). Its defaultParams: temp=0, presence_penalty=0.3, clear_thinking=false vs Linux: temp=1, presence 0, clear_thinking=true + repetition_detection.

## Workflow preferences (user instruction, 2026-11-15)

- Before posting any issue/PR/comment to GitHub, ALWAYS show the user the final full text and wait for their explicit approval; never post immediately after drafting or editing. Applies to edits of already-posted content too.

## Ops hygiene

- Push is disabled on this repo (`origin` push URL = `no_push`).
- Copilot CLI hooks for Serena: `~/.copilot/hooks/serena-hooks.json` (version 1, PascalCase events — see ~/Templates/serena-hooks-copilot.md).
- OpenWebUI has a `thinking_preserve_thinking_switcher` pipe function in DB, but current GLM usage bypasses it (direct OpenAI-style requests from vllm-copilot).

## Follow-up status (2026-11-15)

- vllm#67684: contributor Schrodingerscat07 volunteered to fix it (2026-11-14, not a maintainer).
  We posted a comment (issuecomment-6883957338) with verified root cause / repro / fix facts and
  offered to test patches against the live setup. No upstream PR from our side.
- PR eugr/spark-vllm-docker#462: OPEN. Verification note (standard recipe + mod,
  `--chat-template-content-format` no effect, prompt_tokens 65057 identical) added to the body.
  Commit 20ad082 fixed the mod README Usage (`glm-5.3-flash` is cluster-only → plain
  `./run-recipe.sh glm-5.3-flash`, no `--solo`/`--apply-mod`).
- Next: respond to contributor/maintainer follow-ups via the user-approval flow; remove the mod
  once upstream ships the fix.

- Background issue eugr/spark-vllm-docker#463 created (2026-11-15, user-approved): documents
  symptom, root cause, why agent clients loop, repro, and the `--apply-mod` workaround for
  general users. Exact container vLLM version recorded there:
  `0.1.dev21830+g8244d1c44.d20261102`, commit 8244d1c44348ae8a5bbfc9f1b607021652283ed4
  (local-inference-lab/vllm branch dev/karmic-kraken, wheel built 2026-11-02, see container
  /workspace/build-metadata.yaml). PR #462 Overview links to #463.

## Related upstream issues: three glm47 failure modes (2026-11-15)

Three OPEN upstream issues are three distinct failure modes of the same in-band
marker protocol in glm47 (markers are non-special added tokens in OUR tokenizer
too: 154843-154846 special=False, verified via AutoTokenizer):

1. vllm#67684 (ours): unknown tool NAME -> silent drop -> empty/stop retry loop.
   Workaround: our mod (validate_tool_names=False). Fix path: name gate / fallback.
2. vllm#60703: stray TOOL_CALL_START in prose -> phantom TOOL_NAME, no
   (TOOL_NAME, "TOOL_START") transition -> tail of stream (and any later real
   call's opener) accumulates as phantom name, dropped at finish -> truncation.
   This matches our assistant-chat truncation symptom (a). Fix PRs #60842
   (nikoder, blocked/merge conflicts, stale since Oct 12) and #60813 (dup):
   add recovery transition. LIMITATION: recovery only helps when a real tool
   call follows the stray opener; prose tail with no subsequent call is still
   swallowed. Full fix would need emitting phantom text as fallback content.
3. vllm#67471: quoted/mentioned tag sequence in prose -> parser converts it
   into a REAL tool call with arguments="{}" (fabricated call, schema-invalid).
   User hit this several times. Fix PR #67480 (opt-in validate_required_args
   schema gate, OPEN). Workaround per issue: tool_choice="none" or omit tools.

## DECIDED: upstream/PR strategy for the 3 mods (user, 2026-11-15) — BINDING

- vLLM upstream: NO new issues/PRs from us (movement already exists on all three).
- Pattern C equivalent-code proposal to upstream: ON HOLD (user decision later).
- spark-vllm-docker: issue + PR for ALL THREE mods, as SEPARATE issues (mod removal timing
  differs per upstream trajectory — that separation rationale must be stated in each issue).
- All 3 issues/PRs: state the GLM-5.3-model defect origin explicitly and CROSS-LINK each other
  (each references the other two + the upstream issues).
- Pattern B (unknown-tool-name) + C (stray-tool-start): normal PRs (upstream direction known).
- Pattern A (phantom-fallback): EXPERIMENTAL-feature track — RESEARCH spark-vllm-docker's
  experimental convention first (mods/exp-* dirs exist: exp-b12x, exp-w4a16; also check
  issue/PR title/label conventions) before drafting; then decide its PR form.
- Order: (1) research experimental conventions ✓DONE (see session research memory: no
  experimental label/process exists; #459 DFlash2 is the precedent — experimental = title/body
  wording + not in default recipe; PR body pattern = our #462's Overview/Claims/Validation/
  disclosure) → (2) draft issues → (3) SHOW USER the full text of each post and get explicit
  approval BEFORE posting (standing rule) → (4) PRs after issues, linked.
- POST PLAN (user, 2026-11-15): B (unknown-tool-name) = NO new issue; instead UPDATE existing
  eugr/spark-vllm-docker#463 with (i) why the three mods got separate issues (different upstream
  removal timings) and (ii) cross-references to the two new issues + upstream. A (phantom-fallback,
  experimental) and C (stray-tool-start) = NEW issues. Draft texts → user approval → post.
- Task 3 docs scope: final recipe + plain startup command + this classification (A/B/C)
  in README mod list (+ docs/AGENT_RUNBOOK.md if user wants — pending user answer).
- Working-state facts: repo=fork patacchi/spark-vllm-docker branch ops-glm53-local
  (13+ commits ahead of origin, push disabled — PR branches must be pushed via gh from a
  branch; remember push is disabled on origin: push URL = no_push → gh pr create needs the
  fork's GitHub remote or a temporary push target decision by the user).

## MOD CLASSIFICATION (2026-11-15, for Task 3 docs) — 3 patterns

Pattern A (independent/own implementation, NOT upstream-pending):
- fix-glm47-phantom-fallback: phantom-prose -> content rescue. NO upstream counterpart
  (#60703 discussion mentions the need, but no PR exists; our design + heuristic is own work).
  Prototype → adopt decision pending final user confirmation.

Pattern B (pre-empts upstream, applies to current release as-is):
- fix-glm47-unknown-tool-name: flips validate_tool_names=True→False. Equivalent to the
  eventual upstream direction for vllm#67684 (kimi_k2/inkling already ship False; minimax_m2
  still True upstream) — trivially applicable to any release carrying glm47_moe.py
  (context-free 1-line hunk). PR eugr/spark-vllm-docker#462 carries it upstream-of-us.

Pattern C (pre-empts upstream, does NOT apply to the dev build as-is):
- fix-glm47-stray-tool-start: implements vllm#60842/#60813's (TOOL_NAME,"TOOL_START")
  recovery transition, but as a context-exact 4-line insertion for OUR build (upstream
  diff context drifted: our build splices arg_tag_transitions separately; upstream folded
  ARG_KEY_START into the main dict). Upstream PRs are stale/blocked since Oct 12.
  Applies ONLY to builds matching our layout (pinned via test fixture; fails closed on drift).

--apply-vllm-pr mechanics (launch-cluster.sh): fetches PR diff, validates
runtime-only (vllm/ paths only; tests/ ignored), bundles as generated mod, at
container start runs `git apply --binary --include='vllm/**'` against installed
package; --reverse --check first for idempotency; refuses on dirty-apply.
Dry-run result: NEITHER #60842 NOR #60813 applies cleanly to our Nov 2 build
(context drifted: upstream TOOL_NAME state has ARG_KEY_START transition inside
the main dict; ours has it in a separate arg_tag_transitions dict spliced at
the dict tail via **arg_tag_transitions). A context-exact 4-line insertion of
the same transition applies cleanly on a file copy and passes ast.parse.
Note: recipe mods run before CLI mods; our flag-flip patch and the PR hunk are
in different regions, so both can coexist.

## Current plan (2026-11-15, in progress) — TODOs in session DB

Task 1 [mod-recovery-transition]: create mods/fix-glm47-stray-tool-start — 4-line patch
  adding (ParserState.TOOL_NAME, "TOOL_START"): Transition(ParserState.TOOL_NAME,
  (EventType.TOOL_CALL_END, EventType.TOOL_CALL_START)) into glm47_moe_config transitions.
  Patch shape (context-exact, verified applicable on a container-file copy; container lines
  187-193: (CONTENT,"TOOL_START") transition block then (TOOL_NAME,"TOOL_END") block):
    --- a/vllm/parser/glm47_moe.py
    +++ b/vllm/parser/glm47_moe.py
    @@ -186,6 +186,10 @@
                 (ParserState.CONTENT, "TOOL_START"): Transition(
                     ParserState.TOOL_NAME,
                     (EventType.TOOL_CALL_START,),
                 ),
    +            (ParserState.TOOL_NAME, "TOOL_START"): Transition(
    +                ParserState.TOOL_NAME,
    +                (EventType.TOOL_CALL_END, EventType.TOOL_CALL_START),
    +            ),
                 (ParserState.TOOL_NAME, "TOOL_END"): Transition(
                     ParserState.CONTENT,
                     (EventType.TOOL_CALL_END,),
  (Note: our container build has TOOL_NAME->ARG_KEY_START in a separate
  arg_tag_transitions dict, so upstream PR #60842/#60813 diffs do NOT apply as-is.)
  Idempotent run.sh (already-applied check -> skip), README, wire into
  recipes/modified/glm-5.3-flash.yaml mods list, add tests/test_fix_glm47_stray_tool_start_mod.sh,
  live 2-node restart + regression (replay, known-name tools, no-tools).

Task 2 [mod-phantom-fallback-eval]: prototype phantom-name->content fallback as
  Glm47MoeParser overrides (NO parser_engine.py changes): streaming path override
  _handle_tool_end -> put rejected/unclosed phantom name(+args) into self._deferred_content
  (engine emits deferred as content at TOOL_CALL_END when no tool deltas, engine
  parser_engine.py L843-852 verified); non-streaming via _build_extracted_result override;
  reconstruct opener literal via config token_id_terminals. ~60-100 lines in glm47_moe.py
  patch. Decide adopt-vs-document after prototype validation.

Task 3 [ops-convenience] DECIDED: no consolidation; recipe lists multiple mods
  (order-independent, patches touch disjoint regions; launcher applies mods in list order,
  recipe mods before CLI mods). Document final recipe + plain startup command.

Working rules: user approves all GitHub posts/comments/PR edits before sending; markers
(154843 etc.) written raw in MY chat get swallowed — refer to them by token ID or escaped.

Pending user decision: which repo to use as the VS Code working repo (fork patacchi/ vs
origin eugr/) before adding submodules .github/agents (git@github.com:patacchi/commonAgent-vscode.git)
and .github/skills (git@github.com:patacchi/commonSkills-vscode.git) — must check for
conflicts with existing .github/ contents first.

## Repo/branch migration (2026-11-15) — DONE, working dir moves to fork

Decision: VS Code working repo = fork /home/punipower/gitrepos/patacchi/spark-vllm-docker,
branch ops-glm53-local (operational). Prepared and pushed:
- 4149872: submodules .github/agents + .github/skills (patacchi/commonAgent-vscode,
  patacchi/commonSkills-vscode; SSH auth OK, no conflicts — .github had only workflows/)
- c772c9f: recipes/modified/glm-5.3-flash.yaml (copied from eugr local, identical content)
- branch is ff of fix-glm47-unknown-tool-name (20ad082), so PR #462 commits are included
Plan memory lives in .serena/memories/troubleshooting.md of the EUGR repo — after switching
the VS Code workspace to the fork, that memory must be read from
/home/punipower/gitrepos/eugr/spark-vllm-docker/.serena/memories/troubleshooting.md (or copied).

## Workspace .mcp.json + Serena config committed to fork (2026-11-15)

- e188dd0 on ops-glm53-local: .mcp.json (5 servers, relative paths per submodule README
  conventions: git-ro, session-close, harness-info via uv+mcp<2 python; context-usage,
  context7 via node), .serena/project.yml + project.local.yml (copied from eugr), all
  migration memories. Smoke-tested: uv+mcp import OK, node OK.
- After switching the VS Code workspace to the fork: start each server once in the MCP
  view (first-run trust), then Reload Window. context7 needs CONTEXT7_API_KEY in the
  user environment (set on this host already? verify with check-api-key).

- b129074: serena-mcp registered in fork's workspace .mcp.json per serena-mcp-setup skill
  canonical form: key "serena-mcp", --context=ide, --enable-web-dashboard=false, timeout
  60000. NOTE the eugr checkout used .vscode/mcp.json with --context=vscode --project
  ${workspaceFolder} (non-canonical scope) — that old registration stays only in the eugr
  checkout, do not replicate. restart_language_server is already enabled globally in
  ~/.serena/serena_config.yml (included_optional_tools). After switching workspace: start
  serena-mcp once in MCP view + Reload Window; audit says exactly 1 serena instance
  (--context=ide) should run for this workspace. Currently 2 old instances run on the host
  (context=vscode, project=eugr) from the old setup — harmless for the new workspace but
  candidates for cleanup per the skill's reconciliation ladder if they misbehave.

## clear_thinking verified (2026-11-15; user tested false, then reverted to true)

- Template-verified: clear_thinking only affects PAST-turn reasoning blocks in the prompt history
  (true = strip from prompt; false = keep as think blocks, model can re-read its past thinking).
  CURRENT-turn reasoning/content separation is identical either way. Single-turn outputs differ
  only by generation variance.
- pcap-verified (user ran sudo tcpdump, /tmp/ct-check.pcap): the client request DID carry
  chat_template_kwargs {"clear_thinking": false} — but the request body (~2.2MB, 646 assistant
  messages) contains ZERO "reasoning_content" fields: the client (vllm-copilot) strips past
  reasoning from history when sending. So clear_thinking=false is a NO-OP for THIS client;
  restoring past thinking to the prompt would require the client to send reasoning_content in
  history. Request also carries repetition_detection (max_pattern_size 64, min_pattern_size 8,
  min_count 3) and reasoning_effort "max". User reverted to true.
- EDGE CASE OBSERVED (no-tools request): with NO tools passed and the stray-opener prompt, the
  model hallucinated tool calls and mod1 delivered them: streaming emitted EMPTY-name calls
  ({"name":"","args":""}) — the unknown-name call's name is empty (name deltas absent), finish=
  tool_calls. With clear_thinking=false the model emitted 4 empty-name calls + 1 "BANANA" call
  (prose rescued as a NAME because it matched the identifier regex!). "BANANA" is a valid
  identifier string → not a phantom → bogus call. This is the documented single-word blind spot
  (progress item 6) observed LIVE. v2 renders rescued args, but empty-name calls streaming as
  name-less deltas are a cosmetic client-side artifact (client should ignore empty names).
  No server errors; log clean.

## Task 2 status (2026-11-15): mods/fix-glm47-phantom-fallback prototype implemented, live validation pending

- Prototype COMPLETE + offline-verified 17/17 in-container (module loaded from patched copy; no
  server restart). Rescues prose swallowed after a stray unclosed opener: phantom name (regex
  ^[A-Za-z0-9_.-]{1,128}$ heuristic) moved into deferred-content channel → emitted as content.
  Covers streaming (closed via mod2 recovery + unclosed via finish_streaming sweep) AND parse()
  (single-pass sweep+flush — needed because parse() bypasses finish_streaming and ignores
  tool_call_info.content). Engine (parser_engine.py) unchanged; Glm47MoeParser overrides only.
- Explorer's original draft was revised: _build_extracted_result override CANNOT cover parse()
  (_single_pass_parse calls it with no deltas and returns delta.content). Also: pre-call prose in
  thinking mode arrives as REASONING_CHUNK (initial state REASONING) — test expectations must
  account for it.
- Marker lesson: test fixtures must build marker strings from module constants (mod.TOOL_CALL_START
  etc.) — visually-copied glyphs were NOT the real ASCII strings and silently mis-parsed.
- Mod order: 3 glm47 mods are order-independent (disjoint regions; mod1 context-free); documented
  in mod README. Future mods: disjoint regions or pin recipe order deliberately (user question).
- Files/commits: mod dir + test + recipe 3rd entry + root README entry, commit 505d45c; session
  docs 509ec38. PENDING: live 2-node restart + regression (user approval), then ADOPT-vs-DOCUMENT.

## Task 1 status (2026-11-15): mods/fix-glm47-stray-tool-start implemented, live restart pending

- Mod created in the fork (ops-glm53-local), mirroring fix-glm47-unknown-tool-name (strict
  run.sh pattern: VLLM_PACKAGE_ROOT ladder, grep-post-token idempotency, patch -p1 -d
  "$VLLM_PACKAGE_ROOT"). Patch header is a/parser/glm47_moe.py — the earlier memory draft
  (a/vllm/...) was wrong for the run.sh invocation shape; regenerated the patch from the
  live container file with diff -U4, so hunk @@ -187,8 +187,12 @@ and context indentation
  (12/16) are exact. Verified on a container-file copy: clean apply, 4-line insertion only,
  ast.parse OK.
- Wired into recipes/modified/glm-5.3-flash.yaml (2nd mods entry); root README mod list
  entry added; tests/test_fix_glm47_stray_tool_start_mod.sh (apply/ast/idempotency/negative
  drift fail-closed) OK; template mod test OK; test_recipes.sh 68/68.
- Recipe resolution fact: bare `./run-recipe.sh glm-5.3-flash` = STANDARD recipe (500k ctx,
  unknown-tool-name mod only); ops variant = explicit path
  `./run-recipe.sh recipes/modified/glm-5.3-flash.yaml` (800k ctx, DEV_MODE, both mods).
- Patch regeneration was needed because the previous session's memory transcription of the
  hunk context had inconsistent indentation/counts — trust the live file, not transcriptions.
- LIVE VERIFIED 2026-11-15 (user performed the 2-node restart manually: stop both
  vllm_node containers, then ./run-recipe.sh recipes/modified/glm-5.3-flash.yaml; launcher
  refuses to start while containers are RUNNING — stopped-but-existing containers are fine,
  it recreates them):
  * both patches confirmed in container (recovery transition grep=1, validate_tool_names=False=1);
    server healthy, 800k ctx.
  * (a) 65k replay + unknown names → finish=tool_calls, both hallucinated calls intact,
    prompt_tokens 65057 (identical to pre-verification).
  * (b) known-name tool → correct tool_calls; (c) no-tools → prose + reasoning, finish=stop.
  * (d) forced stray-opener probe (user msg + reply both carry the raw opener ASCII string,
    built via chr() so it never transits chat): model emitted opener after a prose note →
    content ends exactly at the emission point (opener consumed as parser structure), phantom
    name created, then the REAL call still parsed (finish=tool_calls, get_weather) — the fix
    path vs pre-fix truncation; no junk phantom call in output; streaming variant also OK.
  * engine log: no parser errors/warnings.
  * KNOWN LIMITATION (documented in mod README, upstream #60703): prose tail with no
    subsequent tool call is still swallowed (recovery only rescues a following real call).
- Serena `core` map retrofitted with Rule 3 headers (full-replacement after apply_core_delta
  refused header-only vs headerless map); full restructure (sections, recipes/backup
  staleness, .github/.mcp.json/.serena coverage) is Task 2's Explorer's job.

## Task 3 draft revisions (user feedback, 2026-11-15 22:14) — BINDING for all posts (incl. #463)

- Title style (user, final): MODEL NAME FIRST, then the purpose in extreme brevity, then
  "(glm47 parser; ...)". USER: titles MUST carry the model — which model it occurred on is
  a major triage branch (2026-11-15 22:14, two review rounds).
  - C: "nvidia/GLM-5.3-Flash-FP4-Local: stop a stray tool-call opener in prose from swallowing the rest of the message and the real tool call that follows (glm47 parser; tracked upstream: vllm-project/vllm#60703)"
  - A: "nvidia/GLM-5.3-Flash-FP4-Local: return prose swallowed by a stray tool-call opener to the user as message content (experimental glm47 parser mod; related: vllm-project/vllm#60703)"
  - B: RETITLE #463 to "nvidia/GLM-5.3-Flash-FP4-Local: deliver tool calls with unknown names instead of silently dropping them (glm47 parser; empty responses + agent-client retry loop; tracked upstream: vllm-project/vllm#67684)"
  (Same principle will apply to the later PR titles.)
- EVERY mod issue/PR (A, C, and the #463 update) carries a shared comparison table:
  columns Without-the-mod / With-the-mod / Mod / Issue / PR; rows B=#463/#462, C, A(experimental).
  Self-row marked "this issue"; in the C post A's row says "link to follow" and a follow-up
  C edit fills A's number after A is posted.
- Model names ALWAYS carry the org prefix: nvidia/GLM-5.3-Flash-FP4-Local (the repo has
  same-model-different-repo cases).
- C gets an explicit "what this mod does NOT fix" section pointing at A (C rescues only a
  following real call; the phantom text between openers is still dropped).
- A gets an explicit distinction section vs B/C (own design + heuristic, experimental, no
  upstream PR to wait for).
- B update = retitle + BODY edit (org-prefix fix in body + table with real numbers +
  separation rationale + cross-links; full final body shown before editing).
- Post order unchanged: C -> A (carries C's number) -> C small edit (fill A link) -> B update.

## POSTED (2026-11-15 22:47) — issues DONE, PRs NEXT
- C = eugr/spark-vllm-docker#464 (stray-tool-start; title/model-first, comparison table,
  "does not fix" section; marker literal in body built via ast from /tmp/task2-src source —
  never transited chat).
- A = eugr/spark-vllm-docker#465 (phantom-fallback experimental; table with real #464,
  differs-from section, experimental status).
- #464 edited: A row now links #465; "does not fix" section references #465.
- #463 edited: retitled (model-first) + body edit (nvidia/ prefix in Summary, comparison
  table with real numbers #464/#465, separation rationale, cross-links) — user approved.
- #465 edited twice (user review): (a) removed the LOCAL-ONLY recipe reference
  (recipes/modified/glm-5.3-flash.yaml) and "3-mod recipe" phrasing; (b) added a
  "## How to apply" section with the 3x --apply-mod command against the OFFICIAL
  glm-5.3-flash recipe. #464/#463 verified clean of local-recipe references.

## NEW STANDING RULES from issue review (user, 2026-11-15 22:47-23:00) — BINDING
- Reference ONLY officially git-tracked recipes (e.g. glm-5.3-flash) in issues/PRs/READMEs.
  NEVER mention recipes/modified/... (local-only ops variant) in any public post.
- Model names ALWAYS with org prefix (nvidia/GLM-5.3-Flash-FP4-Local) everywhere public.
- Titles (issues AND PRs): model name FIRST, then the purpose in extreme brevity, then
  "(glm47 parser; ...)".
- The companion-mod requirement (phantom-fallback requires the other two) must be stated in
  the mod READMEs (phantom README has it; stray-tool-start README must ADD a mention of the
  experimental companion) and in PR bodies.

## Task 3 doc-pass items (fold into the PRs — PR branches carry the public file versions)
- mods/fix-glm47-phantom-fallback/README.md: drop the local-recipe line ("The recipe lists
  them ..."), usage = --apply-mod against official glm-5.3-flash (as in #465); Status
  "Prototype for evaluation" -> adopted-experimental wording.
- mods/fix-glm47-stray-tool-start/README.md: add mention that fix-glm47-phantom-fallback
  (experimental, #465) addresses the prose-tail limitation.
- Root README entries: phantom-fallback "Prototype for evaluation" -> experimental wording;
  keep companion-requirement note.
- NEXT: PRs — B/C normal PRs (#462 body pattern: Overview/Claims/Validation/AI-disclosure),
  A experimental-framed PR. Push mechanics VERIFIED 2026-11-15 22:47: origin = patacchi fork
  (https) with a NORMAL push URL — the old "no_push" memory is STALE (it described the eugr
  checkout era). origin/ops-glm53-local is BEHIND local (~15 commits: mods, tests, recipe,
  README, .mcp.json, memories not yet on the fork). PR branches must be created FRESH from
  upstream/main + only the PR's files (ops branch carries private memory/mcp commits that
  must NOT enter PRs). PR titles: model name first, purpose, then (glm47 parser; ...).
- RECIPE POLICY DECIDED (user, 2026-11-15 23:20): option (a) — C's PR wires the OFFICIAL
  glm-5.3-flash recipe (same as #462); issues/READMEs document BOTH forms (--apply-mod
  primary + "wired into recipe, plain command applies it after merging"). A stays unwired
  (experimental). Upstream house style confirmed: many recipes pre-wire mods. #462 merge
  order: both PRs touch the recipe's mods: list -> C's PR body notes rebase-after-#462.
- APPLIED: #464 gained the recipe-wiring bullet; #465 gained "Alternatively, add the three
  mods to a recipe's mods: list"; C README Usage rewritten (--apply-mod + wiring note,
  Coexists line references #462/this PR); A README gained the recipe-addition line.
- PR BRANCHES READY (local, not pushed): fix-glm47-stray-tool-start @ /tmp/pr-c (6 files,
  220+, amended) and fix-glm47-phantom-fallback @ /tmp/pr-a (5 files, 321+, amended); tests
  pass in worktrees (mod tests OK, test_recipes.sh 68/68 both). Push to origin (https,
  normal push URL) then gh pr create --repo eugr/spark-vllm-docker --head patacchi:<branch>
  --base main — AFTER user approves title+body.

## ALL POSTED (2026-11-15 23:59) — issues + PRs COMPLETE
- PRs: #466 = stray-tool-start (6 files, 222+; recipe wiring), #467 = phantom-fallback
  experimental (6 files, 333+; recipe wiring added per final user decision — A is wired
  like B/C, opt-out = remove from recipe mods:[]). Both branches pushed to origin.
- UNIFIED apply wording EVERYWHERE (issues #463/#464/#465 + all three mod READMEs):
  "recipe applies automatically after merge; opt out = remove line from recipe's mods:
  list; on-demand = --apply-mod CLI". #462's branch also got the unified README Usage
  (commit cd42ac2, pushed — PR #462 auto-updated).
- Issue tables now carry real PR numbers (#466/#467); no "PR to follow"/"link to follow"/
  "Not wired"/local-recipe references remain (verified by grep).
- #465 EDIT (user, 2026-11-16 00:37): removed the #459 precedent reference (DFlash2 is
  technically unrelated to the parser-defect class; cross-link web = the 3 mod issues +
  upstream ONLY). The "experimental reference implementation, not a fix proposal" framing
  itself stays.
- UPSTREAM #67684 MOVEMENT (2026-11-14/15, seen 00:50 11-16): contributor Schrodingerscat07
  volunteered (Nov 14), user (patacchi) replied Nov 14 with the verified root-cause/repro/fix
  summary (mentions #462 + our live 2-node deployment, offers to test patches), contributor
  reproduced on main Nov 15 and is checking interaction with open parser-engine PRs
  vllm-project/vllm#63582 (contiguous indices on rejected calls — relevant to our phantom
  sparse-index note) and #67480 (reject calls omitting schema-required args) before a PR.
  IMPLICATION: Pattern B's mod (#462) removal timing is approaching (upstream PR may land);
  keep mod-removal-timing wording as-is (already conditional). If their PR changes
  validate_tool_names handling, C/A context-exact hunks may need re-checks.
- UPSTREAM PR LANDED (2026-11-16 20:33 observed): vllm-project/vllm#67797 (OPEN, by
  Schrodingerscat07, "Stop dropping unknown tool names in GLM-4.7 parser") = the #67684
  fix. Same 1-line change as our #462 (validate_tool_names=False) PLUS ling3 flipped too.
  4414 tests pass; explicitly compatible with #63582/#67480; credits user's live
  verification + eugr/spark-vllm-docker#462; AI-assisted disclosure included.
  IMPLICATIONS: (1) Pattern B (#462) removal timing nears once #67797 merges — #463's
  conditional wording already correct, no edit needed. (2) His diff touches only
  glm47_moe_config() line 264-ish — C's transition-table hunk and A's overrides unaffected
  in CONTENT; patch line numbers will shift (context-based apply, fine; test fixtures are
  context-pinned and remain valid). (3) Re-check our 3 mods' context hunks when we ever
  rebuild the container on a post-#67797 vLLM.
- UPSTREAM #67480 MENTIONS #67797 (2026-11-16 06:00 comment, observed 20:46): PR adds
  validate_required_args (reject calls missing schema-required args). Author flagged the
  "described as opt-in but glm47 config sets True" tension, citing #67797 as the same
  tension for validate_tool_names — asking maintainers whether to default ON for GLM or
  off + per-request. NOTE: #67480 re-surfaces a rejected call as CONTENT in its original
  position — same philosophy as our Pattern A (phantom prose -> content), different scope
  (missing required args vs prose-like name). If #67480 lands, A gains an upstream
  partial-counterpart; consider a cross-link note on #465/#467 then. Watch both PRs.
- EMAIL PRIVACY FIX (2026-11-16 01:25): Samuel Hartmann / GPUwerk (DGX Spark rental
  vendor, legitimate DKIM/SPF, sent via mailforbots.com outreach service) cold-emailed
  patacchi@patamon.jp — address harvested from COMMIT author metadata (public on GitHub
  API despite the profile email being private; PR #462 + all our commits carry it).
  Fix applied: git global user.email -> 78113802+patacchi@users.noreply.github.com
  (verified on a fresh commit). Old commits keep the real address (rewrite declined /
  not needed). Vendor reply: user's call — no draft requested yet.
- TASK 3 CLOSED (user, 2026-11-16 00:45): issues + PRs = the delivered scope; RUNBOOK
  mod-status section = NO (user decision); ops-side "final recipe + plain startup command"
  docs = DROPPED from this session (user chose complete_session). Ops branch untouched by
  the PR work (PR branches are fresh from upstream/main). Recipe/ops facts remain recorded
  in this memory (Recipe resolution fact, launcher behavior, restart procedure).
