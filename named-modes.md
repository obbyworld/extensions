---
title: Vendored named mode names
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Obby Team"
    period: "2026"
---

# named-modes

This is a work-in-progress specification.

## Notes for implementing experimental vendor extension

This is an experimental specification for a vendored extension.

No guarantees are made about the stability of this extension.
Backwards-incompatible changes can be made at any time without prior notice.

Software that implements this work-in-progress specification MUST NOT use the
unprefixed mode names. Instead, implementations SHOULD use the vendor-prefixed
names, so that they work with other software that implements a compatible
work-in-progress version.

## Description

This document gives the names that ObbyIRCd uses for its modes under the
[IRCv3 named modes] extension. Each name maps to a mode letter of the
UnrealIRCd family.

A client MUST match on the full name, including the prefix. A client that does
not know a name MUST show the raw name rather than hide the mode.

### Channel modes

Name | Parameter | Description
-----|-----------|------------
`obsidianirc/floodprot` | Yes | Per-channel flood rules
`obsidianirc/floodprot-profile` | Yes | A named preset of flood rules
`obsidianirc/censor` | No | Replaces the words of a bad-word list
`obsidianirc/nocolor` | No | Rejects messages that contain color codes
`obsidianirc/stripcolor` | No | Removes color codes from messages
`obsidianirc/noctcp` | No | Drops CTCP messages other than ACTION
`obsidianirc/nonotice` | No | Rejects notices to the channel
`obsidianirc/noinvite` | No | Rejects invitations to the channel
`obsidianirc/noknock` | No | Rejects KNOCK on the channel
`obsidianirc/nokick` | No | Rejects KICK on the channel
`obsidianirc/nonickchange` | No | Rejects nick changes in the channel
`obsidianirc/regonlyspeak` | No | Only logged-in users can talk
`obsidianirc/operonly` | No | Only IRC operators can join
`obsidianirc/delayjoin` | No | Hides a member until they talk
`obsidianirc/delayjoin-rejoinhide` | No | Hides a member again after they rejoin
`obsidianirc/history` | Yes | The amount of history that the channel keeps
`obsidianirc/link` | Yes | The channel that users go to when this one is full
`obsidianirc/issecure` | No | Every member is connected with TLS
`obsidianirc/isregistered` | No | The channel is registered with services

### User modes

Name | Description
-----|------------
`obsidianirc/regonlymsg` | Only logged-in users can send a private message
`obsidianirc/secureonlymsg` | Only users connected with TLS can send a private message
`obsidianirc/privdeaf` | The user does not receive private messages

### Ban types

Name | Description
-----|------------
`obsidianirc/timedban` | A ban that expires after a set time

[IRCv3 named modes]: https://github.com/ircv3/ircv3-specifications/pull/484
