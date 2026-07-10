# ai — a provider-agnostic AI tool-calling loop for Katari

The tool-calling loop, factored so the model provider is one `use` line. The whole abstraction is a
single request — `ai.infer_step` — so the loop never names a provider; adding one means adding a
module, not editing the loop.

## Modules

- `ai` — the loop. `ai.infer_with_tools(history, tools, max_steps)` runs the model until it stops
  calling tools, dispatching each tool batch concurrently (`parallel for`) and validating every call
  against its tool's schema (`reflection.call_agent`). Also exposes `ai.view_image` (a tool that
  inlines an image for the model to look at).
- `ai.types` — the provider-agnostic vocabulary: `turn`, `tool_call`, `tool_result`, `step`. Apps
  and providers speak only this; no wire-format detail leaks in.
- `ai.gemini` — a provider over Google's `generateContent` API. `use gemini.provider(model = ...,
  api_key = ...)` serves `ai.infer_step` for the continuation.
- `ai.openai` — a provider over the OpenAI responses API. `use openai.provider(model = ...,
  api_key = ...)` — swap it for `gemini.provider` (or vice versa) with no other change.

Pure Katari — no FFI sidecar. The only network call is the prelude's `http.post_json`; request
bodies are built and responses parsed as `json` values in Katari.

## Secrets / env

- A model API key, provided to whichever provider you `use`:
  - Gemini: `GEMINI_API_KEY` → `use gemini.provider(api_key = env.get_secret(key = "GEMINI_API_KEY"))`
  - OpenAI: `OPENAI_API_KEY` → `use openai.provider(api_key = env.get_secret(key = "OPENAI_API_KEY"))`

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
