# Tech Stack

- Bash: primary orchestration language. `#!/bin/bash`, quote expansions, arrays for command assembly (not string concatenation).
- Python: 3.10+ (repo code). Serena LSP uses python language server. Prefer `pathlib.Path`, argument lists for subprocess, helpers in `run-recipe.py`.
- Docker: multi-stage image build (`Dockerfile` regular, `Dockerfile.mxfp4` B12X variant); images `eugr/spark-vllm:latest` (nightly) and `eugr/spark-vllm-b12x:latest`; container name `vllm-node` default. WSL/CUDA-UMA specific patches included.
- vLLM: rapidly moving target; built-in vLLM fixes are applied at build time (in `docker/`) and included in exported wheels — never add runner-stage patching for unpatched wheels.
- fastsafetensors / InstantTensor: high-performance model loaders; local patches at repo root (`fastsafetensors*.patch`) and `mods/instanttensor-*`.
- No package manager manifest in repo (no pyproject/requirements at top level); tests run with system python3. Git remote: fetch-only origin (push disabled by URL rewrite).
- Host: Linux, `uv`/`serena` available; NVIDIA DGX Spark target with shared CPU/GPU UMA.