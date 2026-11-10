# Stop Reasons by Provider: How Did the Call End?

**Source aside:** Ch. 2 (the Reader Contract's worked example) and Ch. 1 (the half-sentence brief).
**Source incident:** `war-stories/forge-max-tokens.md`.
**Last reviewed:** 2026-11-10.
**Valid as of:** Q4 2026, against the Anthropic Messages API, the OpenAI Chat Completions and Responses APIs, and the Gemini API.

## What the body says

Ch. 2's durable shape: explicitly check the signal for *how the call ended*, branch on it, and only ship in the clean-finish branch. The body names no provider. This entry names the field, the clean-finish value and the truncation value for each provider, plus a Ruby and a TypeScript version of the check.

## The fields

| Provider / API | Field | Clean finish | Truncated by your output limit | Other values worth handling |
| --- | --- | --- | --- | --- |
| Anthropic Messages | `stop_reason` | `end_turn` | `max_tokens` | `stop_sequence`, `tool_use`, `pause_turn`, `refusal`, `model_context_window_exceeded` |
| OpenAI Chat Completions | `choices[].finish_reason` | `stop` | `length` | `tool_calls`, `content_filter`, `function_call` (legacy) |
| OpenAI Responses | `status` + `incomplete_details.reason` | `status: "completed"` | `status: "incomplete"` with `reason: "max_output_tokens"` | `failed`, `cancelled`; incomplete reasons `content_filter`, `max_messages`, `steered` |
| Google Gemini | `candidates[].finishReason` | `STOP` | `MAX_TOKENS` | `SAFETY`, `RECITATION`, `MALFORMED_FUNCTION_CALL`, `CONTINUATION`, and others |

Notes that bite:

- **"Clean" depends on the call.** A `tool_use` / `tool_calls` stop is the expected end of a tool-calling turn, not a failure. Decide per call site which values count as *done*.
- **Anthropic's `model_context_window_exceeded`** is also a truncation, but from the context window rather than your `max_tokens` setting.
- **OpenAI Responses with reasoning models:** reasoning tokens count against `max_output_tokens`. An incomplete response can contain reasoning items and no visible text, so check `status`, not just whether text came back.
- **Gemini's `STOP`** covers both a natural stop and a provided stop sequence. `CONTINUATION` means generation paused at the per-request limit and can resume with the returned continuation token.

## The check, as of this entry

Ruby, with the official `anthropic` gem (`stop_reason` and content block `type` are symbols; `content` is an array of blocks):

```ruby
def ship_brief_safely(response)
  unless response.stop_reason == :end_turn
    queue_for_retry(response)
    return
  end
  text = response.content.select { |block| block.type == :text }.map(&:text).join
  ship(text)
end
```

TypeScript, with the official `@anthropic-ai/sdk` package:

```typescript
function shipBriefSafely(message: Anthropic.Message): void {
  if (message.stop_reason !== "end_turn") {
    queueForRetry(message);
    return;
  }
  const text = message.content
    .filter((block): block is Anthropic.TextBlock => block.type === "text")
    .map((block) => block.text)
    .join("");
  ship(text);
}
```

TypeScript, with the official `openai` package and the Responses API:

```typescript
function shipBriefSafely(response: OpenAI.Responses.Response): void {
  if (response.status !== "completed") {
    // incomplete_details.reason === "max_output_tokens" is the truncation case
    queueForRetry(response);
    return;
  }
  ship(response.output_text);
}
```

## Sources

- Anthropic, [Handling stop reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons), and the `Anthropic::StopReason` and `Anthropic::TextBlock` models in [anthropic-sdk-ruby](https://github.com/anthropics/anthropic-sdk-ruby).
- OpenAI, `ChatCompletion.Choice.finish_reason`, `Response.status` and `Response.incomplete_details` types in [openai-python](https://github.com/openai/openai-python).
- Google, [Gemini API: generateContent reference](https://aistudio.google.com/api-reference/generate-content), `FinishReason` enum.
