# ai — a provider-agnostic AI tool-calling loop for Katari

The tool-calling loop, factored so the model provider is one `use` line. The whole abstraction is a
single request — `ai.infer_step` — so the loop never names a provider; adding one means adding a
module, not editing the loop.

## Modules

- `ai` — the loop. `ai.infer_with_tools(history, tools, max_steps)` runs the model until it stops
  calling tools, dispatching each tool batch concurrently (`parallel for`) and validating every call
  against its tool's schema (`reflection.call_agent`). Also exposes `ai.view_image` (a tool that
  inlines an image for the model to look at); `use ai.serve_session(...)` — the chat-bot serve
  skeleton as one provider (a durable conversation answering `ai.session_message`, with token
  metering off the seam's `usage` and automatic compaction past a threshold; the summary prompt,
  the threshold and the kept tail are overridable scalars); `ai.compact(history, policy)` on its
  own for a hand-rolled loop; `ai.infer_structured[T](history)` — structured output with the
  type as the contract (`reflection.schema_of[T]` reifies the schema, the provider decodes against
  it natively, `json.validate[T]` guarantees the returned value IS a T); and
  `ai.infer_with_region(...)` — the long-lived, structured-concurrency sibling of `infer_with_tools`: the
  model can START background *monitors* (forked fibers that wake it on a schedule), STOP them with
  `cancel_monitor`, and see what it is running with `list_monitors`, all without leaving the conversation,
  and a step failure is absorbed one turn at a time (a notice out, a marker in) instead of ending the loop.
- `ai.types` — the provider-agnostic vocabulary: `turn`, `tool_call`, `tool_result`, `step`, and the
  step's token accounting (`usage`, carried by `step_result` — what `ai.infer_step` returns). Apps
  and providers speak only this; no wire-format detail leaks in.
- `ai.gemini` — a provider over Google's `generateContent` API. `use gemini.provider(model = ...,
  api_key = ...)` serves `ai.infer_step` for the continuation. **Images: yes** — a turn's image files
  inline as `inlineData` parts (freshest turn only, to keep the request small).
- `ai.openai` — a provider over the OpenAI responses API. `use openai.provider(model = ...,
  api_key = ...)` — swap it for `gemini.provider` (or vice versa) with no other change. **Images: no**
  — a turn's files appear to the model as handle notes only.
- `ai.anthropic` — a provider over the Anthropic Messages API. `use anthropic.provider(api_key =
  ...)` — `model` defaults to `claude-sonnet-5` and `max_tokens` (the per-step output cap the API
  requires) to 4096; swap it for either other provider with no other change. **Images: yes** — a
  turn's image files inline as `image` content blocks.

Pure Katari — no FFI sidecar. The only network call is the prelude's HTTP client: the Gemini and
Anthropic providers post an `http.json` value tree with `http.fetch` (so a turn's image `file` leaves
base64 at the send boundary, never on the value plane), OpenAI a plain-body `http.post_json`; every
request body is built and every response parsed as `json` values in Katari.

**The seam is outcome-typed; providers never throw.** `ai.infer_step` / `ai.infer_object` return an
OUTCOME sum — `inferred(result)` on success, `inference_failed(error)` on failure — rather than throwing a
`step_error`. Handing the failure back as a VALUE is what lets each loop decide at its own perform site: the
long-lived `infer_with_region` ABSORBS a failure (it abandons just that turn, tells the model and the
channel, and awaits the next event, so a step failure can no longer end the resident loop), while the
one-shot loops (`reply`, `infer_with_tools`, `infer_structured`, `serve_session`) RE-RAISE it at the call
site as `prelude.throw[step_error]` so the caller's own handler catches it. A deep provider handler can only
throw OUT of its own context, never INTO the loop's — the resume value is the only channel that reaches the
loop.

**Transient retry.** Before surfacing a failure, each provider first retries a transient one — a 429 rate
limit, a 5xx, a dropped connection — with exponential backoff. `retry_attempts` (default 6) and
`retry_base_ms` (default 2000, doubled per retry and capped at one minute) tune it. A fatal error (a
non-retryable 4xx, malformed JSON) or an exhausted budget is what becomes the `inference_failed` outcome.

## Secrets / env

- A model API key, provided to whichever provider you `use`:
  - Gemini: `GEMINI_API_KEY` → `use gemini.provider(api_key = env.get_secret(key = "GEMINI_API_KEY"))`
  - OpenAI: `OPENAI_API_KEY` → `use openai.provider(api_key = env.get_secret(key = "OPENAI_API_KEY"))`
  - Anthropic: `ANTHROPIC_API_KEY` → `use anthropic.provider(api_key = env.get_secret(key = "ANTHROPIC_API_KEY"))`

Store it in the runtime, never in a file: `katari env set GEMINI_API_KEY --secret`. The key is a
`string of private`; it can flow into the provider's auth header but never out to a user boundary.

## Usage

```katari
import ai
import ai.types
import ai.gemini

agent solve(task: string) -> string with io {
  use gemini.provider(
    model = "gemini-3.5-flash",
    api_key = env.get_secret(key = "GEMINI_API_KEY"),
    system = "You are a helpful assistant.",
  )
  ai.infer_with_tools(
    history = [types.turn(role = "user", text = task, files = [])],
    tools = [ai.view_image],   // add your own tools here
    max_steps = 12,
  )
}
```

Imported modules are referenced by their last path segment: `ai.*`, `types.*`, `gemini.provider`.
