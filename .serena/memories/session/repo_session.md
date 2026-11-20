# Session anchor (Agent mode)

- session_key: none — previous sessions CLOSED: sess-glm47-stray-tool-start-20261115
  (Task 1), sess-glm47-phantom-fallback-20261115 (Task 2 + Task 3: issues #463/#464/#465,
  PRs #466/#467 — all posted 2026-11-15/16, unified apply wording).
- Workspace: fork /home/punipower/gitrepos/patacchi/spark-vllm-docker, branch ops-glm53-local.
- Plan + full debug history + standing rules (official recipes only in public posts,
  nvidia/ model prefix, model-first titles, marker-token chat rule): read `mem:troubleshooting`.
- Upstream watch (as of 2026-11-16): vllm-project/vllm#67684 — contributor
  Schrodingerscat07 reproduced on main, PR forthcoming; interacts with vllm#63582/#67480.

## CURRENT POSITION (compact-safe checkpoint, 2026-11-21 01:25)

NVFP4 EVALUATION (PR #421 adaptation) — PHASE 1 READY, waiting for user launch:
- recipes/modified/glm-5.3-flash-nvidia-nvfp4.yaml @ b9b1f35: KV 5G / ctx 300K /
  util 0.86 / PYTORCH_ALLOC_CONF=per_process_memory_fraction:0.86 / MTP moe_backend=humming
  / load-format b12x / 7 mods. Goal: ONE run where the flashinfer fused_moe JIT completes.
- Why: NVFP4 loads 92.22 GiB vs FP4-Local 83.62 (+8.6 online-quant transients); at KV 8G
  + 0.83 headroom was 0.7 GiB → JIT spike killed the worker silently; node2 swap nearly
  exhausted (now 40GB). JIT cache cleaned (half-built fused_moe_120 removed via busybox).
- Baseline saved: ~/bench-results/baseline_fp4local_1120-2020.json (pp 1.8-1.9k t/s,
  tg 28.6→20.5 by depth). llama-benchy installed; corpus cache seeded (Gutenberg 503).
- Launch = USER action: stop both vllm_node containers → run-recipe.sh
  recipes/modified/glm-5.3-flash-nvidia-nvfp4.yaml. Then agent verifies (JIT done, cache
  warm, 7 mods, MTP acceptance) → NVFP4 bench → comparison table → PHASE 2 (ctx 800K,
  KV 6-8G, single edit).
- Full detail: mem:troubleshooting "NVFP4 evaluation" section.