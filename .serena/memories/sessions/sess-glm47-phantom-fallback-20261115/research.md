# Research — sess-glm47-phantom-fallback-20261115 (Task 2 design analysis)

Explorer delegation e744a510; merged by caller 2026-11-15. Sources: /tmp/task2-src/{glm47_moe.py (280 L),
parser_engine.py (1268 L)} — container-extracted copies of build 8244d1c44 (1-based line numbers below).
Marker literals are `<OPENER>`=TOOL_CALL_START / `<CLOSER>`=TOOL_CALL_END (raw sequences never in chat).
Gates all passed; no core map mismatch (map fresh from the Task 1 close, source commit 5f8aa0f).

## Key code facts (verified verbatim)

- glm47_moe.py: `import regex as re` (NOT stdlib re); imports ParserEngine, EventType, ParserState,
  Transition; does NOT import ExtractedToolCallInformation (engine imports it from
  `vllm.entrypoints.generate.base.protocol` L17-24 with DeltaFunctionCall/DeltaMessage/DeltaToolCall/
  FunctionCall/ToolCall). Module constants L42-47 (markers), _ARG_RE/_TAIL_ARG_RE L59-63,
  glm47_moe_config L90, Glm47MoeParser(ParserEngine) L238 (overrides _emit_name_delta L261,
  _handle_tool_end L266 — both extend via pre-processing + super()), extract_content_ids L272,
  extract_reasoning L277. content_events L231: TOOL_NAME state → EventType.TOOL_NAME (prose after
  opener NEVER becomes TEXT_CHUNK).
- parser_engine.py: ToolCallSlot L48 (_args_parts/_args_joined/name_sent + name + args property);
  ParserEngine L79; _deferred_content init L122; finish_streaming L205-209 (gate: events or deferred);
  _events_to_delta L780-872; _handle_tool_name L897-899 (slot.name += event.value — VERBATIM);
  _is_valid_tool_name L432-437 / _accept_tool_name L439-443 (mod1: accepts all non-falsy);
  _try_extract_name L1170-1177 (regex over slot.args → None for phantom: args empty);
  _handle_tool_end L953-995 (rejected name → slot untouched, name stays only in slot.name — dropped
  everywhere); _build_extracted_result L1180-1229 (content only from delta.content; rejected slots
  excluded from tool_calls, text LOST; empty-args slot → args_json="{}" → with mod1 a bogus
  ToolCall(name=prose, arguments="{}")); _single_pass_parse L719-749 calls self._engine.finish()
  DIRECTLY (bypasses finish_streaming!) → parse() path needs the _build_extracted_result sweep;
  extract_tool_calls() L577-594 DOES go through finish_streaming.

## Deferred content mechanics (L780-872, decisive details)

- Gate L843: `if self._deferred_content and (not seen_tool_event or not tool_call_deltas)` —
  emission per-delta (a batch with a real call's deltas suppresses THIS delta only; flushes on the
  next tool-free delta or in the final chunk even when tool calls exist).
- carried_deferred L792 (from previous deltas) vs deferred_now L847 (TEXT_CHUNKs after a tool event
  in this batch, L802).
- PITFALL L866: `elif not finished: self._deferred_content = content_str` OVERWRITES carried deferred
  when content_str is ws-only, no tool slots yet... reachable in the phantom scenario (recovery delta
  batch with real-call deltas while deferred holds the rescue) → MITIGATION: set
  `self._content_has_nonws = True` when rescuing (chain then hits `if self._content_has_nonws: pass`).
- tool_choice="none" (_suppress_tool_calls): handlers never run → no rescue (documented limitation).

## Confirmed plan corrections (mem:troubleshooting "Current plan" Task 2)

1. `_handle_tool_end` override covers only CLOSED phantoms; UNCLOSED (EOS in TOOL_NAME, no TOOL_END)
   needs a `finish_streaming` pre-sweep (robust whether or not StreamingParserEngine.finish()
   synthesizes TOOL_CALL_END — not verifiable from these two files; empirical evidence says it does not).
2. "engine emits deferred at TOOL_CALL_END" imprecise — gate fires in any tool-delta-free batch
   (earlier/broader); suppression per-delta.
3. `_build_extracted_result` override is REQUIRED (parse() bypasses finish_streaming) — also flushes
   gate-suppressed leftover deferred.
4. Mod1 changes the non-streaming failure mode: bogus ToolCall(name=prose, arguments="{}") instead of
   silence (stray-tool-start README wording predates mod1).

## REVISED design (implemented + offline-verified 17/17) — supersedes the Explorer draft

The Explorer's draft #6 (`_build_extracted_result` override) FAILED the parse() path:
_single_pass_parse calls `self._build_extracted_result()` with NO deltas and parse() returns
`delta.content` (L745), NOT tool_call_info.content — rescued text never reached parse() output.
Also learned in offline debugging: with thinking enabled (default), pre-call prose arrives as
REASONING_CHUNK (initial state REASONING) — S5-style "content preserved" checks must use
chat_template_kwargs={"thinking": False} or assert reasoning. Final design (6 pieces, NO new
imports, NO _build_extracted_result/_events_to_delta overrides):
1. Module-level `_PHANTOM_NAME_RE = re.compile(r"^[A-Za-z0-9_.\-]{1,128}$")` (file uses `regex as re`).
2. `_is_phantom_name(name)`: non-empty AND not fullmatch.
3. `_accept_tool_name` override: phantom → False; else super() (preserves mod1).
4. `_rescue_phantom_slot(slot)`: skip if not name/name_sent/not phantom; text = stripped name +
   (" " + slot.args if args non-empty); `self._deferred_content += text`;
   `self._content_has_nonws = True` (L866 overwrite guard); clear slot.name/_args_parts/_args_joined.
5. `_handle_tool_end` override EXTENDED (existing strip + rescue + super) — closed phantoms.
6. `finish_streaming` override: sweep all slots, then super() — unclosed phantoms (robust whether
   or not engine finish() synthesizes TOOL_CALL_END: swept slots are empty → rejected → no delta
   → finish-batch gate still flushes deferred).
7. `_single_pass_parse` override: super() → sweep slots → flush leftover _deferred_content into
   content — covers parse() (bypasses finish_streaming; single-batch gate suppression when a real
   call produced deltas).
- Path coverage: streaming closed (rescue at TOOL_CALL_END → gate flush on next tool-free delta),
  streaming unclosed (finish_streaming sweep), parse() both (single_pass sweep+flush),
  extract_tool_calls() both (pass-1 rescue + finish_delta flush). No double-emission (rescued
  slots are emptied; flushed deferred is cleared).
- Index handling: mod2 recovery emits TOOL_CALL_END(phantom idx) + TOOL_CALL_START(idx+1) — per-slot.
  Streaming index gap (phantom consumes an index) — accept, document.
- Verbatim emission chosen (slot.name = exactly what the model wrote after the opener).

## Offline behavioral verification (container, python3, module loaded from patched file copy)

17/17 PASS (offline_parser_test.py, markers built from module constants — raw glyphs must never
transit chat/test files: the first fixture attempt used visually-copied glyphs that were NOT the
real marker strings and silently mis-parsed). Scenarios: S1 streaming unclosed phantom → content;
S2/S8 streaming phantom+real call (one-shot and across deltas) → both; S3/S4 parse() unclosed /
phantom+real → content + real call; S5 thinking=False normal call (name/args/content intact);
S5b thinking=True pre-call prose → reasoning (engine semantics, regression); S6 unknown-but-valid
identifier still delivered (mod1 semantics); S7 phantom after THINK_END → content. Note: engines'
ARG values stream as ARG_VALUE_CHUNKs incl. the raw marker text; args JSON conversion unchanged.
Idempotency token count in patched file = 4 (def + 3 call sites).

## Open questions (progress.md has the full list; key ones)

- StreamingParserEngine.finish() behavior for open TOOL_NAME slots (not in study files; design robust
  to both; verify empirically with a forced unclosed-opener probe on live).
- tool_index assignment sequence (assumed 0,1,2... per TOOL_CALL_START; verify in
  streaming_parser_engine.py if a prototype probe shows otherwise).
- Phantom-with-args rendering (raw vs converted) — chose raw with space join; tunable.
- Heuristic false positives: single-word prose names ("hello") still pass as tool names (regex matches)
  → junk call remains for those; tune live if the 65k corpus shows it.