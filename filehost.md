---
title: The FILEHOST ISUPPORT token
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Obby Team"
    period: "2026"
---

# filehost

This is a work-in-progress specification.

## Motivation

The IRCv3 [filehost] specification gives servers the `draft/FILEHOST` ISUPPORT
token, which names a service where a user can upload a file.

That token describes a service that anyone can post to. Some networks run an
uploader that accepts only their own users. A client cannot tell the two apart
from one token, and it needs to know, because one of them needs credentials.

This specification separates the two.

## Architecture

This specification introduces the `obby.world/FILEHOST` ISUPPORT token.

Token | Uploader | Credentials
------|----------|------------
`obby.world/FILEHOST` | Only for users of this network | Necessary
`draft/FILEHOST` | Open to anyone | None

The value of each token is a space-separated list of URIs. The rules of
[filehost] apply to both tokens: each URI SHOULD use the `https` scheme, and a
client MUST ignore a URI scheme that it does not support.

A server MAY advertise both tokens at the same time, with a different list in
each.

### Credentials

A client that uploads to a `obby.world/FILEHOST` URI MUST authenticate the
request. It mints a token with `draft/authtoken`, then sends it as an HTTP
Bearer credential.

A client that uploads to a `draft/FILEHOST` URI sends no credentials.

A client SHOULD prefer `obby.world/FILEHOST` when both are present. A file on
the uploader of the network stays under the rules of that network.

## Examples

A server that runs both kinds of uploader:

    :irc.example.org 005 alice obby.world/FILEHOST=https://irc.example.org/upload draft/FILEHOST=https://s.example.net/

An upload to the uploader of the network:

    POST /upload HTTP/1.1
    Host: irc.example.org
    Content-Type: image/jpeg
    Content-Disposition: attachment; filename="picture.jpeg"
    Content-Length: 4242
    Authorization: Bearer c2V1bmdoeWU6bm8=

[filehost]: https://github.com/ircv3/ircv3-specifications/pull/562
