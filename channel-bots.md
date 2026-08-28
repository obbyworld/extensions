---
title: The channel-bots capability
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Obby Team"
    period: "2026"
---

# channel-bots

This is a work-in-progress specification.

This document covers the client-facing part of the bot system. The gateway that
bot authors connect to is described in the [PushBot specification].

## Motivation

A bot on IRC is a user with a `+B` mode. A client cannot tell which bots are in
a channel, which commands each bot answers, or which of its own messages a bot
produced.

This specification gives the client that information, and it gives a bot a way
to mark the message that caused its reply.

## Architecture

This specification introduces the `obby.world/channel-bots` capability, the
`obby.world/bot-info` message tag, and the `+obby.world/invoked-by` client-only
message tag.

### Discovery

When the server acknowledges the capability, it sends the list of server-wide
bots. It sends the list of channel bots when the client joins a channel. It
sends an update when a bot is added, removed, or changes its commands.

Each list is a batch of type `obby.world/bot-list`. Each member of the batch is
a `TAGMSG` that carries the `draft/bot-cmds` tag. One format covers all three
cases, so a client writes one parser.

### Invocation

The capability enables structured slash-command invocation. A client without the
capability sees a bot as a plain `+B` user, and can only talk to it with
`PRIVMSG`.

### Attribution

A bot MAY put the msgid of the message that triggered it in the
`+obby.world/invoked-by` tag on its reply. A client can then show the reply next
to the message that caused it.

The server MUST relay this tag without inspection, in the same way as other
client-only tags.

## Permissions

The `manage-bots` member-role permission is channel-scoped. A user who holds it
can add a bot to the channel, remove a bot, and answer pending requests to add a
bot.

Any member of a channel can ask for a bot to be added. A bot can ask on its own
behalf. A pending request closes after 7 days. After a denial, the same pair of
bot and channel is limited to one new request every 24 hours.

## Examples

A discovery batch, after the capability is acknowledged:

    :irc.example.org BATCH +xyz obby.world/bot-list
    @batch=xyz;draft/bot-cmds=weather,forecast :weatherbot TAGMSG *
    :irc.example.org BATCH -xyz

A bot reply that names the message that triggered it:

    @+obby.world/invoked-by=Ac8h2Kd9 :weatherbot PRIVMSG #general :22C and clear.

[PushBot specification]: https://github.com/obbyworld/ObbyIRCd/blob/unreal60_dev/doc/specs/pushbot-spec.md
