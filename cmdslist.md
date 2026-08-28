---
title: The cmdslist capability
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Obby Team"
    period: "2026"
---

# cmdslist

This is a work-in-progress specification.

## Motivation

A client that offers slash-command completion must know which commands the user
can call. The set is not fixed. It depends on the modules that the server loads,
on the oper status of the user, and on the services that are online.

Without this information, a client guesses. It offers commands that fail, and it
hides commands that work.

This specification gives the server a way to tell the client the exact set, and
to update the set when it changes.

## Architecture

This specification introduces the `obsidianirc/cmdslist` capability and the
`CMDSLIST` command.

The server sends the list inside a batch of type `obsidianirc/cmdslist`. The
`batch` capability is a prerequisite.

Each parameter of a `CMDSLIST` line is one entry:

- `+<command>` adds a command to the set.
- `-<command>` removes a command from the set.

The server packs entries onto each line until the line approaches the 512-byte
limit. Then it starts a new line inside the same batch. The client MUST
concatenate the parameters of every line in the batch to get the full set.

The batch gives the client a clear end marker. A stream of single lines has no
end, so the client can never know that the set is complete.

The server MAY send a later batch when the set changes. This batch carries only
the entries that changed.

## Examples

The full set, after connection registration:

    :irc.example.org BATCH +xyz obsidianirc/cmdslist
    @batch=xyz :irc.example.org CMDSLIST +join +part +topic +kick
    @batch=xyz :irc.example.org CMDSLIST +nick +away +oper
    :irc.example.org BATCH -xyz

A later update, after the user opers up:

    :irc.example.org BATCH +abc obsidianirc/cmdslist
    @batch=abc :irc.example.org CMDSLIST +kill +gline -oper
    :irc.example.org BATCH -abc
