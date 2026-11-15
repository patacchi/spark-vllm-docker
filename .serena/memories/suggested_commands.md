# Suggested Commands

Dev/test (CI-equivalent for recipe/runner changes):
```bash
./tests/test_recipes.sh -v
./tests/test_launch_cluster_image_sync.sh
./tests/test_launch_cluster_vllm_pr.sh
./tests/test_launch_cluster_earlyoom.sh
python3 tests/test_launch_cluster_networking.py
```

Focused checks:
- `./tests/test_build_and_copy.sh` — build-and-copy.sh / Dockerfile behavior
- `python3 tests/test_<component>.py` — targeted Python patch/feature tests (e.g. test_vllm_flashinfer_b12x_patch.py)
- `bash -n path/to/script.sh` — shell syntax for edited scripts
- `python3 tests/test_setup_cluster.py` — cluster setup/restore (no live nodes/sudo)
- Some mods have dedicated `tests/test_*_mod.sh` — run the matching one when editing that mod

Recipe dry-run (development validation):
```bash
./run-recipe.sh recipes/<name>.yaml --dry-run --solo          # solo recipes
./run-recipe.sh recipes/<recipe>.yaml --dry-run -n 10.0.0.1,10.0.0.2 --config /dev/null  # cluster recipes
```

Generic utils behave as standard Linux; no project-specific wrappers beyond the above.