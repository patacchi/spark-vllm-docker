# fix-glm47-phantom-fallback

Rescues text swallowed by the GLM-4.7 (glm47) tool-call parser when the model
writes a stray tool-call opener inside prose and never closes it: the phantom
"tool name" (the prose after the opener) is returned to the client as regular
message content instead of being dropped or turned into a bogus tool call.

## Problem

When the model emits a tool-call opener in plain text (quoting or imitating
the call syntax), the parser enters the `TOOL_NAME` state and accumulates the
following prose as a tool name. Depending on what follows:

- **A later real tool call** (with `fix-glm47-stray-tool-start` applied): the
  stray call is closed by the recovery transition and parsing continues —
  but the phantom text itself is still dropped.
- **No later tool call** (streaming): the phantom slot is never closed and
  the accumulated prose is silently dropped at finish.
- **No later tool call** (non-streaming, with
  `fix-glm47-unknown-tool-name` applied): the prose name passes name
  validation and becomes a bogus `ToolCall(name=<prose>, arguments="{}")`.

This mod makes the parser recognize prose-like names (anything that cannot
match a real tool name) and move that text into the parser's deferred-content
channel, which the engine emits as normal message content.

## Scope

- Only `vllm/parser/glm47_moe.py` (the `Glm47MoeParser` subclass); the engine
  (`parser_engine.py`) is unchanged.
- A phantom name is a non-empty name that does not match
  `^[A-Za-z0-9_.\-]{1,128}$` (real OpenAI-style tool names are
  `[A-Za-z0-9_-]{1,64}`). This preserves
  `fix-glm47-unknown-tool-name` behavior: unknown-but-well-formed names are
  still delivered as tool calls.
- Rescued text is emitted verbatim (the prose exactly as the model wrote it
  after the opener; the opener itself is consumed as parser structure).
- Requires `fix-glm47-unknown-tool-name` and `fix-glm47-stray-tool-start`
  (same recipe); the hunks touch disjoint regions of the file and the three
  mods apply in any order. The recipe lists them
  unknown-tool-name → stray-tool-start → phantom-fallback.
- Known limitations:
  - Streaming: a phantom slot consumes one tool-call index, so a later real
    call arrives with a sparse `index` (clients merging by index handle this;
    documented upstream behavior).
  - Single-word prose names that happen to look like identifiers (e.g. one
    Latin word) still pass name validation and can become a tool call.
  - `tool_choice="none"` suppresses all tool events, so there is nothing to
    rescue (pre-existing semantics).
- Idempotent: skips when the `_rescue_phantom_slot` helper is already present.
- `VLLM_PACKAGE_ROOT` overrides the vLLM package location (used by the test).

## Usage

```bash
./run-recipe.sh recipes/modified/glm-5.3-flash.yaml
```

Applied at container start; a vLLM restart is required for an already
running container.

## Status

Prototype for evaluation (vllm#60703 follow-up): the mod addresses the
"prose tail with no subsequent tool call" limitation documented in
`fix-glm47-stray-tool-start`. Adopt or remove based on live validation.
