# Session anchor (Agent mode)

- session_key: none active. Closed sessions: sess-glm47-stray-tool-start-20261115 (Task 1),
  sess-glm47-phantom-fallback-20261115 (Task 2 + Task 3), sess-glm53-nvfp4-eval-20261121
  (NVFP4 eval). Full history + standing rules: `mem:troubleshooting`.
- Workspace: fork /home/punipower/gitrepos/patacchi/spark-vllm-docker, branch ops-glm53-local.
- Two live recipes (both verified 2026-11-22): FP4-Local (3 glm47 mods) and NVFP4
  (recipes/modified/glm-5.3-flash-nvidia-nvfp4.yaml, 7 mods incl. PR#421's 5; KV 6.5G/0.83,
  800K, MAX_JOBS=2). Currently serving NVFP4. NVFP4 eval numbers: `mem:nvfp4-eval-results`.

## CONSOLIDATED STATE (compact-safe, 2026-11-22 02:36)

### Posted (all user-approved full text)
- Issues: #463 (B, retitled + comparison table), #464 (C stray-tool-start), #465
  (A phantom-fallback, experimental; #459 reference removed later as unrelated)
- PRs: #466 (C mod + official-recipe wiring), #467 (A mod experimental + wiring),
  #462 (B, README Usage unified; branch cd42ac2). All OPEN, no maintainer reaction yet.
- Unified apply wording everywhere: recipe auto-applies after merge / opt-out = remove
  from recipe mods:[] / on-demand = --apply-mod CLI.
- PR branch mechanics: one fresh branch per PR from upstream/main; origin = patacchi fork,
  normal push URL (no_push memory was stale).

### Standing rules (BINDING, user-set)
- Public posts reference ONLY official git-tracked recipes (never recipes/modified/).
- Model names with org prefix (nvidia/GLM-5.3-Flash-FP4-Local); titles: MODEL FIRST, then
  purpose, then (glm47 parser; ...).
- Full text pre-approval before any GitHub post/edit; live restarts need prior approval.
- Raw markers (token ids 154843-154846) never in chat — ids or chr() only.
- git user.email now = 78113802+patacchi@users.noreply.github.com (commit-metadata harvest
  contact from GPUwerk; profile email already private; SSH key comment clean).

### Upstream watch (as of 2026-11-22) — stable, waiting on reviews/merges
- vllm#67797 (B's fix, 1-line + ling3, same as our #462): OPEN, labels bug/tool-calling/glm
  (triage passed), credits user's live verification; awaiting review/merge. When merged →
  B mod removable; #463 wording already conditional.
- vllm#67480 (validate_required_args): OPEN, active review (smiyamoto3), author rebasing
  after Nov 18 conflict notice. Implements rejected-call→content re-surface (args-only
  scope; does NOT cover name rejection). If merged: partial counterpart for mod3's
  philosophy; consider cross-link note on #465/#467 then.
- vllm#60703 trio (C's upstream): #60842 rebased Oct 20 (applies to current main,
  dry-run verified), #60813 stale but also applies; both stalled without review. Our
  mod2 stays REQUIRED (our build has extra states TOOL_BETWEEN/TOOL_ARG_END_PENDING;
  upstream PRs don't apply to our build — verified).
- mod3 (name-phantom→content): still OURS ONLY upstream. Philosophy validated by #67480's
  design; generalization to name-rejection not yet proposed by anyone.
- glm47 files on upstream main: last touched Oct 29/Nov 8 (docstring hook). No structural
  change since our build was cut.

### NEXT (user decides)
- Wait for upstream merges (watch #67797, #67480, #60842); on merge, update issue wording
  and consider mod removal timing.
- eugr PR reviews: none yet — follow up if silent for a long period.
- Bench artifacts: ~/bench-results/ (NVFP4 phase 2 + FP4-Local baseline, llama-benchy).