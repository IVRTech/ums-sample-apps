# OpenAI BS AGI Sample Flow Notes

This note explains the three Python AGI samples and focuses on `detect_intent()` and the grammar strings they build.

## Files and intent

- `agi_openaibs.py`: baseline conversational loop.
- `agi_openaibs_functioncall.py`: baseline + OpenAI function/tool call round-trip.
- `agi_openaibs_instructions.py`: primes a welcome turn with explicit `instructions` and `method`, then continues like baseline.

## Shared call loop pattern

Across all variants, the AGI app:

1. Reads runtime options from Asterisk channel variables (`LANGUAGE`, `VOICENAME`, `TRANSCRIPTION_MODEL`, `MODALITIES`, etc.).
2. Builds grammar URI strings for UniMRCP's built-in recognizers.
3. Calls `SynthandRecog` with prompt + grammar + options.
4. Reads completion state from `RECOG_STATUS` and `RECOG_COMPLETION_CAUSE`.
5. Reads result payload fields from `RECOG_INSTANCE(...)` and chooses the next prompt.

## Deep dive: `detect_intent()` and grammar design

In `detect_intent()`, the app sets:

```python
self.grammars = "%s$%s" % (self.compose_speech_grammar(), self.compose_dtmf_grammar())
```

That means two recognizers are active in one `SynthandRecog` turn:

- left side: speech grammar (`builtin:speech/transcribe`)
- right side: DTMF grammar (`builtin:dtmf/digits`)
- `$` delimiter: combines both grammars for one MRCP recognition request.

### Speech grammar

`compose_speech_grammar()` starts with:

- `builtin:speech/transcribe`

Then appends query parameters:

- first with `?`
- subsequent with `;`

Typical parameters include:

- `transcription-model=<model>` (for example `gpt-4o-mini-transcribe`)
- `modalities=<value>` (for example `text,audio`)

Function-call variant can append an additional parameter:

- `tools-json=<escaped compact json>` loaded from the file in `JSONPATH`.

Instruction variant can set `method` and `instructions` during the initial welcome trigger path.

### DTMF grammar

`compose_dtmf_grammar()` starts with:

- `builtin:dtmf/digits`

Then appends the same `transcription-model` and `modalities` pattern, allowing button presses to participate in the same turn machinery.

### Why both grammars are used

Using `speech$dtmf` provides mixed input handling in one prompt/recognize cycle:

- voice callers can speak naturally,
- keypad users can still interact,
- completion/cause handling remains centralized in one loop.

## `SynthandRecog` argument construction

All variants build args like:

```python
"\\\"%s\\\",\\\"%s\\\",%s" % (prompt, grammars, options)
```

Where:

- `prompt` is spoken before listening (or blank space when only recognition state update is desired).
- `grammars` is the combined grammar string.
- `options` carries recognizer/synth options plus language/voice (`spl`, `vn`).

## Output extraction behavior

`get_prompt()` inspects:

- `RECOG_INSTANCE(0/0/response/output/0/content/0/type)`

Then chooses where to read content from:

- `input_audio` -> `/transcript`
- `uri` -> `/audio`
- default -> `/text`

This makes the AGI resilient to text-only, transcript-first, or URI/audio style responses.

## Variant-specific additions

### Function call sample

- Detects when `response/output/0/type == function_call`.
- Reads `name`, `arguments`, `call_id`.
- On next loop, sends a `function_call_output` conversation item via grammar parameter `conversation-item-json`.
- Demo implementation hardcodes weather output `24C` for `get_weather`.

### Instructions sample

- Calls `trigger_welcome_state()` before normal loop.
- Adds `method` and `instructions` to grammar for that first turn.
- If the welcome trigger does not return `OK` + `000`, it aborts early.

## Practical debugging tips

- Always log the final grammar string and options when tuning behavior.
- Verify channel variables exist in dialplan (`LANGUAGE`, `VOICENAME`, `MODALITIES`, model, etc.).
- If no tools are invoked in function mode, check `tools-json` escaping and JSON schema file path.
- If first turn in instructions mode fails, check `METHOD` and quoting of `INSTRUCTIONS` in dialplan.
