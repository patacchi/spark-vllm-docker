generated_at: 2026-11-15T16:25:00+0900
source commit: 5f8aa0f
coverage: Repo entry points, directory layout incl. .github/ submodules, workspace .mcp.json, .serena/ config, agent routing invariants (vLLM-on-DGX-Spark orchestration repo)
thoroughness: Full-replacement restructure via one read-only Explorer delegation (root entry points, recipes/, mods/, docker/, docs/, tests/, wheels/, .github/, .mcp.json, .serena/)

# Core

## Overview

Bash + Python orchestration for running vLLM on one or more NVIDIA DGX Spark systems (single node to multi-node Ray/PyTorch-distributed clusters, NCCL/InfiniBand-RDMA). Development host is Linux; targets are DGX Spark (integrated NVIDIA GPU, shared CPU/GPU UMA memory).

## Entry points

Repo-root launchers:
- `run-recipe.sh` → `run-recipe.py`: recipe launcher (model/image/launch orchestration); `--dry-run` validates generated commands
- `launch-cluster.sh`: low-level container launcher; writes `.env` settings
- `build-and-copy.sh`: Docker image build (runner from DockerHub `eugr/spark-vllm:latest` by default; `--use-wheels`, `--rebuild-vllm`, `--exp-b12x` variants)
- `hf-download.sh` / `hf-cache.py`: model download and HF cache distribution
- `setup-cluster.sh` / `setup-cluster.py` / `setup_cluster_node.py`: CX7 networking + passwordless SSH cluster setup (with `--doctor`/dry-run/restore)
- `autodiscover.sh`: cluster node discovery
- `cluster_topology.py`: topology helpers shared by launcher

## Layout

- `recipes/`: YAML launch recipes; `recipes/3x|4x|8x-spark-cluster/` = topology-bound nested recipes; `recipes/modified/` = archived alternatives (do not edit unless targeted); some recipes chain multiple `mods/` entries
- `mods/<name>/run.sh`: runtime patches applied to a freshly created vLLM container (each dir also carries its README and .patch file)
- `docker/`: build-time vLLM patches (`patch_vllm_*.py`, `apply_patch_series.sh`) and wheel/image tooling
- `docs/`: AGENT_RUNBOOK.md (operation) / AGENT_DEVELOPMENT.md (development) / NETWORKING.md (cluster networking)
- `tests/`: per-component suites (see `mem:suggested_commands`)
- `wheels/`: vLLM/FlashInfer wheels released alongside images
- `.github/`: git submodules `agents` and `skills` plus `workflows/` CI
- `.mcp.json`: workspace MCP servers (git-ro, session-close, harness-info, context-usage, context7, serena-mcp)
- `.serena/`: Serena project config (`project.yml`, `project.local.yml`) and the `memories/` map store

## Agent routing invariant

AGENTS.md routing: operational tasks (real builds, downloads, `--setup`, launches) → `mem:operation`; development → `mem:development`; recipe `--dry-run` validation counts as development.

## Read next

`mem:development` for dev workflow and `mem:operation` for runbook pointers, `mem:conventions` for code style, `mem:tech_stack` for toolchain, `mem:suggested_commands` for commands, `mem:task_completion` for done-checks, `mem:troubleshooting` for live-server debugging and known runtime issues.
