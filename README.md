# ai — a provider-agnostic AI tool-calling loop for Katari

The tool-calling loop, factored so the model provider is one `use` line. The whole abstraction is a
single request — `ai.infer_step` — so the loop never names a provider; adding one means adding a
module, not editing the loop.

## Modules

- `ai` — the loop, and everything built over it:
  - `ai.take_turn(conversation, tools, max_steps)` — ONE model turn as a VALUE (`session_turn` or
    `failed_turn`), tool batches dispatched concurrently (`parallel for`) and every call validated
    against its tool's schema (`reflection.call_agent`). Everything below is a composition over it.
  - `ai.advance_desk(...)` — **the desk**: a keyed, resident conversation whose state is this
    package's data. See **Desks** below.
  - `ai.advance_in_table(...)` — **the desk table**: a keyed collection of desks plus the
    reused-name generation rule, for a program that spawns conversations by name at runtime (a
    table of workers). See **Keyed desk tables** below.
  - `use ai.with_breaker(on_transition = ...)` — one failure breaker for every model call in the
    program. See **The breaker** below.
  - `use ai.serve_session(...)` / `use ai.serve_observations(...)` — a single conversation served
    behind one request (a chat session; a resident answering `ai.observation`), with token metering
    off the seam's `usage` and automatic compaction past a threshold.
  - `ai.infer_with_tools(history, tools, max_steps)` — the one-shot loop: run until the model stops
    calling tools and return the reply.
  - `ai.infer_structured[T](history)` — structured output with the type as the contract
    (`reflection.schema_of[T]` reifies the schema, the provider decodes against it natively,
    `json.validate[T]` guarantees the returned value IS a T).
  - `ai.compact(history, policy)` on its own, for a hand-rolled loop.
  - `ai.view_file` — a tool that reads an attachment (an image, a PDF, a text document) — and the
    `file_kind` classification every provider dispatches over (see **Attached files**).
  - `ai.supervise(session, on_restart)` — run a resident forever, restarting it with backoff.
- `ai.types` — the provider-agnostic vocabulary: `turn`, `role`, `tool_call`, `call_ref`,
  `tool_result`, `step`, and the step's token accounting (`usage`, carried by `step_result` — what
  `ai.infer_step` returns). Apps and providers speak only this. Every element is a CLOSED Katari
  type, never a wire token: a `role` is `types.user_role()` / `types.model_role()`, not a string, so
  no provider gets an identity mapping and none of them can widen the vocabulary by accident.
- `ai.wire_names` — the PROVIDER LAYER's tool-name reshaping, for the providers whose APIs validate
  tool names against an alphabet that forbids the qualified name's dot. It is not in `ai` because it
  is not a fact about tool calling; a provider whose wire admits the qualified name never imports it.
- `ai.gemini` — a provider over Google's `generateContent` API. `use gemini.provider(model = ...,
  source = credentials.env(key = "GEMINI_API_KEY"))` serves `ai.infer_step` for the continuation.
  **Files: images, PDFs, audio, video and text** — the binary kinds all ride the one `inlineData`
  part keyed by their mime type, restricted to the formats Gemini documents; text inlines as text
  (freshest turn only, to keep the request small).
- `ai.openai` — a provider over the OpenAI Chat Completions API. `use openai.provider(model = ...,
  source = ...)` — swap it for `gemini.provider` (or vice versa) with no other change. **Files: text
  only** — a message's content here is a plain string, which text needs no wire shape to ride; an
  image or a PDF would need a `data:` URI (base64 inside a string), which the `http.json` slot
  contract cannot build, so those report to the model as unreadable rather than being faked.
- `ai.anthropic` — a provider over the Anthropic Messages API. `use anthropic.provider(source =
  ...)` — `model` defaults to `claude-sonnet-5` and `max_tokens` (the per-step output cap the API
  requires) to 4096; swap it for either other provider with no other change. **Files: images, PDFs
  and text** — an `image` content block (jpeg / png / gif / webp), a `document` content block
  (base64 `application/pdf`), and inlined text respectively; audio, video and any other image format
  have no Messages API block, so they report as unreadable.

Pure Katari — no FFI sidecar. The only network call is the prelude's HTTP client; every request body
is built and every response parsed as `json` values in Katari, and an inlined `file` leaves base64 at
the send boundary rather than on the value plane.

## Desks

A resident program with ONE conversation is `serve_observations`. A program with SEVERAL — a private
assistant, a public-facing character, a table of workers spawned by name — needs a **desk**: a
conversation, its freshest context measurement, and the arrivals it is holding because the model could
not be called. Nothing about a desk is keyed, ordered or routed. The app holds one desk per
conversation however it likes (a handler var, a `record[desk]` keyed by name) and turns each arriving
message into one call:

```katari
let advanced = ai.advance_desk(
  state = core,                                     // the desk as it stands
  arrival = ai.arrival(source = source, content = content),
  tools = core_tools,                               // per turn, as data
  persona = core_persona,                           // re-derived at every model step
  deliver_to = to_operator,
  policy = ai.default_desk_policy(),
)
// advanced.state is the new desk; advanced.outcome is `answered` / `filed` / `step_failed`;
// advanced.tool_events says which tools ran and how each ended.
```

Four behaviours are built in, each of them paid for by an outage somewhere:

1. **A bounded backlog.** While the model cannot be called, arrivals are held — the newest
   `backlog_retention` of them — and the stored conversation DOES NOT GROW. An outage of any length
   costs the conversation nothing, which is what stops the recovery turn from being the one that
   fails on a context the outage itself piled up.
2. **A dropped marker.** What the bound discarded is SAID, in one derived line, when the backlog
   rejoins the conversation — never appended, so it cannot accumulate.
3. **The shed.** A call the provider REFUSED loses the attachments it carried, each loss noted in
   place. A file's recorded content type is the sender's unverified claim, the provider checks the
   bytes, and the mismatch is a refusal on the whole request — which then repeats on every later
   call, because the refused message is re-sent whole.
4. **The undelivered reply, said out loud.** Under `announce_discarded_reply` a turn that ended with
   text it did not deliver gains one line saying so, so a model that answered in its closing words
   finds that out on its next turn instead of believing it spoke.

`ai.desk_policy` is one datum holding every knob (step budget, per-call deadline, exemptions,
compaction, retention, what a reply is for), so a program states its policy once — or a different one
per desk; it is data.

## Keyed desk tables

One desk is a var; a fixed handful of them are a var apiece, routed by hand — two conversations are two
names, not a collection. A program that spawns conversations BY NAME AT RUNTIME — a table of workers
admitted and dismissed while it runs — has a genuine collection, and `ai.desk_table` is that collection
plus the one rule such a program always needs: a **generation** that keeps a reused name honest.

**Why the generation.** A worker is a name, and a name is reusable. Dismiss `scribe`, admit `scribe`
again for a different errand, and the second scribe must inherit neither the first one's conversation
nor its still-in-flight mail. The name alone cannot tell the two apart; a monotonic generation stamped
at each admit does. The roster mints a fresh higher number every admit, each arrival carries the
generation it was dispatched under, and the table remembers the generation the conversation it holds
belongs to:

- **older** than the table holds → a straggler from a superseded incarnation: dropped (`stale`), the
  table unchanged, no model call spent;
- **equal** → the ordinary case, one more message in the same incarnation's conversation;
- **newer** → the name was reused and the held conversation is superseded: it is discarded and a fresh
  desk takes the turn.

**The generation is the app's, not the table's.** It is minted where a name is admitted — a site that
also validates the name, refuses the reserved ones, holds the brief and lists the roster — and passed
in to `ai.advance_in_table`; the table only stores it and compares. That split is the same one
`with_breaker` made, and it is why the table offers no `bump`: a second counter beside the roster's
would be a second source of truth for one fact. The per-arrival move is `ai.advance_desk`'s, whole and
unchanged — the table adds only the lookup, the fresh-desk default and the staleness verdict.

```katari
import ai
import ai.types

@"The current launch generation for a worker name — `null` if it has been dismissed. Served by the app's
roster, the SoT that mints a fresh higher number at every admit." request roster_generation(name: string) -> integer | null
@"One worker's inbound message, dispatched by the mail bridge." request worker_message(to: string, source: string, content: string) -> null
@"Where a worker's undeliverable mail bounces." request bounce_to_core(content: string) -> null

@"A worker has no channel, so its reply text goes nowhere." agent discard(reply: string) -> null { null }

// `infer_step` is served by the ambient provider — a `use <provider>.provider(...)` installed above
// this agent at the app root, exactly as in the one-shot and resident examples further up.
agent serve_worker_table(value: null) -> null with io | ai.infer_step | roster_generation | bounce_to_core | prelude.throw[ai.duplicate_tool] {
  use handler (var table: ai.desk_table = ai.new_desk_table()) {
    request worker_message(to: string, source: string, content: string) {
      match (roster_generation(name = to)) {
        // Dismissed (or never admitted): bounce, and forget any desk still held under the name.
        case null -> {
          bounce_to_core(content = f"[no worker named ${to}] ${content}")
          next null with { table = ai.forget_desk(table = table, key = to) }
        }
        // Live: advance the key's desk under the generation the roster minted for this incarnation.
        case generation -> {
          let advanced = ai.advance_in_table(
            table = table,
            key = to,
            generation = generation,
            arrival = ai.arrival(source = source, content = content),
            tools = [ai.view_file],
            persona = ai.no_persona,
            deliver_to = discard,
          )
          // advanced.table is written back whether the arrival was served or dropped as `stale`;
          // advanced.tool_events reports the turn's tool calls (empty on a stale drop).
          next null with { table = advanced.table }
        }
      }
    }
  }
  worker_message(to = "scribe", source = "mail:core", content = "start")
}
```

`ai.advance_in_table` returns a `table_turn`: `.table` to write back, `.outcome` (`advanced(desk_turn)`
when a desk took the turn, or `stale` when a straggler was dropped), and `.tool_events`. `ai.forget_desk`
drops a key (the retire move) and `ai.desk_generation` reads the generation a key is on. **This is for
dynamic keys only** — a program's fixed desks (a core, a herald) stay their own vars; wrapping a
one-and-two set in a table would buy nothing and cost a generation nobody reuses.

## The breaker

Absorbing a failed step is right when the provider is having a bad minute and wrong when it is out for
the evening. `ai.with_breaker` is middleware over the seam: install it INSIDE the provider and every
desk, session and hand-rolled `take_turn` in the program passes through the one breaker by dynamic
scope.

```katari
use gemini.provider(model = "gemini-3.5-flash", source = credentials.env(key = "GEMINI_API_KEY"))
use ai.with_breaker(on_transition = announce)
```

Three consecutive failed steps open it (`breaker_policy` is data); while open, `infer_step` answers
`inference_failed(...)` without calling out, except for one probe step per interval — no wait before
the first, then 30s doubling to ten minutes. `on_transition` fires on the STATE CHANGE and nowhere
else, so the notice is structurally un-floodable: exactly two lines per outage, one out and one back.
WHERE they go and HOW they read is the app's — the sink gets an `outage_kind`, and `ai.outage_line` /
`ai.recovery_line` are the wordings this package suggests. A remedy ("check the API credit balance")
is information for the one person who can act on it, so an app posting to a channel strangers read
hands `outage_line` a `waits_out()` whatever the kind actually was and routes the remedy elsewhere.

A desk needs to know nothing about any of this: it sees its turn come back refused, files the arrival
into its backlog, and spends nothing.

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
at the upload boundary that recorded the type. What IS normalized is the label itself, never the bytes: RFC
2045 makes a media type case-insensitive and lets a sender append parameters, so `Text/Plain; charset=utf-8`
reads as `text/plain` for both the dispatch and the provider's `media_type` field — and a confirmed ALIAS
resolves to the registered type for the same format (`image/jpg` → `image/jpeg`, `video/mov` →
`video/quicktime`), so the commonest real-world mislabelings inline instead of taking the note. Only labels
confirmed to name the identical format are in that table; a guess there would turn a recoverable note into
a 400.

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
identical, so a repeated request prefix stays byte-identical and every API's prefix reuse still applies;
only a genuinely new attachment moves it.

**The textual arm puts a file's content on the value plane, and privacy travels with it.** An inlined text
file is read, decoded and placed in message text, so a `file of private` — a Gmail attachment fetched with
an OAuth token, anything downloaded through a private request — makes that message text private too, where
the same file previously contributed only an opaque handle. Toward a provider this changes nothing (a
model API is a private-capable boundary and always was). It matters where an app forwards tool results or
history text onward: a channel post, a webhook reply, a run result read by a user. Those are public
boundaries, so a private-tainted string is refused there — the same rule that already governed the file
itself, now reaching the text it decodes to. An app that echoes raw tool results to a channel should echo
its own summary instead, or keep the private file's content out of what it forwards.

## Failures

**The seam is outcome-typed; providers never throw.** `ai.infer_step` / `ai.infer_object` return an
OUTCOME sum — `inferred(result)` on success, `inference_failed(error)` on failure — rather than throwing a
`step_error`. Handing the failure back as a VALUE is what lets each loop decide at its own perform site: a
long-lived loop ABSORBS a failure (it abandons just that turn and awaits the next event, so a step failure
cannot end a resident), while the one-shot loops (`reply`, `infer_with_tools`, `infer_structured`,
`serve_session`) RE-RAISE it as `prelude.throw[step_error]` so the caller's own handler catches it. A deep
provider handler can only throw OUT of its own context, never INTO the loop's — the resume value is the
only channel that reaches the loop.

**`step_error` has three transport arms and one open one.** A non-2xx status (`http.status_error`), a
transport failure (`http.fetch_error`) and a malformed reply (`json.parse_error`) are what an app
dispatches on to tell "check your credit balance" from "wait it out". The fourth arm,
`ai.provider_error(source, message, transient, detail)`, is what keeps those three from being a
specification: a provider carries a failure only it can name — a reply that finished with no answer, a
refusal reported out of band, a local model's own error — as its own value in `detail`, states whether a
retry could help in `transient`, and needs no arm of its own here. `ai.gemini.empty_reply` is exactly
that: it lives in the provider that reports it, not in the neutral layer.

**Transient retry.** Before surfacing a failure, each provider first retries a transient one — a 429 rate
limit, a 5xx, a dropped connection — with exponential backoff. `retry_attempts` (default 6) and
`retry_base_ms` (default 2000, doubled per retry and capped at one minute) tune it. A fatal error (a
non-retryable 4xx, malformed JSON) or an exhausted budget is what becomes the `inference_failed` outcome.
That ladder is how hard ONE call tries; `with_breaker` is whether to start a call at all, and the two are
deliberately separate.

**Tool failures are observable.** A failing tool never fails the conversation — a panic, a typed throw,
malformed arguments, a deadline, a hallucinated name all fold back to the model as a result it reads and
corrects from. That is right for the model and blind for the app, so a turn also carries
`tool_events: array[tool_event]` out: one per call, with `tool_succeeded` / `tool_failed` /
`tool_timed_out` / `tool_not_found`. Nothing in the loop branches on them; what to do with them is yours.

## Secrets / env

A model API key, provided to whichever provider you `use`, as a `credentials.source`:

- Gemini: `use gemini.provider(source = credentials.env(key = "GEMINI_API_KEY"), model = ...)`
- OpenAI: `use openai.provider(source = credentials.env(key = "OPENAI_API_KEY"), model = ...)`
- Anthropic: `use anthropic.provider(source = credentials.env(key = "ANTHROPIC_API_KEY"))`

Store it in the runtime, never in a file: `katari env set GEMINI_API_KEY --secret`. The key is a
`string of private`; it can flow into the provider's auth header but never out to a user boundary. A
`credentials.oauth(name = "...")` source works the same way, resolved by the runtime at each request.

## Usage

One shot, with tools:

```katari
import ai
import ai.types
import ai.gemini

agent solve(task: string) -> string with io | prelude.throw[ai.duplicate_tool | ai.step_error | env.missing_secret | oauth.server_error] {
  use gemini.provider(
    model = "gemini-3.5-flash",
    source = credentials.env(key = "GEMINI_API_KEY"),
    system = "You are a helpful assistant.",
  )
  ai.infer_with_tools(
    history = [types.turn(role = types.user_role(), text = task, files = [])],
    tools = [ai.view_file],    // add your own tools here
    max_steps = 12,
  )
}
```

A resident with two desks under one breaker:

```katari
import ai
import ai.anthropic

@"Where a desk's replies go." request post(channel: string, text: string) -> null
@"Mail addressed to the assistant." request assistant_message(source: string, content: string) -> null

agent to_operator(reply: string) -> null with post { post(channel = "control", text = reply) }
agent voice(value: null) -> string { "You are the operator's assistant." }

@"The two lines per outage — the remedy goes where the person who can act on it reads."
agent announce(event: ai.breaker_event) -> null with post {
  match (event) {
    case ai.breaker_opened(kind => kind) -> to_operator(reply = ai.outage_line(kind = kind))
    case ai.breaker_closed(kind => _) -> to_operator(reply = ai.recovery_line())
  }
}

agent resident(value: null) -> string with io | post | prelude.throw[ai.duplicate_tool | env.missing_secret | oauth.server_error] {
  use anthropic.provider(source = credentials.env(key = "ANTHROPIC_API_KEY"))
  use ai.with_breaker(on_transition = announce)
  use handler (var assistant: ai.desk = ai.new_desk()) {
    request assistant_message(source: string, content: string) {
      let advanced = ai.advance_desk(
        state = assistant,
        arrival = ai.arrival(source = source, content = content),
        tools = [ai.view_file],
        persona = voice,
        deliver_to = to_operator,
      )
      next null with { assistant = advanced.state }
    }
  }
  assistant_message(source = "boot", content = "Say hello.")
  "running"
}
```

Imported modules are referenced by their last path segment: `ai.*`, `types.*`, `gemini.provider`.
