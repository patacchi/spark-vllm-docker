# Task Completion

Development task is done when (from `docs/AGENT_DEVELOPMENT.md` checklist):
1. Worktree contains only intended changes plus pre-existing user work (`git status`/`git diff`).
2. Changed recipes load and produce expected commands: dry-run each changed recipe with explicit mode + intended node count (`--dry-run --solo` or `--dry-run -n <dummy-list> --config /dev/null`).
3. Relevant focused tests pass:
   - recipe/runner changes → `./tests/test_recipes.sh -v` + the four launch-cluster suites (see `mem:suggested_commands`)
   - touched component → its `tests/test_<component>.py` or `tests/test_*_mod.sh`
   - edited shell script → `bash -n <script>`
   - missing deps → report, don't fake results
4. Documentation, recipe fields, mod paths, and `tests/expected_commands.sh` agree.
5. No secrets or `.env` contents in diff or test output.

Refactoring note: after `rename_symbol`/`safe_delete_symbol` succeed, trust the tool — do not re-read files or rerun builds just to confirm propagation.