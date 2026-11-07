---
title: Bot Tools
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Valerie Pond"
    email: "v.a.pond@outlook.com"
    period: "2026"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `bot-cmds` or `bot-tools` capability names. Instead, implementations
SHOULD use the `draft/bot-cmds` and `draft/bot-tools` capability names to be
interoperable with other software implementing a compatible work-in-progress
version. The final version of the specification will use unprefixed names.

For the same reason, every message tag and batch type defined here carries the
`draft/` prefix during the work-in-progress period:

- Capabilities: `draft/bot-cmds`, `draft/bot-tools`
- Client-only tags: `+draft/bot-cmds-query`, `+draft/bot-cmds`, `+draft/bot-cmds-changed`, `+draft/bot-cmd`, `+draft/bot-cmd-error`, `+draft/bot-tools`
- Batch types: `draft/bot-cmds`, `draft/bot-tools-workflow`, `draft/bot-tools-content-stream`

The `+reply` and `+draft/channel-context` client tags used for command replies
are defined by separate specifications and are reused here unchanged.

## Introduction

A *bot* is any automated IRC client, from a simple command responder to one that
performs complex multi-step tasks. Bots interact with users through chat, but
clients have no standard way to understand a bot beyond the words it sends. This
specification defines two complementary capabilities that let a bot expose
itself to other clients in a structured, machine-readable way, built on
[message tags][] and `TAGMSG`:

- **`draft/bot-cmds`** lets a bot advertise the commands it accepts, with typed
  arguments, and lets clients discover and invoke them as structured *slash
  commands* rather than ad-hoc text.
- **`draft/bot-tools`** lets a bot stream the multi-step workflow it runs before
  replying (its reasoning, the tools it calls, and their inputs and results), so
  users can follow, audit, approve, or cancel it.

The two capabilities are independent. A bot may implement either, both, or
neither. They are specified together because they share the same transport and
encoding and are commonly implemented together. Everything defined here is
carried in `TAGMSG` and message tags, and is invisible to clients and servers
that do not implement it. None of this is specific to any particular kind of
bot; any automated client can use it.

## Motivation

Command discovery on IRC has historically relied on convention: users type
`.help`, `!help`, or `?help` and hope the bot responds. A client cannot learn
what a bot accepts before interacting with it, there is no machine-readable list
of commands, and an invocation is unstructured text. Clients therefore cannot offer
autocomplete, argument validation, inline help, or a command palette, and a bot
cannot reliably parse what was asked of it.

Separately, a bot that performs a multi-step task (reasoning about a request,
calling external tools such as web search, code execution, or file access,
processing results, and iterating) sends only its final message. Users cannot
see which tool ran, with what inputs, or whether it succeeded, nor intervene
while it runs. This makes a bot's output difficult to understand, audit, or
trust, and rules out the kind of live, interactive workflow display that
graphical tooling provides elsewhere.

This specification addresses both. It remains backwards compatible and entirely
opt-in.

## Architecture

### Dependencies

Clients and bots implementing this specification MUST negotiate the
[message tags][] and [message ids][] capabilities. [message ids][] is required
because command replies are correlated to the invocation that prompted them, and
workflows to their triggering message, by `msgid`.

Bots SHOULD also set the `+B` user mode from [bot mode][] so clients know the
sender is automated; clients SHOULD use that mode to decide whether to trust the
tags defined here (see [Security considerations](#security-considerations)). A
server implementing [bot mode][] adds the valueless [`bot`][bot tag] tag to the
messages a bot sends, which the examples below show on bot replies.

The [batch][] capability is REQUIRED to send a command list or workflow that
does not fit in a single message, and to stream large tool results; it is
otherwise OPTIONAL. The [labeled response][] capability is OPTIONAL.

A server that also implements [channel-bots][] folds bots' command
advertisements into `obby.world/bot-list` discovery batches. The two
specifications are independent but complementary.

Command replies use two existing client tags: [`reply`][] (`+reply`) to point a
reply at the invocation it answers, and [`channel-context`][]
(`+draft/channel-context`) to display a whispered reply against the channel a command
related to. This specification does not define them.

### Capabilities

Both capabilities are advertised by the server in `CAP LS` and negotiated by
clients in the usual way. A bot, being a client, negotiates them like any other
client. Neither carries a value.

#### `draft/bot-cmds`

A client negotiates `draft/bot-cmds` to take part in the command protocol
defined in [Slash commands](#slash-commands). A server that offers it relays the
command tags as it would any [client-only tags][] and needs no behaviour
specific to this specification.

#### `draft/bot-tools`

A client negotiates `draft/bot-tools` to receive bots' workflow tags. A server
that offers it relays `+draft/bot-tools` only to clients that have negotiated
it, so the workflow stream does not reach clients that would not display it. A
bot sends those tags using [message tags][] regardless of this capability; the
capability governs who receives them, not who may send.

The optional behaviours a particular bot supports are not part of this
capability, because a server cannot speak for an individual bot. A bot
advertises its own behaviours in its [`workflow` messages](#message-schema).

### Value encoding

Every tag defined by this specification that carries structured data
(`+draft/bot-cmds`, `+draft/bot-cmd`, and `+draft/bot-tools`) does so as a JSON
object serialised to compact JSON (no insignificant whitespace) and then
encoded with standard [base64][] (RFC 4648 §4, with `=` padding). A receiver
base64-decodes the value and then parses it as JSON.

base64 is used rather than raw JSON because its alphabet (`A-Z`, `a-z`, `0-9`,
`+`, `/`, `=`) contains no character that [message tags][] value escaping
touches, so a value is placed in and read from a tag with no escaping step and
no risk of an escaping bug corrupting the payload.

A single message MUST NOT carry more than 4094 bytes of tag data (the client tag
limit). Because base64 expands its input by about one third, the compact JSON
behind a single tag MUST be kept under roughly 3000 bytes; a sender MUST verify
the encoded length before sending. When the data does not fit:

- A `+draft/bot-tools` message MUST truncate its `content` field, set
  `"truncated": true` on the same object, and re-verify before sending (or use
  the [content-stream batch](#batch-types) for a large string `content`).
- A command list (`+draft/bot-cmds`) that does not fit MUST be split across a
  [batch][] (see [Discovering commands](#discovering-commands)).

In the examples throughout this document, the decoded JSON of a base64 value is
shown in a preceding `;` comment, and the wire value itself is written as
`<base64>` for readability.

### Slash commands

This section defines the `draft/bot-cmds` capability: how a bot's commands are
discovered, and how they are invoked. The five tags it introduces
(`+draft/bot-cmds-query`, `+draft/bot-cmds`, `+draft/bot-cmds-changed`,
`+draft/bot-cmd`, and `+draft/bot-cmd-error`) are [client-only tags][] (they
begin with `+`, so servers relay them verbatim). Replies additionally use the
existing [`reply`][] and [`channel-context`][] client tags.

A *command* has a name, a human-readable description, typed arguments
("options"), and the contexts (public in a channel, private in a channel, or in
a private message) in which it may be used. Commands are invoked structurally
via the `+draft/bot-cmd` tag, not by matching a text prefix.

#### Command schema

A bot's command list is a JSON object. Its `commands` field is an array of
command objects, which MAY be empty. It MAY also carry a `prefix`: the historical
text prefix the bot responds to (for example `!`), used only by servers performing
[legacy command compatibility](#legacy-command-compatibility) translation.

Each command object:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Command name, without any prefix. Unique among a bot's commands; matched case-insensitively. |
| `description` | string | yes | Short human-readable description. SHOULD be 100 characters or fewer. |
| `contexts` | array of string | yes | Where and how the command may be invoked; any of the values below. MUST be non-empty. |
| `options` | array of object | no | The command's typed arguments, in order. |
| `requires` | object | no | Conditions the invoker must meet; see [Command gating](#command-gating). |

A command's `contexts` is any combination of:

| Context | Meaning |
|---|---|
| `public` | Invoked in a channel, with the call and reply visible to the channel. |
| `private` | Invoked in a channel but kept between the caller and the bot, the reply whispered back in the channel's context. |
| `pm` | Invoked in a private message to the bot, with no channel. |

Each option object:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Option name. |
| `type` | string | yes | One of `string`, `int`, `bool`, `user`, `channel`. |
| `required` | bool | no | Whether the option must be supplied. Defaults to `false`. |
| `description` | string | no | Human-readable description of the option. |
| `choices` | array | no | If present, the value MUST be one of these. |

The value supplied for an option when the command is invoked MUST match its
declared `type`:

| Type | Value |
|---|---|
| `string` | Arbitrary UTF-8 text, as a JSON string. |
| `int` | An integer, as a JSON number. |
| `bool` | A JSON boolean. |
| `user` | A nickname identifying a user, as a JSON string. |
| `channel` | A channel name including its prefix (e.g. `#chan`), as a JSON string. |

```json
{
  "commands": [
    {
      "name": "weather",
      "description": "Look up the weather for a city.",
      "contexts": ["public", "pm"],
      "options": [
        {"name": "city", "type": "string", "required": true,
         "description": "Which city?", "choices": ["london", "berlin", "tokyo"]}
      ],
      "requires": {"min-channel-rank": "voice"}
    }
  ]
}
```

#### Command gating

A command's optional `requires` object lists conditions the invoker MUST satisfy.
All conditions present MUST hold.

| Key | Type | Meaning |
|---|---|---|
| `min-channel-rank` | string | Minimum rank in the channel the command was invoked from: `voice`, `halfop`, `op`, `admin`, or `owner`. |
| `account` | bool | If `true`, the invoker MUST be logged in to an account. |
| `tls` | bool | If `true`, the invoker MUST be connected over TLS. |

The bot evaluates `requires` against the invoker (see
[Invoking a command](#invoking-a-command)).

`min-channel-rank` applies only to the `public` and `private` contexts, which
have an associated channel. It has no meaning in `pm`, where a bot MUST ignore
it; a command gated on `min-channel-rank` SHOULD NOT list `pm` in its
`contexts`.

#### Discovering commands

A client learns a bot's commands in one of three ways. In every case the bot
sends a `+draft/bot-cmds` tag whose base64 value decodes to the
[command schema](#command-schema) object directly to the client. If the encoded
list exceeds the tag limit it MUST be sent inside a [batch][] of type
`draft/bot-cmds`: the value is split across the batch's messages and the client
concatenates the base64 fragments, in order, before decoding.

**Direct query.** A client sends a query to a bot's nickname; the bot replies
privately:

    C: @+draft/bot-cmds-query TAGMSG <botnick>
    S: @+draft/bot-cmds=<base64> TAGMSG <querying-nick>

**Channel query (channel bots).** On joining a channel, a client MAY send a
single query to the channel; every bot in the channel that implements
`draft/bot-cmds` SHOULD reply privately to the querying client:

    C: @+draft/bot-cmds-query TAGMSG <channel>
    S: @+draft/bot-cmds=<base64> TAGMSG <querying-nick>     ; from each bot

**Connect-time advertisement (network-wide bots).** A network MAY operate bots
that are reachable from any channel rather than joined to one, such as services
or operator-run bots. Such a bot MAY proactively send a user its command list as
a `+draft/bot-cmds` `TAGMSG`, so the client learns it without querying. The bot
sends this once the user has completed registration, that is after that user has
received the `RPL_WELCOME` (`001`) numeric, not during capability negotiation: by
that point the client has finished negotiating its capabilities and is online and
ready to receive it.

    S: @+draft/bot-cmds=<base64> :<botnick> TAGMSG <connecting-nick>

This is bot behaviour, not server behaviour: the bot originates the `TAGMSG` and
the server relays it as it would any other client-only tag. A client that has
not negotiated [message tags][] does not receive it, and one that has not
negotiated `draft/bot-cmds` ignores it. Reaching pre-existing or non-negotiating
clients reliably is left to a future backwards-compatibility layer.

A client SHOULD cache a bot's command list, keyed by network and bot nickname. A
bot whose commands change SHOULD send a `+draft/bot-cmds-changed` `TAGMSG` to
each affected channel; a client receiving it SHOULD re-query that bot. A client
SHOULD also expire its cache after a reasonable time (24 hours is typical) and
on an explicit user refresh.

    S: @+draft/bot-cmds-changed TAGMSG <channel>

#### Invoking a command

A command is invoked with the `+draft/bot-cmd` tag, whose base64 value decodes
to an object with the command `name`, an `options` object of supplied arguments,
an optional `bot` naming the target bot (for disambiguation, see below), and, for
a command invoked in the `private` context, the `channel` it relates to:

```json
{"name": "weather", "options": {"city": "london"}}
```

How the invocation is addressed depends on the context being used:

- In the `public` context the invocation is a `TAGMSG` to the channel, so it is
  visible there. The channel is the message target, so the object omits
  `channel`:

      C: @+draft/bot-cmd=<base64> TAGMSG <channel>

- In the `private` context the invocation is a `TAGMSG` to the bot's nickname,
  naming the channel it relates to in `channel` (for example
  `{"name":"balance","options":{},"channel":"#casino"}`):

      C: @+draft/bot-cmd=<base64> TAGMSG <botnick>

- In the `pm` context the invocation is the same `TAGMSG` to the bot's nickname,
  with no `channel`.

The channel is carried in the payload rather than the [`channel-context`][] tag
because that tag is only valid on `PRIVMSG` and `NOTICE`, not `TAGMSG`.

A `public` invocation to a channel is seen by every bot in it; a bot acts only
on a command `name` it advertises. Command names are unique per bot, and SHOULD
be unique among the bots in a channel. When more than one bot in a channel
advertises the same command name, a client disambiguates by setting the `bot`
field of the invocation to the intended bot's nickname; a bot MUST ignore a
`public` invocation whose `bot` names a different nickname. The `bot` field is
meaningful only in the `public` context, since a `private` or `pm` invocation
already names the bot as the message target. A client MAY alternatively address
one specific bot by sending the invocation privately to its nickname, naming the
`channel` in the payload to preserve the channel context.

On receiving an invocation a bot MUST validate it against the advertised schema
for that command. If the command is unknown, a required option is missing, or a
value has the wrong `type` or is outside its `choices`, the bot MUST reject the
invocation as described in [Reporting an error](#reporting-an-error). A bot
SHOULD NOT act on an invocation of a command in a context it did not advertise,
and SHOULD reject such an invocation with `BAD_CONTEXT`.

The bot MUST also evaluate the command's `requires` against the invoker. If the
invoker does not qualify the bot SHOULD silently ignore the invocation; it MAY
instead reject it with `NOT_PERMITTED`.

Validation is the bot's responsibility. A server does not validate invocations
or reject them; it relays `+draft/bot-cmd` as an ordinary client-only tag.

#### Reporting an error

A bot reports an invalid invocation with the `+draft/bot-cmd-error` client tag on
a `NOTICE` to the invoker, correlated with the invocation using the [`reply`][]
tag (`+reply`). The tag's value is one of the codes below; the `NOTICE` text is a
human-readable explanation. A `private` invocation's error is whispered to the
invoker and carries the [`channel-context`][] tag, like a private reply; a `pm`
invocation's error is a plain whispered `NOTICE`; a `public` invocation's error
MAY be sent to the channel or whispered to the invoker.

| Code | Meaning |
|---|---|
| `INVALID_COMMAND` | No command by that name is advertised. |
| `INVALID_OPTIONS` | A required option is missing, or a value has the wrong `type` or is outside its `choices`. |
| `BAD_CONTEXT` | The command was invoked in a context it does not advertise. |
| `NOT_PERMITTED` | The invoker does not satisfy the command's `requires`. A bot MAY instead ignore the invocation silently. |

The value is an uppercase token; a client that does not recognise it SHOULD fall
back to displaying the `NOTICE` text. Unlike the structured tags, this value is a
plain token and is not base64-encoded.

    S: @+reply=m42;+draft/bot-cmd-error=INVALID_OPTIONS :WeatherBot!bot@example.com NOTICE Alice :Missing required option: city

#### Replying

A bot replies to an invocation as an ordinary message, correlating it with the
invocation using the [`reply`][] client tag (`+reply`) set to the `msgid` of the
invocation `TAGMSG`.

- A `public` invocation is answered with a `PRIVMSG` to the channel:

      S: @+reply=<invocation-msgid> PRIVMSG <channel> :London: 14°C, light rain

- A `private` invocation is answered with a `NOTICE` to the caller, carrying the
  [`channel-context`][] tag so the client can display the whisper against the
  channel the command related to:

      S: @+reply=<invocation-msgid>;+draft/channel-context=<channel> NOTICE <user> :London: 14°C, light rain

- A `pm` invocation is answered with a `NOTICE` to the caller, with no
  `+draft/channel-context`.

### Legacy command compatibility

`+draft/bot-cmd` invocations and `+draft/bot-cmds` advertisements travel in
client-only tags, so a client that has not negotiated `draft/bot-cmds` sees
nothing of them: a `TAGMSG` carrying only client tags has no displayable body.
Conversely, a user on such a client invokes a bot the historical way, by typing a
prefixed line such as `!weather london`, which a structured bot does not parse. A
server that knows a bot's commands can translate between the two, so that capable
and legacy participants share a channel without either losing function.

No new capability is required for this. A *capable* client or bot is one that has
negotiated `draft/bot-cmds`, which is signal enough for the server; a bot opts in
by advertising a `prefix` in its [command list](#command-schema). From the bot's
advertisements the server caches that `prefix` and the command schemas; a bot
SHOULD advertise on connecting so the server can translate from the start.
Translation is optional server behaviour, and the only behaviour this
specification asks of a server: a bot with no advertised `prefix`, or one a server
chooses not to translate for, is simply not bridged.

Translation applies to the `public` context only. The `private` and `pm` contexts
are addressed to the bot and seen by no one else, so nothing is lost there.

#### Downgrading a structured invocation

When a capable client invokes a `public` command in a channel, the server relays
the `+draft/bot-cmd` `TAGMSG` as usual and additionally synthesises, from the
invoker, a plain `PRIVMSG` rendering the invocation in the bot's legacy form,
`<prefix><name> <values...>`, with the option values in schema order:

    ; Alice's capable client sent: @+draft/bot-cmd=<base64> TAGMSG #general
    S: :Alice!~a@host PRIVMSG #general :!weather london

The server delivers this synthesised line only to channel members that have not
negotiated `draft/bot-cmds`, so a capable client (which renders the structured
form) never sees it twice. The server MUST NOT deliver it to any user with the
`+B` [bot mode][] flag: a capable bot already received the `TAGMSG`, and a legacy
bot MUST NOT be induced to act on a command it was not the target of. The
rendering is best-effort and lossy (a value containing spaces does not
round-trip); it exists so a human on a legacy client can see what was run, not as
a precise encoding.

#### Upgrading a legacy invocation

When a user sends a plain message beginning with a registered bot's `prefix`
immediately followed by one of that bot's command names, in a channel the bot is
in or in a private message to it, the server synthesises a `+draft/bot-cmd`
`TAGMSG` from that user, mapping the whitespace-separated arguments to the
command's options in schema order:

    ; Carol's legacy client sent: :Carol!~c@host PRIVMSG #general :!weather paris
    S: @+draft/bot-cmd=<base64> :Carol!~c@host TAGMSG #general
    ;   decoded: {"name":"weather","options":{"city":"paris"}}

The server delivers the synthesised `TAGMSG` to the matching bot *in place of* the
original line, and relays the original line to every other member unchanged. The
bot therefore acts on the structured invocation exactly once and never receives
the raw text, so it needs no knowledge of whether the server is translating: on a
translating server it receives only the `+draft/bot-cmd`, and on a non-translating
server it receives only the raw line, which it may parse itself. A server MUST NOT
translate a message it synthesised, so the two directions cannot loop.

### Workflow transparency

This section defines the `draft/bot-tools` capability: how a bot streams the
multi-step workflow behind a reply. The model is push-only: the bot streams its
state to the channel as it runs, and clients observe and optionally interact;
clients do not poll.

#### The `+draft/bot-tools` tag

All workflow state is carried in a single [client-only tag][client-only tags]
named `draft/bot-tools`. Its base64 value decodes to a JSON object (see
[Value encoding](#value-encoding)). Using one tag with a JSON body, rather than
many tags, avoids exhausting a client's `clienttagdeny` slot capacity.

A server that offers `draft/bot-tools` SHOULD relay this tag only to clients that
have negotiated it. A server that does not offer the capability relays it as an
ordinary client-only tag, to every client that negotiated [message tags][];
clients that do not implement this specification ignore it.

#### Message schema

Every decoded `+draft/bot-tools` value is a JSON object with a `msg` field that
identifies its type: `workflow`, `step`, or `action`.

A **`workflow`** message reports the lifecycle of a whole workflow, and is sent
by the bot to the channel:

| Field | Type | Required | Description |
|---|---|---|---|
| `msg` | string | yes | `"workflow"` |
| `id` | string | yes | Opaque workflow identifier, unique per bot per session. |
| `state` | string | yes | One of the [workflow states](#states). |
| `name` | string | no | Human-readable label, e.g. `"Searching the web"`. |
| `trigger` | string | no | `msgid` of the message that triggered this workflow. |
| `cancelled-by` | string | no | Nick of the canceller; set when `state` is `cancelled`. |
| `features` | array of string | no | Behaviours the bot supports; see below. SHOULD be set on the `start` message. |

The `features` array is how a bot advertises what it supports, so a client can
show the right controls before any step arrives. Defined values:

| Feature | Meaning |
|---|---|
| `interactive` | The bot accepts `action` control signals (cancel, approve, reject, input). |
| `reasoning` | The bot emits `reasoning` steps exposing its planning. |
| `approval` | The bot uses approval gates (`pending-approval` steps). |

A **`step`** message reports one step within a workflow, and is sent by the bot
to the channel:

| Field | Type | Required | Description |
|---|---|---|---|
| `msg` | string | yes | `"step"` |
| `wid` | string | yes | The `id` of the workflow this step belongs to. |
| `sid` | string | yes | Opaque step identifier, unique within the workflow. |
| `type` | string | yes | One of the [step types](#states). |
| `state` | string | yes | One of the [step states](#states). |
| `tool` | string | no | Tool name for a `tool-call` step; see [Tool names](#tool-names). |
| `label` | string | no | Human-readable description of the step. |
| `content` | any | no | The step payload; see [the content field](#the-content-field). |
| `truncated` | bool | no | `true` if `content` was truncated to fit the tag size limit. |
| `cancelled-by` | string | no | Nick of the canceller; set when step `state` is `cancelled`. |

An **`action`** message is a control signal, sent by a client directly to the
bot's nickname:

| Field | Type | Required | Description |
|---|---|---|---|
| `msg` | string | yes | `"action"` |
| `action` | string | yes | One of the [actions](#states). |
| `target` | string | yes | The workflow `id` or step `sid` the action targets. |
| `content` | string | no | For `input`: the data or instruction to provide to the workflow. |

```json
{"msg":"workflow","id":"7f3a9b","state":"start","name":"Research query","trigger":"m0042","features":["interactive","reasoning"]}
{"msg":"step","wid":"7f3a9b","sid":"s2","type":"tool-call","state":"start","tool":"web-search","content":{"query":"who invented IRC"}}
{"msg":"action","action":"input","target":"7f3a9b","content":"use the staging server"}
```

#### States

**Workflow `state`:**

| State | Meaning |
|---|---|
| `start` | The workflow has begun; sent before any steps. |
| `reasoning` | The bot is reasoning or planning, with no step executing. |
| `running` | One or more steps are executing. |
| `complete` | The workflow finished successfully; the reply follows or has been sent. |
| `failed` | The workflow ended with an unrecoverable error. |
| `cancelled` | The workflow was cancelled by a control signal. |

**Step `type`:**

| Type | Meaning |
|---|---|
| `reasoning` | Internal reasoning or planning, with no external side effects. |
| `tool-call` | The bot is invoking a tool; the `tool` field MUST be set. |
| `tool-result` | The result of a preceding `tool-call` step. |
| `text` | A streaming text fragment generated before the reply. |

**Step `state`:**

| State | Meaning |
|---|---|
| `start` | The step has begun. |
| `running` | The step is executing. |
| `pending-approval` | The bot is paused, awaiting an `approve` or `reject`. |
| `complete` | The step finished successfully. |
| `failed` | The step failed; the workflow may continue or halt. |
| `cancelled` | The step was aborted by a cancel signal. |

**`action`:**

| Action | Meaning | Target |
|---|---|---|
| `cancel` | Abort the entire workflow. | workflow `id` |
| `approve` | Allow a `pending-approval` step to proceed. | step `sid` |
| `reject` | Deny a `pending-approval` step; the bot MUST NOT execute it. | step `sid` |
| `input` | Provide additional input to a running workflow. | workflow `id` |

A bot MUST honour an `approve`/`reject` only for a step in `pending-approval`,
and MUST maintain an access-control list for control signals (see
[Security considerations](#security-considerations)).

#### The content field

The type of a step's `content` depends on the step `type`:

| Step type | `content` |
|---|---|
| `tool-call` | A JSON object of the tool's input parameters (nested, not stringified). |
| `tool-result` | A string summarising the result (not necessarily the full output). |
| `reasoning` | A string with a brief excerpt of the bot's reasoning. |
| `text` | A string holding a streaming text fragment. |

For a `tool-call`, `content` is a native JSON object so that a client can render
the arguments directly rather than decoding a doubly-encoded string.

#### Tool names

The `tool` field of a `tool-call` step is a short label for the tool being
invoked, used by clients for display (for example to choose an icon for a search,
a shell command, or a file edit). It SHOULD be lowercase kebab-case, for example
`web-search`, `read-file`, or `shell`. This specification does not define a fixed
set of names: a client MAY recognise common names and render affordances for
them, but MUST treat any name it does not recognise as an opaque display string.

### Batch types

This specification defines three [batch][] types.

`draft/bot-cmds` groups the fragments of a [command list](#discovering-commands)
too large to send in one message.

`draft/bot-tools-workflow` groups the complete message sequence of a finished
workflow so a client can process it atomically, for history replay, scroll-back
rendering, or log display. Its parameter is the workflow `id`:

    BATCH +<ref> draft/bot-tools-workflow <workflow-id>

A bot MUST NOT use this batch type for real-time updates of a live workflow,
where message ordering as it arrives is the point.

`draft/bot-tools-content-stream` delivers a single large string-valued step
`content` (a `tool-result`, `text`, or `reasoning` step) as a sequence of
fragments, for when truncation is undesirable. Its parameters are the workflow
`id` and the step `sid`:

    BATCH +<ref> draft/bot-tools-content-stream <workflow-id> <step-id>

Each message in the batch is a `TAGMSG` carrying `+draft/bot-tools` with a `step`
message whose `content` holds one fragment; a client MUST concatenate the
fragments in arrival order. The terminating `state=complete` (or `failed`)
message MUST be sent after the closing `BATCH` line, not inside the batch, and a
message inside the batch MUST NOT set `truncated`.

## Implementation considerations

This section is non-normative.

### Clients

A client using `draft/bot-cmds` can drive autocomplete, inline help, and command
palettes from the command list, validate options against their `type` and
`choices` before sending, and present commands grouped by bot. A natural UI is a
`/`-prefixed command entry where `/weather city:london` is encoded into a
`+draft/bot-cmd` invocation; a `/command@botnick` form lets the user address a
specific bot when several share a name. A client may show a pending indicator
while awaiting a reply, correlating the reply by its `+reply` msgid.

A client using `draft/bot-tools` is encouraged to render a collapsible workflow
tree inline with the message that triggered it (keyed on `trigger`), showing each
step with its `type`, `tool`, and `state` (for example a spinner while running, a
check on completion, a cross on failure), pretty-printing `tool-call` arguments
and rendering `tool-result` content readably. It should highlight
`pending-approval` steps with approve/reject affordances, offer a cancel
affordance for a running workflow whose `features` include `interactive`, and
collapse a completed tree by default once the reply arrives. It should associate
the final reply with its workflow via the `id` on that message. It should not
present command, discovery, or workflow `TAGMSG` as chat, nor block the user from
sending messages while a workflow runs.

Each bot's workflow is independent; there is no parent/child relationship between
workflow `id`s across bots. If one bot invokes another as a tool, the second
bot's workflow appears in the channel as its own standalone workflow.

### Bots

A bot exposing commands should keep its advertised list accurate and emit
`+draft/bot-cmds-changed` when it changes. On receiving an invocation it should
validate and gate it (above) and reply reasonably promptly, since a client may
give up after a few seconds, correlating the reply with `+reply`.

A bot exposing workflows should send a `workflow` `start` before any steps, a
`step` `start` when a tool call begins and `complete`/`failed` when it ends, and
a terminal `workflow` `complete`/`failed` after the final reply. It should never
leave a workflow non-terminal indefinitely, sending `failed` on timeout. It should use
unique `id`/`sid` values per session, respond to `approve`/`reject` promptly, and
check the encoded tag size before sending, truncating `content` (and setting
`truncated`) when needed. When a workflow ends with a reply, that reply SHOULD
carry the terminal `workflow` message (the workflow `id` with `state` `complete`
or `failed`) so the workflow completes together with the reply and clients can
correlate the two; a separate terminal `TAGMSG` is then redundant and SHOULD NOT
be sent. A workflow that ends without a reply sends its terminal `workflow`
message as a standalone `TAGMSG`.

A bot SHOULD rate-limit `+draft/bot-tools` output (a few messages per second per
workflow is reasonable), and tool inputs or results that may carry sensitive data
SHOULD be omitted or truncated rather than streamed in full.

### Servers

A server that supports [message tags][] and relays `TAGMSG` is compatible with
this specification with no new behaviour: the tags reach every `message-tags`
client, and ones that do not implement this specification ignore them. A server
that offers `draft/bot-tools` SHOULD additionally relay `+draft/bot-tools` only
to clients that negotiated it, sparing the rest the workflow stream. Beyond
relay, the only command behaviour this specification defines for a server is the
optional [legacy compatibility](#legacy-command-compatibility) translation;
absent that, command discovery, invocation, and error reporting are entirely
between clients and bots.

## Security considerations

Every command list, invocation, and workflow message is supplied by another
client and MUST be treated as untrusted input. A receiver MUST catch parse errors
gracefully and silently discard a malformed or oversized value. A client MUST NOT
execute or evaluate any schema field (`name`, `description`, option text) or any
`content` value; they are display strings only. base64 decoding yields arbitrary
bytes, so validate the decoded JSON and apply application-level validation to any
free-text field (in particular the `content` of an `input` action) before use.

Any nickname can send a `TAGMSG` carrying these tags. A client MUST NOT grant
elevated trust to a command list or workflow message without verifying the
sender. For example, a client may require the bot to be authenticated (via
[account tag][]) and/or to carry the `+B` [bot mode][] flag, may require it to be
voiced or opped, and should show when a message comes from an unverified source.
`+draft/channel-context` is set by the sender and is not proof of channel
membership; a client MUST NOT use it for access-control decisions.

A bot MUST evaluate a command's `requires` against the invoker before acting, and
MUST maintain an access-control list for workflow `action` control signals. By
default it SHOULD accept `cancel`, `approve`, and `reject` from the user who
triggered the workflow and from channel operators, and `input` from the user who
triggered the workflow. It MUST NOT accept control signals from other users
without explicit configuration, and MUST silently ignore an unauthorised signal
rather than echo an error to the channel.

A bot SHOULD rate-limit its `TAGMSG` output, and a client SHOULD rate-limit
discovery queries, to limit abuse.

The [legacy compatibility](#legacy-command-compatibility) layer has the server
emit messages on a user's behalf: a `PRIVMSG` in the user's name carrying
downgraded text, and a `+draft/bot-cmd` in the user's name carrying an upgraded
invocation. Each MUST be attributed to the user who prompted it and MUST
faithfully reflect that user's input; a server MUST NOT synthesise a command a
user did not invoke. A server MUST NOT translate a message it synthesised, MUST
NOT deliver downgraded text to `+B` users, and SHOULD subject synthesised messages
to the same rate limits and access checks as ordinary ones.

## Examples

base64 tag values are shown as `<base64>`, with the decoded JSON in a preceding
`;` comment.

### Discovering and invoking a slash command

```irc
; Alice joins #general and asks any bots present for their commands.
C: @+draft/bot-cmds-query TAGMSG #general

; WeatherBot replies privately. Decoded +draft/bot-cmds:
;   {"commands":[{"name":"weather","description":"Look up the weather.",
;    "contexts":["public","pm"],
;    "options":[{"name":"city","type":"string","required":true}]}]}
S: @+draft/bot-cmds=<base64> :WeatherBot!bot@example.com TAGMSG Alice

; Alice runs /weather city:london (a public command, invoked in-channel).
; Decoded +draft/bot-cmd: {"name":"weather","options":{"city":"london"}}
C: @+draft/bot-cmd=<base64>;msgid=m42 :Alice!~a@host TAGMSG #general

; WeatherBot replies, correlating with the invocation's msgid.
S: @bot;+reply=m42 :WeatherBot!bot@example.com PRIVMSG #general :London: 14°C, light rain
```

### A private command with channel context

```irc
; The channel travels in the payload, since +draft/channel-context is not valid on TAGMSG.
; Decoded +draft/bot-cmd: {"name":"balance","options":{},"channel":"#casino"}
C: @+draft/bot-cmd=<base64>;msgid=m43 :Alice!~a@host TAGMSG BankBot

; The reply is whispered back, tagged with the channel it belongs to.
S: @bot;+reply=m43;+draft/channel-context=#casino :BankBot!bot@example.com NOTICE Alice :Your balance is 1,200 chips.
```

### Bridging legacy and capable clients

```irc
; #general holds Alice (capable), Carol (legacy), and WeatherBot (capable, +B).
; WeatherBot advertised, with a prefix so the server can translate:
;   {"prefix":"!","commands":[{"name":"weather","contexts":["public"],
;    "options":[{"name":"city","type":"string","required":true}]}]}

; Alice invokes the structured command; the server downgrades it for Carol only.
; Decoded +draft/bot-cmd: {"name":"weather","options":{"city":"london"}}
C: @+draft/bot-cmd=<base64> :Alice!~a@host TAGMSG #general
S: :Alice!~a@host PRIVMSG #general :!weather london          ; synthesised, to Carol only

; Carol invokes the legacy way; the server upgrades it for WeatherBot, in place of
; the raw line, and relays the raw line to everyone else.
C: :Carol!~c@host PRIVMSG #general :!weather paris
; decoded synthesised +draft/bot-cmd: {"name":"weather","options":{"city":"paris"}}
S: @+draft/bot-cmd=<base64> :Carol!~c@host TAGMSG #general   ; synthesised, to WeatherBot
```

### A multi-step workflow

```irc
; Decoded +draft/bot-tools: {"msg":"workflow","id":"7f3a9b","state":"start","name":"Research query","trigger":"m42"}
S: @+draft/bot-tools=<base64> :HelperBot!bot@example.com TAGMSG #general

; {"msg":"step","wid":"7f3a9b","sid":"s1","type":"reasoning","state":"complete","content":"Search for IRC history"}
S: @+draft/bot-tools=<base64> :HelperBot!bot@example.com TAGMSG #general

; {"msg":"step","wid":"7f3a9b","sid":"s2","type":"tool-call","state":"start","tool":"web-search","content":{"query":"who invented IRC"}}
S: @+draft/bot-tools=<base64> :HelperBot!bot@example.com TAGMSG #general

; {"msg":"step","wid":"7f3a9b","sid":"s3","type":"tool-result","state":"complete","content":"Jarkko Oikarinen, 1988, University of Oulu"}
S: @+draft/bot-tools=<base64> :HelperBot!bot@example.com TAGMSG #general

; The reply correlates to the invocation and carries the terminal workflow message, so the workflow completes with the reply.
; reply tag decodes to {"msg":"workflow","id":"7f3a9b","state":"complete"}
S: @bot;+reply=m42;+draft/bot-tools=<base64> :HelperBot!bot@example.com PRIVMSG #general
; Reply text:
; IRC was invented by Jarkko Oikarinen at the University of Oulu in August 1988.
```

### A tool-call approval gate

```irc
; {"msg":"workflow","id":"c9d8e7","state":"start","features":["interactive","approval"]}
S: @+draft/bot-tools=<base64> :adminbot!bot@example.com TAGMSG #admin

; The bot wants to run a shell command and pauses for approval.
; {"msg":"step","wid":"c9d8e7","sid":"s1","type":"tool-call","state":"pending-approval","tool":"shell","content":{"cmd":"systemctl restart nginx"}}
S: @+draft/bot-tools=<base64> :adminbot!bot@example.com TAGMSG #admin

; A channel operator rejects it (sent to the bot's nick).
; {"msg":"action","action":"reject","target":"s1"}
C: @+draft/bot-tools=<base64> :alice!alice@host TAGMSG adminbot

; The bot acknowledges and winds down.
; {"msg":"step","wid":"c9d8e7","sid":"s1","type":"tool-call","state":"cancelled","tool":"shell","cancelled-by":"alice"}
S: @+draft/bot-tools=<base64> :adminbot!bot@example.com TAGMSG #admin
; {"msg":"workflow","id":"c9d8e7","state":"cancelled","cancelled-by":"alice"}
S: @+draft/bot-tools=<base64> :adminbot!bot@example.com TAGMSG #admin
```

### Streaming a large tool result

```irc
S: :bot!bot@example.com BATCH +cs1 draft/bot-tools-content-stream 7f3a9b s3
; each fragment decodes to {"msg":"step","wid":"7f3a9b","sid":"s3","type":"tool-result","state":"running","content":"<fragment>"}
S: @batch=cs1;+draft/bot-tools=<base64> :bot!bot@example.com TAGMSG #channel
S: @batch=cs1;+draft/bot-tools=<base64> :bot!bot@example.com TAGMSG #channel
S: :bot!bot@example.com BATCH -cs1
; {"msg":"step","wid":"7f3a9b","sid":"s3","type":"tool-result","state":"complete"}
S: @+draft/bot-tools=<base64> :bot!bot@example.com TAGMSG #channel
```

[message tags]: https://ircv3.net/specs/extensions/message-tags
[client-only tags]: https://ircv3.net/specs/extensions/message-tags#client-only-tags
[client-only tag]: https://ircv3.net/specs/extensions/message-tags#client-only-tags
[bot mode]: https://ircv3.net/specs/extensions/bot-mode
[bot tag]: https://ircv3.net/specs/extensions/bot-mode#the-bot-tag
[batch]: https://ircv3.net/specs/extensions/batch
[labeled response]: https://ircv3.net/specs/extensions/labeled-response
[message ids]: https://ircv3.net/specs/extensions/message-ids
[account tag]: https://ircv3.net/specs/extensions/account-tag
[base64]: https://datatracker.ietf.org/doc/html/rfc4648#section-4
[`reply`]: https://ircv3.net/specs/client-tags/reply
[`channel-context`]: https://ircv3.net/specs/client-tags/channel-context
[channel-bots]: channel-bots.md
