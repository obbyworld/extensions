---
title: The link-preview message tags
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Obby Team"
    period: "2026"
---

# link-preview

This is a work-in-progress specification.

## Motivation

A client that shows a preview of a link must fetch the page itself. This has
three problems. Every client in the channel fetches the same page, so one
message causes many requests. The fetch tells the site the IP address of each
reader. A client that refuses to fetch, to protect the reader, shows no preview
at all.

This specification moves the fetch to the server. The server reads the page one
time and sends the result to the channel. A client shows the preview without
any request of its own.

## Architecture

This specification introduces three message tags:

Tag | Maximum length | Description
----|----------------|------------
`obsidianirc/link-preview-title` | 500 | The title of the page
`obsidianirc/link-preview-snippet` | 500 | A short description of the page
`obsidianirc/link-preview-meta` | 2048 | The URL of the preview image

The title tag is always present. The other two tags are present only when the
page gives that information.

Servers MUST reject these tags from a client. Only a server can send them.
Without this rule, a user can send a message that shows a false title for a
link that goes somewhere else.

No capability is necessary. A client that does not know these tags ignores
them, and sees a `TAGMSG` with no content.

### How a preview is made

When a user sends a message to a channel, the server looks for a URL in the
text. If it finds one, the server fetches the page in the background. The
message goes to the channel immediately. The preview follows later, or never,
if the fetch fails.

The server sends the preview as a `TAGMSG` to the channel, from itself.

The `TAGMSG` carries the `+reply` tag, and its value is the msgid of the message
that contained the URL. A client uses this value to show the preview under the
right message.

Servers MUST limit the size of the page that they download. ObbyIRCd stops at
1 MB, and reads a URL of up to 2048 characters.

### The preview image

The value of the meta tag is a URL on the filehost of the server. The server
downloads the image of the page, then uploads it again to its own filehost.

Because of this, a client that shows the image makes a request to the server
only. The original site never learns the IP address of the reader.

Clients SHOULD show the image without a request to any other host.

## Examples

A user sends a message with a link:

    @msgid=Ac8h2Kd9 :alice PRIVMSG #general :look at this https://example.org/article

The server answers with the preview, a moment later:

    @+reply=Ac8h2Kd9;obsidianirc/link-preview-title=An\sExample\sArticle;obsidianirc/link-preview-snippet=A\sshort\sdescription\sof\sthe\spage.;obsidianirc/link-preview-meta=https://files.example.org/ab12cd34.jpg :irc.example.org TAGMSG #general

A page with a title and no description:

    @+reply=Ac8h2Kd9;obsidianirc/link-preview-title=An\sExample\sArticle :irc.example.org TAGMSG #general
