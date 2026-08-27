---
title: The voice capability and the rtc tag
layout: spec
work-in-progress: true
copyrights:
  -
    name: "ObbyIRCd Team"
    period: "2026"
---

# voice

This is a work-in-progress specification.

## Motivation

Voice and video calls need a signaling channel. The two sides must exchange
session descriptions and network candidates before any media flows.

IRC already gives a room, a member list, and an identity for each member. This
specification uses the IRC connection to carry WebRTC signaling, so a call needs
no second account and no second connection.

## Architecture

This specification introduces the `obsidianirc/voice` capability and the
`+obsidianirc/rtc` client-only message tag.

Two channel prefixes are voice channels:

- `^` is a voice channel. Every member can publish audio and video.
- `$` is a stream channel. Some members publish, and the rest watch.

Servers MUST refuse a `JOIN` to these channels from a client that did not
negotiate `obsidianirc/voice`. A plain IRC client that joins sees only signaling
traffic that it cannot read, so the server hides these channels from it.

Clients carry signaling in the `+obsidianirc/rtc` tag on a `TAGMSG` to the
channel. The value is a JSON object.

The server does not implement WebRTC. It validates the tag, then forwards the
frame to a media server. The media server answers through the same tag.

Media never flows between two clients. Every track goes through the media
server, which is a Selective Forwarding Unit (SFU). Media is encrypted between
each client and the SFU with DTLS-SRTP.

## Signal types

The `type` field gives the frame type.

Type | Direction | Description
-----|-----------|------------
`join` | Client to server | A request to enter the room
`joined` | Server to client | The member list and the TURN credentials
`offer` | Client to server | A session description from the client
`answer` | Server to client | A session description from the SFU
`ice` | Both | A network candidate
`presence` | Both | A state change of one member
`react` | Both | A transient emoji, visible to the room
`promote` | Both | A viewer becomes a publisher
`demote` | Both | A publisher becomes a viewer

A `presence` frame carries the state name in the `kind` field. The states are
`mic`, `video`, `speaking`, `silent`, `deaf`, `screen`, and `hand`.

## TURN credentials

The server answers a `join` frame with TURN credentials in the `turn` field.
The credentials are short-lived and are minted per user, as described in
`draft-uberti-behave-turn-rest`.

The server MUST rewrite these credentials for each recipient. A client MUST NOT
receive the credentials of another user.

## Examples

A join request:

    @+obsidianirc/rtc={"type":"join","channel":"^general"} :alice TAGMSG ^general

An offer:

    @+obsidianirc/rtc={"type":"offer","sdp":"v=0\r\no=- 42…"} :alice TAGMSG ^general

A presence change, when Alice mutes:

    @+obsidianirc/rtc={"type":"presence","kind":"mic","state":"0"} :alice TAGMSG ^general

A refused join, from a client without the capability:

    :irc.example.org 403 alice ^general :Voice channels require the obsidianirc/voice client capability
