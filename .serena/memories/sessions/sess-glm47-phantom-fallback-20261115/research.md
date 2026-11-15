# Research — spark-vllm-docker experimental/PR conventions (Task 3, for mod A framing)

Collected 2026-11-15 via gh CLI + repo inspection. Merged into this entry by caller; scratch deleted.

## 1. "Experimental" in this repo — two DIFFERENT meanings

(a) **Build/stack-level experimental flags** (`--exp-b12x`, `--exp-mxfp4`, `exp-b12x`/`exp-w4a16`
mods): whole experimental STACKS or runtime patch bundles for them. mods/exp-b12x and exp-w4a16
are LARGE multi-step runtime patchers (env validation, PR fetching via patch-diff URLs, wheel
rebuilt, self-tests) — NOT simple one-file parser mods. README §"Experimental b12x loader" shows
the user-facing wording: "still experimental, so please update the repository often".
(b) **Commit-level**: "Cluster setup (experimental)", "qwen3.8-next and deepseek vision exp
support" — experimental = not-yet-stabilized feature, no dedicated issue/label process.

## 2. Issue/PR conventions (eugr/spark-vllm-docker)

- Labels exist only for the 9 GitHub defaults (bug/documentation/enhancement/good first issue/
  help wanted/invalid/question/wontfix/duplicate) — NO "experimental" label. Issues/PRs are
  typically posted WITHOUT labels.
- Issue title style: lowercase-ish descriptive sentence + parenthetical context, e.g.
  "#459 DFlash2 on GLM-5.3-Flash-FP4 (B12X, 2x GB10): working recipe, three startup blockers,
  measured A/B"; often "(tracked upstream: vllm-project/vllm#67684)" suffix for upstream-linked.
- PR title style: conventional-commit-ish prefix: "mods: add gb10-wfe-waits (...)",
  "fix(launch-cluster): ...", "feat(hf-download): ...", "recipes: ..." (our #462 used a bare
  sentence title "Add fix-glm47-unknown-tool-name mod" — also accepted).
- Issue body pattern (excellent example: #459, our #463): TL;DR / Summary → Root cause (verified)
  → why it matters → Reproduction → Workaround/mod → links to related issues (both directions).
- PR body pattern (our #462 as the direct template): Overview (with upstream tracking link) →
  Claims (bulleted, with live-measured evidence) → Validation (test suites + live checks) →
  AI-disclosure footer ("drafted with the help of an AI coding assistant and reviewed by a human
  before posting"). PR #458 shows the richer ops-mod pattern: what/why, numbered patch steps,
  opt-out env vars, measured table, "Tested on GB10 only. Related: ..." tail.
- Cross-referencing: issues freely reference other issues and upstream vllm-project/vllm issues.

## 3. Implications for the 3 planned posts

- There is NO dedicated experimental process: the repo treats "experimental" as prose wording +
  warnings, not labels or special flows. So mod A's "experimental" framing = say it in the
  issue/PR title/body (e.g. title suffix "(experimental)") + a Status/Limitations section +
  NOT wiring it into the default recipe OR wiring with a clear caveat. Precedent: #459 posted a
  working-but-experimental recipe as a REFERENCE (not merged as default); DFlash2 got its own
  feature-request issue (#446) first.
- Recommended shape for A: issue "phantom-fallback (experimental): rescue prose swallowed by
  glm47 parser after stray tool-call opener (related: vllm-project/vllm#60703, #463)" + PR with
  experimental wording in README + explicit "not wired into any default recipe" or
  "wired into the ops recipe with a caveat" decision point. The adopt decision (recipe wiring)
  can be part of the PR description rather than the code.
- For B/C: follow #462's structure exactly (it is accepted style; B is already #462).

## 4. Push/branch mechanics for PRs from this fork

- origin push URL = no_push (workaround: `gh pr create --repo eugr/spark-vllm-docker --head
  patacchi:<branch>` requires the branch to exist on the patacchi fork on GitHub — so pushing
  needs a real remote; earlier PR #462 was created from branch fix-glm47-unknown-tool-name on
  the fork. A dedicated push remote (the fork's git@github.com:patacchi/... URL) can be added
  without touching origin's no_push. This is a user decision at PR time.)
