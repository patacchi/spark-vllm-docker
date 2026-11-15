# Conventions

Recipes (`recipes/*.yaml`):
- Required fields: `recipe_version: "1"`, `name`, `container`, `command` (uses `{port}`/`{host}` placeholders backed by `defaults` or runner overrides).
- Optional: `description`, `model`, `mods`, `build_args`, `defaults`, `env`, `cluster_only`, `solo_only` (never both cluster_only and solo_only true).
- General recipes flat in `recipes/`; topology-bound ones under `3x|4x|8x-spark-cluster/`. Nested recipes referenced by path.
- Mods referenced with repository-relative paths (`mods/<name>`).
- Omit `--distributed-executor-backend`, `--nnodes`, `--node-rank`, `--master-addr`, `--master-port`, `--headless` — runner/launch-cluster.sh supply them.
- Quote YAML values where type coercion could change intent (e.g. `"1"` for recipe_version).
- Prefer container `vllm-node` unless documented alternate build required.
- Update docs and `tests/expected_commands.sh` when a documented command changes.

Mods: always applied at runtime to a freshly created vLLM container; no migration/legacy-detection/reversal logic — `mods/<name>/run.sh` must apply cleanly or detect repeat application within the same fresh-container launch.

Build-time vLLM fixes: applied to vLLM source before wheel build (`docker/patch_*.py`, `apply_patch_series.sh`); wheels ship with fixes; no runtime compatibility paths for unpatched wheels.

General: preserve executable bits on shell/Python entry points; Bash with arrays; Python 3.10+ with pathlib; keep changes scoped; no credentials/machine IPs/`.env` values in tracked files.