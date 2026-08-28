# Obby IRC extensions

This repository documents the Obby-specific IRC extensions. The extensions are
under the vendored `obby.world` and `obsidianirc` namespaces.

`obsidianirc` is the older namespace. It stays in use because the name travels
on the wire, and a rename breaks every client that reads it. New extensions use
`obby.world`.

## Extensions

Name | Namespace | Kind
-----|-----------|-----
[e2ee](e2ee.md) | `obby.world/e2ee` | Client-only message tag
[filehost](filehost.md) | `obby.world/FILEHOST` | ISUPPORT token
[whois-batch](whois-batch.md) | `obby.world/whois` | Batch type
[channel-bots](channel-bots.md) | `obby.world/channel-bots` | Capability, tag
[cmdslist](cmdslist.md) | `obsidianirc/cmdslist` | Capability, command, batch type
[link-preview](link-preview.md) | `obsidianirc/link-preview-*` | Message tags
[voice](voice.md) | `obsidianirc/voice` | Capability, client-only tag
[named-modes](named-modes.md) | `obsidianirc` | Named mode names

## Implementations

The server is [ObbyIRCd]. The reference client is [Obby].

## License

Copyright the Obby Team.

Unlimited redistribution and modification of these documents is allowed provided
that the copyright notice of each document and this permission notice remain
intact.

[ObbyIRCd]: https://github.com/obbyworld/ObbyIRCd
[Obby]: https://github.com/obbyworld/obby
