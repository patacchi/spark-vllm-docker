# NVFP4 evaluation results (2026-11-21/22) — final, Phase 2

NVFP4 (nvidia/GLM-5.3-Flash-NVFP4) vs FP4-Local baseline on 2-node DGX Spark,
branch ops-glm53-local. All numbers: llama-benchy 0.2.2, same corpus/shapes/runs
(pp 2048/8192, tg 128, depth 0/16K/64K, 3 runs + 1 warmup, T=0, thinking off,
prefix caching disabled), 1 concurrent.

## Final config (ops, commit a73f463)

- recipes/modified/glm-5.3-flash-nvidia-nvfp4.yaml: max_model_len 800000,
  KV 6.5G, util 0.83, PYTORCH_ALLOC_CONF "per_process_memory_fraction:0.83,
  expandable_segments:True" (single var — torch 2.13 ignores old-name var when
  new-name set), MAX_JOBS=2 (+FLASHINFER_JIT_PREBUILD_MAX_JOBS/NINJAFLAGS).
- Live verified: pool 6.05 GiB → 2,059,770 tokens (2.57x @ 800K), 7 mods OK,
  MTP healthy (accept length up to 3.40), PING OK.
- Commits in order: b9b1f35 (phase 1) → b79b840 (MAX_JOBS=2 root-cause fix) →
  73bd9ca (phase 2 800K/8G/0.84) → f6c7648 (KV 7G landing + measured comments) →
  a73f463 (KV 6.5G + util 0.83 + allocator alignment).

## Performance verdict (vs FP4-Local baseline) — BINDING interpretation

- decode: FASTER, and the win GROWS with context depth. tg 30.8 (+8%) at d0 →
  34.4 (+48%) at d16K → 32.7 (+59%) at d64K (pp2048). FP4-Local degrades
  28.6→20.5 t/s by depth; NVFP4 holds ~31-34.
- prefill: SLIGHTLY slower, roughly flat: -11..-14% at depth; only clean d0
  short prompts +8%. 1.5-1.6k t/s both.
- 1-tok latency: 23.6 vs 33.9 ms = -30% (NVFP4 wins; streaming smoothness).
- TTFT: +7-20% slower on 8K prompts (prefill cost); worst pp8192d64k
  47.8s vs 40.5s (no-game; was +90% with game).
- MTP acceptance live during bench: 2.65-3.65 mean length (healthy, k=3).

## Agent-workload synthesis (user-approved view)

- Long-history generation (agent tail state): NVFP4 clearly wins (+48..+59% at
  depth; peak tg holds 37-43 t/s where FP4-Local collapses 39→29).
- Tool-result-heavy turns (big prefill, short reply): ~+7-10%/turn slower
  (prefill + TTFT cost); e2e 300-tok replies: d0 -7%, d16K -9%, d64K ±0%.
- Practical rule: reading light + writing long → NVFP4; heavy ingest + short
  replies → FP4-Local slightly better; both tolerable.

## Co-residency findings (Nov 22)

- With normal desktop apps + a Steam/Proton game (~7 cores) running: decode
  -10%, prefill -10..-35% (worst pp8192d64k -35%, TTFT +67%), 1-tok 42.2 ms.
- Killing the game → everything recovers to KV-7G-standalone level: prefill
  1522-1562, tg 25.7-34.4, 1-tok 23.6 ms. KV 6.5G vs 7G: ZERO measurable cost.
- Conclusion: the tuned profile (6.5G/0.83) is free; CPU-heavy co-resident
  processes, not the KV shrink, were the penalty. Bench-standalone for records.

## KV pool density: two-component model (SOLVED, validated)

Pool = attention KV (token-proportional) + mamba/SSM state (fixed per seq).
Fit over 4 measured points (incl. PR #421's reported 6G → 1,711,646):
- attention: 1.596 KiB/token (build-independent)
- mamba state: ~1.13 GiB/seq × 3 seqs = 3.38 GiB fixed
- Model predicts PR #421 point within +0.7% — holds.
- Consequence: apparent density (KiB/tok) improves with pool size because the
  fixed state amortizes: 3.68 (6G@512K) → 3.31 (6.5G@800K) → 3.08 (7G@800K).
  The "~285K tok/GiB" PR comment is a point estimate, not a constant.
- Caveat: a/M are effective values (include block rounding, kv lcm block 14336).

## Launch OOM history (context for the numbers above)

- MAX_JOBS=16 (image ENV) → ninja -j16 → 16 concurrent cicc compiles of
  fused_moe_120 (97 cutlass targets) → node2 global OOM (kernel log 01:42:35,
  cicc trigger). INDEPENDENT of KV/ctx/util — that is why Phase 1 died.
  MAX_JOBS=2 bounds the spike (measured -j2 peak 88.5% memused, ~4-5G headroom;
  -j4 OOM risk). Compile one-time; cache volume persists (warm: 90 keys/0.28s).
- KV 8G + util 0.84 OOM'd AFTER the KV capacity line (2,721,839 tokens printed)
  during warmup/first requests: 0.5G margin insufficient at 800K (activations
  larger than the 1.3G estimate). KV 7G and 6.5G both come up healthy.

## Artifacts

- ~/bench-results/nvfp4_phase2_kv7g.{json,log}      — Phase 2 standalone (7G)
- ~/bench-results/nvfp4_phase2_kv65_coresident.*    — with game running
- ~/bench-results/nvfp4_phase2_kv65_noGame.{json,log} — final tuned profile
- ~/bench-results/baseline_fp4local_1120-2020.{json,log} — FP4-Local baseline
