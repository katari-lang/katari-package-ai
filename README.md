# ai — provider-agnostic AI for Katari

The tool-calling loop, factored so the model provider is one `use` line: the whole abstraction is a single
request, `ai.infer_step`. There is one turn loop (`ai.take_turn`), one serving mechanism
(`ai.serve_observations`), and one surface for a program with several AIs — the route.

```katari
import ai
import ai.anthropic

@"Where this program says things out loud." request notify(line: string) -> null

@"Tool: send a message to another agent by name."
agent tell(to: string, content: string) -> string with ai.inbound_provenance | ai.mail {
  let whence = ai.inbound_provenance()
  match (ai.mail(to = to, source = f"mail:${whence.origin}", content = content, hop = whence.hop + 1)) {
    case ai.posted(_) -> "sent"
    case ai.no_recipient(to => missing) -> f"no such agent: ${missing}"
  }
}

@"This AI's first breath, as a source: sources are forked at the handshake, so this arrives."
agent greet(self: string) -> null with ai.mail {
  let _posted = ai.mail(to = self, source = "boot", content = "You are online.", hop = 0)
  null
}

@"Where a turn's words go." agent say(reply: string) -> null with notify { notify(line = reply) }

agent main() -> string with io | notify | store.get | store.set | store.delete | store.list |
  prelude.throw[env.missing_secret | oauth.server_error] {
  use anthropic.provider(source = credentials.env(key = "ANTHROPIC_API_KEY"))
  // Above the route, because a fiber runs under the watch: this is where the AI's ending is answered.
  use handler {
    request region.crashed(id: string, name: string, message: string) { break f"stopped: ${name} (${message})" }
    request region.failed(id: string, name: string, error: unknown) { break f"stopped: ${name}" }
  }
  let root = use ai.route[io | notify]()
  let _core = ai.spawn[io | notify](name = "core", tools = [tell], max_steps = 16,
                                    deliver_to = say, sources = [greet], workspace = "core")
  region.watch(nursery = root)
}
```

## How the route composes

**The route owns the composition**: the nursery the AIs live in, the table of who is addressable, a
delivery fiber per `mail`, a cancellation per `dismiss`, and the eviction of an AI the moment it dies.
Under it runs one `ai.resident` fiber per AI — its own conversation, its own mailbox, optionally its own
store workspace.

**The watch is where the AIs start.** An AI becomes addressable when its `register` handshake is served,
and a fiber's escalations are held durably until a `watch` is up, so the first report to an AI comes from
something running after `region.watch(nursery = root)`: a `sources` entry, a tool, or one of your own
handlers. A `mail` performed before that answers `no_recipient`.

**Your own handlers are installed above the route.** A handler serves what is performed inside its extent,
and the route re-performs a death outward after forgetting the AI — so the provider, the breaker, whatever
serves your tools' effects and your `region.crashed` / `region.failed` handlers go above `use ai.route`.

**A restart goes at the altitude of what died.** A `crashed` names a dead fiber whose region is alive, so a
re-hiring handler stands inside the route's extent (below the `use` line, above the watch), where a `spawn`
reaches the route. Above the route is outside the region, and that altitude's business is the region's own
death: the notice, the stop, the `supervise` replay around the session.

**Sources belong to an AI.** Each `sources` entry is forked at the register handshake with `self` = the
AI's name, under the fiber name `<ai>/source/<n>` in the order listed, so a `crashed` names which watcher
died. One entry is one fiber and one death; subscriptions that must share a fate go inside one entry with
`parallel`.

**Mail carries its hop.** `hop` is how many agent-to-agent relays a report has already made: 0 for a person
or a clock, `ai.inbound_provenance().hop + 1` in a tool that relays. It has no default, being a fact about
the message rather than about the sender, and a program that damps replies reads it.

**Nothing serializes across AIs.** Every middleware at the seam — a provider's install, `ai.with_breaker`,
`ai.with_context` — is a parallel handler. What is ordered is one AI's own turns (a conversation is a
sequence) and the route's clauses, which share one table.

**Spell the effect argument.** `ai.route[E]()` and `ai.spawn[E](...)` take the row your own agents run under
as an explicit type argument, and those agents are named top-level agents over plain data, since everything
in a `spawn` record travels into a fiber the caller does not own. They take your row plus
`prelude.throw[unknown]`, `region.fork`'s own contract: an uncaught throw ends the fiber as `region.failed`.

## Modules

- `ai` — the loop and everything over it: `ai.take_turn` (one model turn as a value),
  `use ai.serve_observations(...)`, `use ai.route[E]()` with `ai.spawn` / `ai.mail` / `ai.dismiss`,
  `ai.resident` (one AI's body as a task to fork), `ai.deliver` (the delivery fiber),
  `use ai.with_breaker(...)`, `use ai.with_context(...)`, `ai.compact`, `ai.view_file`, and the one-shot
  calls `ai.infer` / `ai.infer_with_tools` / `ai.infer_structured[T]`.
- `ai.types` — the provider-agnostic vocabulary (`turn`, `role`, `tool_call`, `call_ref`, `tool_result`,
  `step`, `usage`); every element is a closed Katari type and never a wire token.
- `ai.wire_names` — tool-name reshaping for the providers whose name alphabet forbids a qualified name's dot.
- `ai.anthropic` / `ai.gemini` / `ai.openai` — one provider each, swappable by changing the one `use` line
  and sharing the same knobs: `model` (each defaults to its series' general-purpose tier), `source`,
  `system`, `max_output_tokens`, `retry_attempts`, `retry_base_milliseconds`. Anthropic's Messages API
  requires an output cap, so `null` there sends its stand-in (4096); Gemini adds `thinking_budget`.

Pure Katari, no FFI sidecar: request bodies are built and replies parsed as `json` values, and an inlined
`file` base64s at the send boundary. The API key reaches a provider as a `credentials.source` —
`credentials.env(key = "ANTHROPIC_API_KEY")` or `credentials.oauth(name = "...")` — resolved at each
request, so a rotation lands mid-run; stored with `katari env set ANTHROPIC_API_KEY --secret` it is a
`string of private` that flows into the auth header and never out to a user boundary.

## The resident

A resident agent is a conversation nothing is waiting on. Reports arrive from background fibers as
`ai.observation`s; the serving handler injects each as a user turn, advances the conversation with
`take_turn`, and routes a non-empty reply to `deliver_to`. An empty reply means "nothing to say" — it is
not delivered and nothing asks again, so a quiet watcher tick costs one model step. Four behaviours come
with it: a **bounded backlog** (the newest 24 arrivals are held while the model cannot be called, and the
stored conversation does not grow, so an outage of any length costs it nothing); a **dropped marker**, one
line derived at each attempt from what the bound discarded; **the shed**, which drops the attachments of a
call the provider refused and notes each loss in place, since a refused message is re-sent whole on every
later call; and **one notice per outage**, on the transition rather than per turn.

`deliver_to` is optional, and `null` says something: this AI speaks only through its tools. A turn that
then ends with undelivered text gains one line in its own conversation saying so; a blank final gains
nothing, silence being a legitimate move. For the extent of a dispatch the serving handler answers
`ai.inbound_provenance() -> {origin, hop}`, so a tool reads the sender and the hop as values rather than
parsing the injected turn's label.

## Attached files

How a file reaches the model follows from one datum — its recorded content type. `ai.classify_file` turns
it into the closed `ai.file_kind` sum, and each provider matches it with one arm per shape its API accepts:

| content type | Anthropic | Gemini | OpenAI |
| --- | --- | --- | --- |
| `image/*` | `image` block — jpeg, png, gif, webp | `inlineData` part — png, jpeg, webp, heic, heif | unreadable note |
| `application/pdf` | `document` block | `inlineData` part | unreadable note |
| `text/*`, `application/json` / `xml` / `csv` | inlined text | inlined text | inlined text |
| `audio/*`, `video/*` | unreadable note | `inlineData` part — the documented formats | unreadable note |
| anything else | unreadable note | unreadable note | unreadable note |

Each provider inlines only the media types its own API documents; a type inside the family but outside that
list takes the unreadable note, which costs the model one honest sentence where a hopeful send costs the
whole step. Inlined text is capped at 24000 code points with an explicit truncation marker, an unreadable
note states the content type and that the content is absent, and the type is taken as recorded — only the
label is normalized (case and parameters dropped, `image/jpg` → `image/jpeg`).

The table holds for a file attached to a turn and for one a tool handed back alike. Only the newest
file-bearing element inlines its content, since every step re-sends the whole conversation; `ai.view_file`
re-reads an older file on demand, and the window is keyed to files rather than to position, so appending a
tool call or a result leaves every earlier byte identical. Inlining a textual file puts its content on the
value plane, so a `file of private` makes that message text private too: a public boundary refuses the
tainted string, and an app that echoes tool results forwards a summary.

## Failures

**The seam is outcome-typed.** `ai.infer_step` / `ai.infer_object` answer `inferred(result)` or
`inference_failed(error)` rather than throwing, so each loop decides at its own perform site: a resident
absorbs a failure and awaits the next event, the one-shot calls re-raise it. `step_error` speaks the
stdlib's vocabulary — `http.auth_error` when the credential failed, `http.api_error` otherwise (both
carrying the status), `http.fetch_error`, `json.parse_error` — so one `supervise` converter covers a model
API too, and `ai.provider_error(source, message, transient, detail)` carries what only a provider can name.

**Two layers of retry.** `ai.with_retries` is how hard one call tries: a transient error backs off
exponentially (`retry_attempts`, default 6, from `retry_base_milliseconds`, default 2000, capped at a
minute) and a fatal one becomes the `inference_failed` outcome. `ai.with_breaker` is whether to start a
call at all: three consecutive failed steps open it, one probe per interval tests for recovery (no wait
before the first, then 30s doubling to ten minutes), and `on_transition` fires on the state change and
nowhere else. Its state lives in the store at `ai/breaker`, so it survives a restart and needs no queue.

**Tool failures are observable.** A failing tool never fails the conversation — a panic, a typed throw,
malformed arguments, a deadline and a hallucinated name all fold back to the model as a result it corrects
from — and a turn carries `tool_events: array[tool_event]` out beside the reply. A served resident has no
sink for them on purpose: the model already sees every tool failure in band, so an operator hears about a
sick tool through a line of `persona` plus the AI's own means of speaking. Where a deterministic ledger is
required, read `ai.take_turn`'s `turn_result` yourself.

## The mechanism layer

The route is built out of parts that stay public: `ai.observation` (the event→model channel),
`ai.serve_observations` (the serving handler), `ai.mailbox` / `ai.mail_scope` (an AI's inbox and its scope
marker), `ai.register` (the handshake), `ai.deliver` (the delivery fiber) and `ai.resident` (one AI's whole
body). A program that needs something wrapped around one AI forks `ai.resident[E]` into the route's nursery
itself; `register` puts it in the table, so `ai.mail` reaches it like a spawned one, and only `dismiss`
tells them apart, since a fiber is cancelled through the nursery that owns it.

A program whose routing is genuinely its own writes a handler over `register` and `deliver` and opens no
route. Three properties of the model shape it. A resident's mailbox is a second nursery and a fiber runs
under its own region's `watch`, so one nursery has one watch and what covers every fiber is what wraps that
watch. A cross-nursery fork carries its task's captured environment into the target's instance, so delivery
is `ai.deliver` — a named agent over plain data, capturing nothing. Delivery is at-most-once, a fork into a
dead nursery panicking, which `route`'s `mail` clause folds into `no_recipient` plus an eviction. Every
mailbox shares one scope marker, so a sender discharges `region.provide[ai.mail_scope, ai.observation]`
once and can fork into any resident's handle; the merge is type-level only, since a fork routes by the
handle's own runtime scope identity.
