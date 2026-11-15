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
