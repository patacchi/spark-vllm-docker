# fix-glm47-stray-tool-start

Adds the missing `(TOOL_NAME, "TOOL_START")` recovery transition to the
GLM-4.7 (glm47) tool-call parser state machine so a stray tool-call opener
marker inside plain prose no longer swallows the rest of the message.

## Problem

`vllm/parser/glm47_moe.py` recognizes the tool-call markers through a state
machine. When the model emits a tool-call opener inside prose (for example
when quoting or imitating tool-call syntax), the parser enters the
`TOOL_NAME` state and starts accumulating a phantom tool name. The table has
no `(TOOL_NAME, "TOOL_START")` transition, so:

- the tail of the message accumulates as a phantom name and is dropped at
  finish (assistant output truncated after the stray opener), and
- if a real tool call follows, its opener cannot be recognized either, so
  the real call is lost together with the phantom.

Upstream tracks this as vllm-project/vllm#60703. The proposed fixes (PRs
vllm-project/vllm#60842 and #60813) add the same recovery transition, but
neither patch applies to this build: upstream folds the `ARG_KEY_START`
transition into the main `transitions` dict while this build keeps it in a
separate `arg_tag_transitions` dict spliced at the dict tail, so the hunk
contexts drifted. This mod applies the equivalent 4-line insertion into the
main dict, between the `(CONTENT, "TOOL_START")` and
`(TOOL_NAME, "TOOL_END")` entries.

The transition stays in `TOOL_NAME` and emits `TOOL_CALL_END` followed by
`TOOL_CALL_START`, which closes the phantom call and lets the parser read
the next one.

Known limitation (same as upstream): recovery only helps when a real tool
call follows the stray opener. A prose tail with no subsequent tool call is
still swallowed; a full fix would need the parser to emit the phantom text
as fallback content.

## Scope

- Only inserts the recovery transition into `vllm/parser/glm47_moe.py`.
- The patch is a context hunk, not context-free: an insertion has no changed
  line of its own to anchor on, so the surrounding transition entries are
  the anchor and the apply fails closed if that region drifts.
- Coexists with `fix-glm47-unknown-tool-name` (disjoint region of the same
  file; the glm-5.3-flash recipe lists both).
- Idempotent: skips when the `(TOOL_NAME, "TOOL_START")` transition is
  already present.
- `VLLM_PACKAGE_ROOT` overrides the vLLM package location (used by the test).

## Usage

```bash
./run-recipe.sh glm-5.3-flash
```

Applied at container start; a vLLM restart is required for an already
running container.
