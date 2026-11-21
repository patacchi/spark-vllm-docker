# Session anchor (Agent mode)

- session_key: sess-glm53-nvfp4-eval-20261121 (NVFP4 evaluation, PR #421 adaptation;
  entry CLOSED + removed 2026-11-22 — results live in `mem:nvfp4-eval-results`).
  Previous sessions CLOSED: sess-glm47-stray-tool-start-20261115 (Task 1),
  sess-glm47-phantom-fallback-20261115 (Task 2 + Task 3: issues #463/#464/#465,
  PRs #466/#467).
- Workspace: fork /home/punipower/gitrepos/patacchi/spark-vllm-docker, branch ops-glm53-local.
- Plan + full debug history + standing rules (official recipes only in public posts,
  nvidia/ model prefix, model-first titles, marker-token chat rule): read `mem:troubleshooting`.
- Upstream watch (as of 2026-11-16): vllm-project/vllm#67684 — contributor
  Schrodingerscat07 reproduced on main, PR forthcoming; interacts with vllm#63582/#67480.

## CURRENT POSITION (2026-11-22 01:4x)

EVALUATION COMPLETE — all results consolidated in `mem:nvfp4-eval-results` (final
config a73f463: KV 6.5G / util 0.83 / 800K; perf verdict; agent synthesis;
co-residency; KV density model; OOM history; artifacts). Session entry
sess-glm53-nvfp4-eval-20261121 CLOSED + removed (commits 69b98f2 consolidation,
7c97a3f entry removal). Server live: NVFP4 KV 6.5G/0.83, pool 2,059,770 tok @
800K (2.57x), 7 mods OK, MTP healthy.
NEXT: user decides Task 3 (issue draft; full text pre-approval required).