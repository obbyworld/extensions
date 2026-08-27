---
title: The e2ee client-only tag
layout: spec
work-in-progress: true
copyrights:
  -
    name: "ObbyIRCd Team"
    period: "2026"
---

# e2ee

This is a work-in-progress specification.

## Motivation

Private messages on IRC are readable by the server and by any bouncer between
the two clients. This specification gives two clients a way to encrypt private
messages so that only they can read them.

The server relays the encrypted bytes. It never holds a key and it cannot read
the contents.

## Architecture

This specification introduces the `+obby.world/e2ee` client-only message tag.

The tag value is opaque. It is the base64 form of an encrypted blob. Servers
MUST NOT inspect the value. Servers MUST validate the shape of the tag and
relay it to the recipients, in the same way as other client-only tags.

The value is limited to 4094 bytes, which is the IRCv3 limit for client tag
data. A client MUST split a larger payload into fragments.

Servers MUST relay the tag on `TAGMSG` and on `PRIVMSG`. A server that strips
client-only tags breaks the handshake. A server that advertises `CLIENTTAGDENY`
MUST re-allow this tag if it supports this specification.

## Transport

The extension uses two carriers, and the choice is not free.

Handshake frames travel in the tag on a bodiless `TAGMSG`. Nothing in a
handshake is a message, so it stays invisible to clients that do not implement
this specification.

Message frames travel in the `PRIVMSG` body, behind the `?obe2ee:` marker. The
body is used because a message needs a real envelope. A `PRIVMSG` has a msgid,
so replies, reactions, redaction and history all continue to work. A `TAGMSG`
has none of these.

## Frames

Each frame is a JSON object. The `t` field gives the frame type. The `v` field
gives the protocol version, which is 1.

Type | Carrier | Description
-----|---------|------------
`init` | Tag | An offer to start an encrypted session
`accept` | Tag | An answer to an offer
`reject` | Tag | A refusal of an offer
`ack` | Tag | The first encrypted payload, which proves the session works
`close` | Tag | The end of a session
`msg` | Body | An encrypted text message
`media` | Body | An encrypted file

## Session

The two clients agree on keys with X3DH, then encrypt each message with a
Double Ratchet. The primitives are X25519, Ed25519, XChaCha20-Poly1305, and
HKDF over HMAC-SHA256.

The responder signs its ephemeral key with its identity key. The initiator MUST
verify this signature before it pins the fingerprint of the peer. Without this
check, an attacker can present the fingerprint of a real user over its own
session.

The responder MUST NOT show a session as established when it sends `accept`. It
must wait for the `ack` frame of the initiator to decrypt. An `accept` frame can
be lost, and a client that shows a lock too early tells the user something that
is not true.

A client pins the fingerprint of a peer the first time they talk. If the
fingerprint changes later, the client MUST stop the conversation and ask the
user before it continues.

## Crossing offers

Two clients can offer a session at the same moment. If both of them answer the
offer of the other, both answers are discarded and no session starts.

To prevent this, the client with the lower fingerprint keeps its own offer and
stays the initiator. The other client answers. If a client cannot read the
fingerprint of the peer, it answers.

## Media

An encrypted file is encrypted whole with XChaCha20-Poly1305, then uploaded to
the filehost with the `.obb` extension. The file starts with the 9-byte magic
`OBBYE2EE\x01`.

The receiver decrypts the whole file in memory. Because of this, clients SHOULD
refuse to encrypt a file larger than 25 MB. This limit protects the receiver,
which can be a phone.

## Examples

An offer, on a bodiless `TAGMSG`:

    @+obby.world/e2ee=eyJ0IjoiaW5pdCIsInYiOjEsImlrIjoi… :alice TAGMSG bob

An encrypted message, in the `PRIVMSG` body:

    :alice PRIVMSG bob :?obe2ee:eyJ0IjoibXNnIiwidiI6MSwiY3QiOiJ…

An `ISUPPORT` token that re-allows the tag:

    :irc.example.org 005 alice CLIENTTAGDENY=*,-obby.world/e2ee
