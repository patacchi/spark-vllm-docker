# Progress — sess-glm53-nvfp4-eval-20261121

NVFP4 (nvidia/GLM-5.3-Flash-NVFP4, PR #421 adaptation) evaluation on 2-node DGX Spark.
Ops recipe: recipes/modified/glm-5.3-flash-nvidia-nvfp4.yaml (branch ops-glm53-local).

## Consolidation note (2026-11-22)

ALL results, decisions and debug history consolidated into `mem:nvfp4-eval-results`
(final config, perf verdict, agent-workload synthesis, co-residency findings, KV
density two-component model, launch-OOM history, artifact paths). The chronological
intermediate states that used to live here (Phase 1 failure analysis, root-cause
hunt, per-launch verification logs) are DISCARDED: the durable record lives in the
results memory, and the commit history (b79b840, 73bd9ca, f6c7648, a73f463) covers
the rest.

## Current state

- Server: NVFP4 UP on both nodes — KV 6.5G / util 0.83 / 800K ctx; pool 6.05 GiB
  → 2,059,770 tokens (2.57x @ 800K), 7 mods applied, MTP healthy, PING OK.
- Recipe committed through a73f463 (KV 6.5G + util 0.83 + allocator fraction
  aligned to 0.83).
- Benchmark: final no-game run shows the tuned profile is free vs KV 7G standalone
  (prefill 1522-1562, tg 25.7-34.4, 1-tok 23.6 ms). Verdict + artifact paths in
  `mem:nvfp4-eval-results`.
- NEXT: user decides Task 3 (issue draft). Standing rule: any GitHub post needs the
  full text approved by the user first.
- map_visible: no (memories only; recipe content changes were committed earlier).
