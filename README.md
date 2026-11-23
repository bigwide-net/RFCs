# Big Wide RFCs

This repository documents and discusses extensions to RSS, JSON Feed and other standards, made for the Big Wide social network initiative.

## Goals

These RFCs extend what the RSS universe can do, so that you can:

- **Write posts and comments in the client.** Today such a client is called an RSS reader. Once it can write, it is more than a reader.
- **Browse and edit social profiles**, based on [SocialSpec](https://socialspec.org).
- **Host group discussions.**
- **Show feeds the way Instagram and TikTok do**, as photos and videos rather than as lists of articles.
- **Do more in the average RSS reader**: styled text and media in posts, shown the same way everywhere.

All of this while:

1. **Keeping everything open and completely decentralized.** No central server, no registration, nothing that only one company can run.
2. **Giving publishers every possible freedom**, and making them independent of commercial platforms. Their content lives on their own domain, and they can change apps and hosts without losing it.
3. **Building small, interchangeable parts.** Each RFC does one thing, can be implemented on its own, and works with existing feeds, readers and servers.
4. **Keeping everything as simple as possible.**
5. **Not reinventing anything.** Existing standards come first, such as RSS, JSON Feed, the source and podcast namespaces, Webmention and OAuth. An RFC adds only what is missing, and builds on what exists.

## The RFCs

| RFC | What it adds |
| --- | --- |
| [RFC-1: Source Namespace Extensions for JSON Feed](RFC-1-Source-for-JSON-Feeds.md) | Replies, comment feeds, comment counts, accounts and Markdown from the RSS source namespace, for JSON Feed. |
| [RFC-2: Feed HTML](RFC-2-HTML-Subset-for-Feed-Content.md) | The subset of HTML and CSS that a post can use and expect to look the same in every reader, including embedded media. |
| [RFC-3: Podcast Namespace Extensions for JSON Feed](RFC-3-Podcasting-for-JSON-Feeds.md) | The Podcasting 2.0 namespace for JSON Feed. |
| [RFC-4: Feed Submissions](RFC-4-Feed-Submissions.md) | Comments and group discussions between feeds, with Webmention: the author publishes on their own domain, and the target republishes. |
| [RFC-5: Comment HTML and Markdown](RFC-5-Comment-HTML.md) | The formatting a comment or group post may use when one feed submits it to another. |
| [RFC-6: Personal Upload Server (PUPS)](RFC-6-Personal-Upload-Server.md) | Lets apps write files to the user's own domain with standard OAuth, and sign the user in with their domain. |
| [RFC-7: PUPS Feeds](RFC-7-Feed-Publishing-with-PUPS.md) | How a PUPS server keeps a blog feed on the user's domain, with RFC-13 archives, and how apps post to it. |
| [RFC-8: Fallback HTML](RFC-8-Fallback-HTML.md) | HTML in a post that only readers without a particular extension show, so feeds can adopt new extensions without losing older readers. |
| [RFC-9: Feed Actions](RFC-9-Feed-Actions.md) | Buttons, choices and inputs on JSON Feed items and feeds, such as moderating submissions or voting in a poll, sent to the publisher's API with a plain `POST`. |
| [RFC-10: Identity Tokens](RFC-10-Identity-Tokens.md) | Lets an app prove to any server that it acts for the owner of a domain, with a short-lived token from their PUPS server, e.g. to vote in a poll as `@tim.example.com`. |
| [RFC-11: Mentions](RFC-11-Mentions.md) | Tells a domain that a post names it, as in *@tim.example.com*, or replies to one of its posts or comments, with a Webmention to the endpoint in its `host-meta.json`. Sent by the system that publishes the post. |
| [RFC-12: Inbox Feeds](RFC-12-Inbox-Feeds.md) | A private JSON Feed at `/pups/private/inbox.json` on the owner's PUPS server, with their mentions and the comments awaiting their approval. One system writes it; the owner's readers show it as their inbox, optionally with actions such as approve, reject or block. |
| [RFC-13: JSON Feed Archives](RFC-13-JSON-Feed-Archives.md) | Keeps every item of a JSON Feed in archives that never lose or move an item, so that the archive's URL plus `#id` addresses a post forever, with item indexes for numbering posts and finding them without reading every archive. |

## What's missing

The main gaps that the RFCs don't cover yet:

- **Blocking users.** A way for authors and group owners to keep particular people out of their comments and discussions, and for readers to hide them.
- **Card-based feeds for photos and videos.** A feed format, or conventions for existing ones, that readers can show as cards or a vertical video stream, like Instagram and TikTok.
- **Links to posts in feeds.** Feeds name only an HTML page as a post's permalink. [RFC-13](RFC-13-JSON-Feed-Archives.md) gives every archived post a permanent address in its feed, the archive's URL plus `#id`, but links in a post can't name it yet. They could name both, as in `<a href="https://example.com/posts/42" data-feed-href="https://example.com/feed/feed.000003.json#42">`, so that readers which support it open the linked post right in the reader, and all others open the web page.

## Repository

The repository is hosted at https://github.com/bigwide-net/RFCs.
