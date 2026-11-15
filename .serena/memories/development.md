# Development Workflow

Guide: `docs/AGENT_DEVELOPMENT.md` (authoritative). Applies to inspection, fixes, features, reviews, tests, changes to scripts/recipes/mods/Dockerfiles/docs.

Key invariants:
- Never run `--setup`, `--build-only`, `--download-only`, `--force-*`, `--discover`, `build-and-copy.sh`, or `hf-download.sh` for development validation. No real deployments from dev tasks.
- Dry-run rules: always pass `--solo` or explicit dummy `-n` node list (e.g. `-n 10.0.0.1,10.0.0.2`); without it the runner may read `.env` or autodiscover the network. Cluster dry-runs still read interface settings from `.env` even with `-n` → add `--config /dev/null` to validate independently of local config. Nested recipes must be passed by path.
- A two-node dry-run does not validate a recipe whose command needs 4/8 nodes; validate declared mode and node count.
- Tests use fixtures/mocks; never substitute a real deployment.
- No secrets, machine IPs, cache paths, `.env` values, or generated launch output in tracked files or test output.
- Keep changes scoped; preserve unrelated user changes and existing local artifacts.

Operation side (if user explicitly requests real runs): `mem:operation`.

Live debugging on this host: see `mem:troubleshooting`.