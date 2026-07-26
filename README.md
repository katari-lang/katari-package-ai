# ai — a provider-agnostic AI tool-calling loop for Katari

The tool-calling loop, factored so the model provider is one `use` line. The whole abstraction is a
single request — `ai.infer_step` — so the loop never names a provider; adding one means adding a
module, not editing the loop.

## Modules

- `ai` — the loop. `ai.infer_with_tools(history, tools, max_steps)` runs the model until it stops
  calling tools, dispatching each tool batch concurrently (`parallel for`) and validating every call
  against its tool's schema (`reflection.call_agent`). Also exposes `ai.view_file` (a tool that reads an
  attachment — an image, a PDF, or a text document) and the `file_kind` classification every provider
  dispatches over (see **Attached files** below); `use ai.serve_session(...)` — the chat-bot serve
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
  api_key = ...)` serves `ai.infer_step` for the continuation. **Files: images, PDFs, audio, video and
  text** — the binary kinds all ride the one `inlineData` part keyed by their mime type, restricted to the
  formats Gemini documents; text inlines as text (freshest turn only, to keep the request small).
- `ai.openai` — a provider over the OpenAI Chat Completions API. `use openai.provider(model = ...,
  api_key = ...)` — swap it for `gemini.provider` (or vice versa) with no other change. **Files: text
  only** — a message's content here is a plain string, which text needs no wire shape to ride; an image or
  a PDF would need a `data:` URI (base64 inside a string), which the `http.json` slot contract cannot
  build, so those report to the model as unreadable rather than being faked.
- `ai.anthropic` — a provider over the Anthropic Messages API. `use anthropic.provider(api_key =
  ...)` — `model` defaults to `claude-sonnet-5` and `max_tokens` (the per-step output cap the API
  requires) to 4096; swap it for either other provider with no other change. **Files: images, PDFs and
  text** — an `image` content block (jpeg / png / gif / webp), a `document` content block (base64
  `application/pdf`), and inlined text respectively; audio, video and any other image format have no
  Messages API block, so they report as unreadable.

Pure Katari — no FFI sidecar. The only network call is the prelude's HTTP client: the Gemini and
Anthropic providers post an `http.json` value tree with `http.fetch` (so an inlined `file` leaves base64
at the send boundary, never on the value plane), OpenAI a plain-body `http.post_json`; every request body
is built and every response parsed as `json` values in Katari.

## Attached files

How a file reaches the model follows from ONE datum — its recorded content type. `ai.classify_file` turns
that string into the closed `ai.file_kind` sum (`image_file`, `document_file`, `text_file`, `audio_file`,
`video_file`, `opaque_file`), and each provider MATCHES it, one arm per shape its own API accepts:

| content type | Anthropic | Gemini | OpenAI |
| --- | --- | --- | --- |
| `image/*` | `image` block — jpeg, png, gif, webp | `inlineData` part — png, jpeg, webp, heic, heif | unreadable note |
| `application/pdf` | `document` block | `inlineData` part | unreadable note |
| `text/*`, `application/json` / `xml` / `csv` | inlined text | inlined text | inlined text |
| `audio/*`, `video/*` | unreadable note | `inlineData` part — the documented audio / video formats | unreadable note |
| anything else (`application/octet-stream`, an archive, an unrecorded or freed handle) | unreadable note | unreadable note | unreadable note |

A family is not a licence: each provider inlines only the MEDIA TYPES its own API documents, and a type
inside the family but outside that list takes the unreadable note instead. `image/*` is the case that
bites — a scanned fax arrives as `image/tiff`, a diagram as `image/svg+xml`, and no API here decodes
either. Shipping one hopefully is a 400 that fails the step and takes the turn with it; the note costs the
model one honest sentence and the conversation continues.

Two rules make this honest rather than merely wide:

- **Inlined text is capped and the cut is announced.** `ai.file_text_note` prefixes the handle and the
  content type, then the decoded text, cut at 24000 code points (~6k tokens) with an explicit `TRUNCATED`
  marker stating how much of the file was sent. A model that cannot tell it is holding a fragment answers
  confidently from the fragment, so the tail is never dropped silently.
- **An unreadable file SAYS it is unreadable.** `ai.unreadable_file_note` states the content type and that
  the content is absent, and tells the model to report that instead of guessing — a bare handle with no
  explanation is what makes a model invent a file's contents.

The content type is taken as recorded; nothing here sniffs a file's bytes to guess a better type. A sender
that labels a `.md` as `application/octet-stream` therefore gets the unreadable note, and the fix belongs
at the upload boundary that recorded the type. What IS normalized is the label's own syntax — RFC 2045
makes a media type case-insensitive and lets a sender append parameters, so `Text/Plain; charset=utf-8`
reads as `text/plain` for both the dispatch and the provider's `media_type` field.

The table holds for a file the user attached to a turn AND for a file a tool handed back (what
`ai.view_file` returns) — one dispatch, both positions:

- **Anthropic** puts the result's file blocks INSIDE the `tool_result`'s own `content` array, which the
  Messages API documents as taking `text`, `image`, `document` or `search_result` blocks. Inside rather
  than beside, because a user message carrying tool results may only append *text* after them, and because
  keeping tool output inside the `tool_result` block is what marks it untrusted.
- **Gemini** renders a result's files as parts of the same user content, as it always did.
- **OpenAI**'s tool message takes a string (its array form admits `text` parts only), so a result's files
  contribute their text or their unreadable note — the same two outcomes as its turn path.

**Only the newest file-bearing element inlines.** Every step re-sends the whole conversation, so a file
that inlined forever would be uploaded again on every step — a 10 MB PDF in a thirty-step tool loop is
300 MB of request bodies, and prompt caching does not help (it spares the model's compute on a prefix, not
the client's bytes). `ai.inline_from_index` keeps the newest file-bearing history element inline and
degrades older files to their handle note, which `ai.view_file` re-reads on demand. The window is keyed to
FILES, not to position: a step that appends a tool call or a tool result leaves every earlier rendered byte
identical, so the prompt-cache prefix still matches; only a genuinely new attachment moves it.

**The textual arm puts a file's content on the value plane, and privacy travels with it.** An inlined text
file is read, decoded and placed in message text, so a `file of private` — a Gmail attachment fetched with
an OAuth token, anything downloaded through a private request — makes that message text private too, where
the same file previously contributed only an opaque handle. Toward a provider this changes nothing (a
model API is a private-capable boundary and always was). It matters where an app forwards tool results or
history text onward: a channel post, a webhook reply, a run result read by a user. Those are public
boundaries, so a private-tainted string is refused there — the same rule that already governed the file
itself, now reaching the text it decodes to. An app that echoes raw tool results to a channel should echo
its own summary instead, or keep the private file's content out of what it forwards.

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
    tools = [ai.view_file],    // add your own tools here
    max_steps = 12,
  )
}
```

Imported modules are referenced by their last path segment: `ai.*`, `types.*`, `gemini.provider`.
