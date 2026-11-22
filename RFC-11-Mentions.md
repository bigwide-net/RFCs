# RFC: Mentions - Telling a Domain That a Post Mentions or Answers It

Nov 19, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-11-Mentions.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-11-Mentions.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-11-Mentions.md)

## Summary

This proposal lets the owner of a domain learn that someone mentioned them or replied to them, wherever that happened. There are two kinds of mentions:

- **By name**: a post names the owner by their handle, e.g. *Great talk by @tim.example.com yesterday*.
- **By reply**: a post or comment answers a post or comment of the owner, e.g. a comment on their blog post, or a reply to their comment in a discussion group.

The system that publishes the post finds the endpoint in the mentioned domain's `host-meta.json` and sends it a [Webmention](https://www.w3.org/TR/webmention/). The receiving system fetches the post, checks that it really mentions or answers the owner, and puts it into the owner's inbox.

```json
{
  "links": [
    { "rel": "https://bigwide.net/ns/mention", "href": "https://tim.example.com/mention" }
  ]
}
```

```
POST /mention HTTP/1.1
Host: tim.example.com
Content-Type: application/x-www-form-urlencoded

source=https%3A%2F%2Falice.example.com%2Ffeed.json%23p-81&target=https%3A%2F%2Ftim.example.com%2F
```

This is the original purpose of Webmention, applied to feeds. Nothing is republished or published at all: a mention is a notification, not a submission. Comments and group posts are [Feed Submissions (RFC-4)](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md); this RFC only tells the people they mention or answer.

How the owner sees their mentions is [Inbox Feeds (RFC-12)](https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md): the receiver writes them into a private feed on the owner's PUPS server, at a fixed path where the owner's reader finds it.

**Why now.** RFC-4 brings conversations into feeds, but the participants don't learn about them. Someone who comments on a post in a group must check the group's comment feeds to see whether anyone answered. A blogger who is named in another blog learns about it by chance. Every social network solves this with notifications, and the RSS universe has nothing comparable. Webmention was designed for exactly this, but it notifies web pages, not people: its sender looks for the endpoint on the page that it links to. A handle names a domain, and a reply answers an item in a feed, often in a feed that isn't on the owner's domain at all, like a group's comment feed. Neither has a page to discover an endpoint on. The owner's domain does have `host-meta.json`, which [RFC-6](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md) already uses as the domain's index.

## Goals and non-goals

**Goals**

- **One place per person.** All mentions of a domain arrive at one endpoint, whether the post is on a blog, in a group or on a web page.
- **No accounts, no registration.** The receiver trusts no one who sends a mention. It fetches the post and checks it, as Webmention does.
- **Exactly one notification.** For each post and each mentioned person, exactly one system sends the mention: the one that publishes the post.
- **Plain Webmention.** The request is an ordinary webmention. Existing senders and receivers need only the new discovery.

**Non-goals**

- **The inbox.** How mentions are stored, shown and read by apps is [RFC-12](https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md).
- **Private messages.** Every mention points to a public post.
- **Blocking and spam filtering.** What a receiver accepts beyond the checks below is its own decision.
- **Comments.** Getting a comment published on someone else's post is RFC-4.

## What is new and what is reused

**Used as they are.**

| Spec | Used for |
| --- | --- |
| [Webmention](https://www.w3.org/TR/webmention/) | The request from sender to receiver, and the receiver's verification by fetching the source. |
| [RFC 6415](https://www.rfc-editor.org/rfc/rfc6415) `host-meta.json`, as in RFC-6 | Finding the mention endpoint of a domain. |
| `source:inReplyTo`, `source:account` and their JSON Feed form from [RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md) | What a post answers, and whose post that is. |
| RFC-4 conventions | Items referenced as feed URL + `#id`, author feeds, archives, and the server that republishes comments. |
| [microformats2](https://microformats.org/wiki/h-entry) `h-entry` and `u-in-reply-to` | Finding the post and its reply link in an HTML page. |

**New in this RFC.**

- **The link relation `https://bigwide.net/ns/mention`** in `host-meta.json`. See [Discovery](#discovery).
- **Handles in text**: how a post names a domain, and how both sides find it. See [Handles](#handles).
- **Domains and feed items as targets.** The target of a webmention is the domain itself, or the item that is answered. See [Target](#target).
- **Who sends**: the publishing system, and for feed submissions the server that republishes them, never the author. See [Who sends](#who-sends).

## Roles

- **Owner**: the person who controls a domain, e.g. `tim.example.com`, and whose handle is `@tim.example.com`.
- **Post**: an item in an RSS or JSON feed, or an HTML page, that mentions or answers the owner. In this RFC, "post" includes comments.
- **Sender**: the system that publishes the post and sends the mention: a blog engine, a feed-publishing app ([RFC-7](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md)), or an RFC-4 server that republishes comments and group posts.
- **Receiver**: the system behind the owner's mention endpoint. It verifies mentions and adds them to the owner's inbox. It may serve one domain or many.

## Overview

1. **Publish.** The sender publishes a post. It names `@tim.example.com` in its text, or answers an item whose author is `@tim.example.com`, or both.
2. **Find the account.** For each handle in the text, the account is the handle. For a reply, it is the socialspec account of the item answered. See [Who is mentioned](#who-is-mentioned).
3. **Discover.** The sender fetches `https://tim.example.com/.well-known/host-meta.json` and takes the link with the relation `https://bigwide.net/ns/mention`. Without one, the domain doesn't accept mentions, and the sender stops here.
4. **Send.** The sender posts a webmention to the endpoint. `source` is the post. `target` is `https://tim.example.com/` for a mention by name, or the answered item (its feed URL + `#id`) for a reply.
5. **Verify.** The receiver answers `202 Accepted`, then fetches the source and checks that it contains `@tim.example.com`, or that its `inReplyTo` is the target and the target is the owner's item.
6. **Deliver.** The receiver adds the mention to the owner's inbox.

![Sequence diagram: the sender publishes a post, fetches the mentioned domain's host-meta.json, sends the webmention to the mention endpoint; the receiver accepts it, fetches the post, verifies the handle or the reply, and adds the mention to the owner's inbox.](diagrams/RFC-11-mention-flow.svg)

## Handles

A handle is `@` followed by a host name, e.g. `@tim.example.com`. It names the whole domain, as a [socialspec](https://socialspec.org) account does, and as the handle of [RFC-10](https://github.com/bigwide-net/RFCs/blob/main/RFC-10-Identity-Tokens.md) does. There are no users within a domain: `tim.example.com` and `example.com` are different owners.

**Syntax.** The host name has at least two labels separated by dots. Each label consists of letters, digits and hyphens, and doesn't start or end with a hyphen; the last label starts with a letter. Internationalized names may be written in Unicode or in their ASCII form (`xn--…`).

**Finding handles in text.** Senders and receivers find handles the same way:

- **Where.** In the post's title and its text: the visible text of its HTML content, or its plain-text content if it has no HTML. Text inside `code` and `pre` doesn't count, and neither do attribute values, such as `href`. Link text counts, so `<a href="https://tim.example.com/">@tim.example.com</a>` is a handle.
- **Before the `@`**: the start of the text, whitespace, or a punctuation character other than `.`, `_`, `-`, `+`, `/` and `@`. That excludes e-mail addresses such as `tim@tim.example.com`.
- **After the host**: the end of the text, whitespace, or a punctuation character other than `-`, `_`, `/` and `@`. A trailing `.` ends the handle when it isn't followed by a letter or digit, so *I did this thanks to @tim.example.com.* names `tim.example.com`. That excludes fediverse addresses such as `@tim@social.example`, and paths.
- **Lists.** Handles separated by `,` or `;`, with or without spaces, are separate handles: `@tim.example.com,@alice.example.com` names both. So does a `/` directly between two handles: `@tim.example.com/@alice.example.com`. Everywhere else, a `/` after a handle makes it a path, and a `/` before an `@` an address in a URL, such as `https://social.example/@tim.example.com`.
- **Comparing.** Hosts are compared in lowercase ASCII form, after converting Unicode names with IDNA.

**Writing handles.** Where links are allowed, publishers should link each handle to the domain's root: `<a href="https://tim.example.com/">@tim.example.com</a>`. Readers can then open the profile, and generic Webmention receivers, which look for a link to the target, accept the mention too. Where links are not allowed, such as a comment under an RFC-4 policy without `allowLinks`, the plain handle is enough.

## Who is mentioned

### Mentions by name

Every handle in the post's text is an account to notify, with the target `https://<host>/`.

### Replies

A post replies to an item when it has `inReplyTo` (`<source:inReplyTo>` in RSS, `_source.inReplyTo` in JSON Feed). Its value is the feed URL of the answered item plus `#` and the item's id, as in RFC-4. The sender notifies the author of that item, with the `inReplyTo` value as the target.

The author is the item's socialspec account (`<source:account service="socialspec">`, `_source.accounts.socialspec`):

1. **The item's own account**, if it has one. An RFC-4 server writes it on every republished post and comment, after checking it.
2. **Otherwise the feed's account**, but only if the feed has no `submissions`. A feed with `submissions` carries posts by many people; its own account is the group's or the blog's, not the author's.
3. **Otherwise there is no one to notify**, and no reply mention is sent.

To read the account, the sender fetches the feed in the target, unless it has a current copy, and finds the item as RFC-4 does, following archive pages if needed. A sender that publishes the answered item itself, such as an RFC-4 server answering an item in its own feeds, looks it up directly.

Some replies have no author to notify:

- **A new post in a group** has the group feed itself as `inReplyTo`, without an `#id`. It answers no one.
- **A reply with a permalink** as `inReplyTo`, e.g. `https://rss.chat/?id=204`, as RFC-1 allows, doesn't name a feed item. Senders don't send reply mentions for it.

### Both

A post may mention several accounts, and may both answer and name the same one. The sender notifies each account once per post:

- If the account is the author of the answered item, it sends the reply mention only.
- Otherwise, it sends one mention by name.
- It doesn't notify the post's own author, e.g. someone who answers their own comment or names themselves.
- It should notify at most 20 accounts per post and may ignore the rest. Receivers may treat posts with many handles as spam.

## Discovery

The sender fetches `https://<host>/.well-known/host-meta.json` ([RFC 6415](https://www.rfc-editor.org/rfc/rfc6415), JSON form) and takes the first link whose `rel` is `https://bigwide.net/ns/mention`. Its `href` is the **mention endpoint**:

```json
{
  "links": [
    { "rel": "https://bigwide.net/ns/personal-storage", "type": "application/json", "href": "https://tim.example.com/personal-storage.json" },
    { "rel": "https://bigwide.net/ns/mention", "href": "https://inbox.example.net/u/tim.example.com/mention" }
  ]
}
```

- **The domain names its endpoint.** Every domain that wants mentions must have this link. The endpoint may be on another host, e.g. a hosted inbox service.
- **No link, no mention.** If `host-meta.json` doesn't exist, isn't valid JSON or has no such link, the domain doesn't accept mentions. The sender sends nothing and doesn't report an error to its user.
- **Fetching.** Discovery uses HTTPS only, follows at most 3 redirects, only to HTTPS URLs, and only `host-meta.json` on the domain itself. The endpoint must be an absolute `https` URL.
- **Caching.** Senders cache `host-meta.json` according to its HTTP cache headers, for at most 24 hours, and cache failed discoveries for at least an hour, so that a busy group doesn't fetch the same file for every comment.
- **Placeholder.** The relation is a URI and needs no registration. **`https://bigwide.net/ns/mention` is a placeholder** until it has a permanent home.

The link belongs to the owner, or to the system that runs their receiver. An app can't add it through an RFC-6 PUPS server unless the server's list of well-known paths allows the relation; the owner adds it, or the PUPS server for a receiver it runs itself.

## Sending

### Request

The sender posts a webmention to the endpoint, as the [Webmention spec](https://www.w3.org/TR/webmention/#sending-webmentions) describes: a `POST` with `source` and `target`, form-encoded. Receivers should accept a JSON body with the same two fields as well.

### Source

`source` is the post that mentions the account:

- **A feed item**: the URL of the feed it is published in, plus `#` and its id (RSS `<guid>`, JSON `id`), as in RFC-4. Feeds that move older items into archives should use a URL that the item never leaves, like the archives of JSON Feed Archives ([RFC-13](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md)), so that the receiver still finds it when it re-checks the mention later.
- **An HTML page**: its URL. A sender that publishes both a feed and pages should send the feed item, which the receiver can read without guessing.

### Target

| Mention | `target` | Example |
| --- | --- | --- |
| By name | the root URL of the domain | `https://tim.example.com/` |
| Reply | the `inReplyTo` of the post: feed URL + `#id` of the answered item | `https://tim.example.com/feed.json#p-12` |
| Reply to a republished comment | the comment feed that contains the comment + `#id` | `https://rm.example.org/groups/twowaves/comments/a8Kd03XpQz/comments.json#a8Kd03XpQz-k2m9x` |

The root URL is normalized as in RFC-6: `https`, the host in lowercase ASCII, no port, the path `/`, no query and no fragment.

### Who sends

**The system that publishes the post sends the mention.** It is the only system that knows when the post is public, and it publishes it under its own URL. A blog engine sends mentions for the posts of the blog; an app that writes the owner's feed through PUPS sends them for the items it adds. A reader or editor that only hands the post to a publishing system doesn't send them.

**Feed submissions are mentioned by the server, not by the author.** For a comment or group post under RFC-4:

- **The author's system must not send mentions** for the item in the author feed: neither for its `inReplyTo`, nor for the handles in its text. The RFC-4 webmention that submits the item is a submission, not a mention.
- **The RFC-4 server sends them**, but only once the item has been accepted, approved by moderation where the group moderates, and republished. A rejected or still pending submission notifies no one.
- **The source is the republished item**: the server's feed + `#` + the new guid. The receiver then points the owner to the conversation, where the answers are, and not to the author feed.
- **With `republish: summary`**, the server notifies the accounts named in the republished abstract, and the author of the answered item. **With `republish: no`**, nothing is published, so the server sends no mentions; the owner of the target feed receives the submission anyway.

This way the owner gets exactly one notification, and only for what readers can actually see.

### When

- **After publishing.** The sender sends the mentions as soon as the post can be fetched at `source`. If the post is served through a cache or a CDN, it waits until the new version is visible, or the receiver won't find the item or the handle.
- **Asynchronously.** Sending must not delay publishing, and a failed mention doesn't fail the post.
- **Retries.** After a network error, a `5xx` or a `429`, the sender retries with a growing interval (respecting `Retry-After`) for up to a day. It doesn't retry other `4xx` answers.

### Edits and deletes

As in the [Webmention spec](https://www.w3.org/TR/webmention/#sending-webmentions-for-updated-posts):

- **Edits.** When a post changes, the sender sends the mention again to every account that the old or the new version mentions. The receiver then adds, updates or removes the mention.
- **Deletes.** When a post is deleted, the sender sends the mention again to every account it mentioned. The source then answers `404` or `410`, or the feed no longer contains the item, and the receiver removes the mention.

An RFC-4 server does the same when an allowed edit or delete changes a republished item, and when moderation removes it.

## Receiving

### Response

The receiver does the cheap checks synchronously:

- `source` and `target` are absolute `http(s)` URLs, at most 2,048 characters long, and not the same.
- A target without a fragment is the root URL of a domain that this endpoint serves.
- A target with a fragment is not rejected at this point, since only fetching shows whose item it is.

If a check fails, it answers `400` with a JSON body with a `message`. Otherwise, it answers `202 Accepted` and verifies the mention asynchronously. It may answer `429` to senders that send too much. The sender learns nothing about the result: a mention that doesn't verify simply doesn't reach the inbox.

### Fetching the source

The receiver fetches `source` and decides by the response what it is:

- **A feed**: RSS 2.0 or JSON Feed. The receiver finds the item by the fragment, as RFC-4 does: if it isn't in the feed, it follows `next_url` (JSON Feed) or `<atom:link rel="next">` (RSS) on the same host. If the feed has exactly one item, the fragment may be left out.
- **An HTML page**: receivers should accept HTML sources. The post is the first [`h-entry`](https://microformats.org/wiki/h-entry) of the page, with its text in `e-content` and its title in `p-name`; without one, the post is the page's `body`, and its title the page's `title`.
- **Anything else**, an error status, or a feed without the item: the mention is invalid.

### Verifying a mention by name

The target is `https://<host>/`, and the post's text, found as in [Handles](#handles), contains `@<host>`.

### Verifying a reply

The target is a feed URL + `#id`, and two things must hold:

1. **The post answers the target.** In a feed item, `inReplyTo` equals the target. As in RFC-4, the RSS and JSON forms of the same feed count as the same target, and the fragment is compared after decoding. In an HTML page, the post contains a link to the target, or to the permalink of the target item (its `link` or `url`), preferably marked as `u-in-reply-to` or `rel="in-reply-to"`.
2. **The target is the owner's item.** One of these:
   - **On the owner's domain.** The target feed is on the owner's host, or is a feed the receiver knows the owner publishes, e.g. because the receiver is part of the owner's blog engine. The item exists in it.
   - **Republished for the owner.** The target feed announces RFC-4 `submissions`, and the item carries the owner's handle as its socialspec account. This is how replies to the owner's comments in groups and on other blogs arrive: those comments live in the server's comment feeds, not on the owner's domain. An RFC-4 server republishes an account only after checking that it belongs to the author feed's host. The receiver fetches the target feed to check this, or relies on a record of the owner's own submission, e.g. the `pubUrl` an RFC-4 status page returned.

### Delivering

A valid mention goes into the owner's inbox, usually the inbox feed of [RFC-12](https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md), with the category `mentions`. The receiver records at least:

| Field | Value |
| --- | --- |
| Kind | `mention` or `reply`. A post that answers the owner and also names them is a `reply`. |
| Source and target | As sent. |
| Author | The author's name, and their socialspec account. As in RFC-4, an account on the source's own host is shown as the author's; an account on another host is shown as stated by the publisher of the source, e.g. *Bob (@bob.example.net) in Two Waves*. For an HTML page, `p-author` of the `h-entry`, else the page's host. |
| Content | The post's title and text, or an excerpt, to show in the inbox. Readers open the source for the rest. |
| Dates | The post's publication date, and when the mention arrived. |

The receiver keeps one mention per source and target. A repeated webmention for the same pair updates it, or, if the post no longer verifies (the source answers `404` or `410`, the item is gone, the handle or the `inReplyTo` was removed), removes it. As in RFC-4, a receiver that gives up while following archive pages treats the item as not found, which removes nothing.

## CORS

A sender may run in the browser, such as an app that publishes the owner's feed through PUPS. Mention endpoints should therefore answer with `Access-Control-Allow-Origin: *`. A form-encoded `POST` needs no preflight; an endpoint that accepts JSON bodies also answers the `OPTIONS` preflight.

RFC-6 already serves `host-meta.json` with CORS. Other domains may not; a browser-based sender can't discover their endpoint and should leave mentions of them to a server, or skip them.

A browser-based sender may also send to an endpoint that doesn't answer with CORS headers. The form-encoded `POST` is a simple request, so a request in `no-cors` mode reaches the receiver, but the sender can't read the answer. It then treats the result as unknown and doesn't retry.

## Security considerations

**Anyone can send a mention.** The receiver trusts nothing in the request. It fetches the source and checks it, so a mention reaches the inbox only if a public post really names or answers the owner. A sender can't forge the author: the inbox shows what the source says, attributed as described in [Delivering](#delivering).

**Fetching untrusted URLs.** Both sides fetch URLs that strangers control: the receiver fetches sources, archive pages and target feeds; the sender fetches `host-meta.json` for every handle in its posts, and posts to whatever endpoint it names. Both fetch with size and time limits, refuse private and loopback addresses, follow few redirects, and never fetch their own endpoint.

**Amplification.** A post with many handles makes its sender contact many domains, and a single mention makes the receiver fetch a source. Senders limit the accounts per post, receivers verify one mention at a time per source host and limit how often they fetch from one host. Senders cache discovery, so a domain that is named often isn't flooded with requests for `host-meta.json`.

**Spam.** Mentions cost nothing to send, and anyone can publish a post that names a domain. Receivers may filter, rate-limit or require that the author is someone the owner follows. That is the inbox's business; this RFC only guarantees that every mention points to a real, public post.

**Republished items.** A reply to a republished comment is accepted on the word of the server that republished it: a feed that pretends to be an RFC-4 server could claim that an item is the owner's. The worst outcome is a reply in the inbox to an item that isn't the owner's, which the inbox shows with its real target. Receivers that want certainty accept only replies to items on the owner's domain, or to submissions they have a record of.

## Privacy considerations

- **Nothing private is sent.** The source is a public post, and the target is a public domain or item.
- **The fetches are visible.** The receiver fetches the source, so its host learns that the mention was checked. The sender fetches `host-meta.json`, so the mentioned domain learns that a post named it, even if it has no endpoint.
- **Replies to republished comments.** To check them, the receiver may fetch the group's comment feeds; the group's server sees that the owner's receiver follows the thread.

## Relation to other specs

- **Webmention.** This is a profile of Webmention. The request and the verification by fetching the source are the same. What differs: the endpoint is found in the mentioned domain's `host-meta.json`, not on the target page; the target is a domain or a feed item, not a page; the source may be a feed item. A domain may point its mention link to a generic Webmention receiver, which will accept mentions whose handle is linked to the domain's root (see [Writing handles](#handles)), and replies whose source links to the target.
- **RFC-4.** Both use the same request, for different purposes and at different endpoints. An RFC-4 webmention goes to the endpoint in the target feed's `submissions`, and asks to have a comment published. A mention goes to the endpoint in `host-meta.json`, and asks for nothing but attention: it never publishes anything. RFC-4 servers send mentions for what they republish.
- **ActivityPub and Mastodon.** Fediverse mentions name `user@host`, are found through WebFinger and delivered as signed activities to an inbox. Here the handle names a whole domain, the endpoint is in `host-meta.json`, and the receiver checks the post instead of a signature.
- **RFC-10.** Identity tokens prove that an app acts for a domain. A mention needs no such proof: the post itself is the proof, wherever it is published.

## Example: mentioned in a blog post

Alice publishes a post in her feed, through an app that writes it to `https://alice.example.com/feed/feed.000004.json` as an RFC-13 archive:

```json
{
  "id": "p-81",
  "url": "https://alice.example.com/2026/11/feeds-talk",
  "title": "Feeds are back",
  "content_html": "<p>Great talk by <a href=\"https://tim.example.com/\">@tim.example.com</a> yesterday.</p>",
  "date_published": "2026-11-19T09:00:00Z",
  "authors": [{ "name": "Alice" }]
}
```

The app finds the handle `@tim.example.com` and fetches `https://tim.example.com/.well-known/host-meta.json`:

```json
{
  "links": [
    { "rel": "https://bigwide.net/ns/mention", "href": "https://inbox.example.net/u/tim.example.com/mention" }
  ]
}
```

It sends:

```
POST /u/tim.example.com/mention HTTP/1.1
Host: inbox.example.net
Content-Type: application/x-www-form-urlencoded

source=https%3A%2F%2Falice.example.com%2Ffeed%2Ffeed.000004.json%23p-81&target=https%3A%2F%2Ftim.example.com%2F
```

The receiver answers `202 Accepted`, fetches the archive, finds `p-81`, finds `@tim.example.com` in its text, and adds a `mention` from Alice to Tim's inbox.

## Example: a reply in a discussion group

This continues the discussion group example of RFC-4. Alice's post in the group feed carries her account:

```json
{
  "id": "a8Kd03XpQz",
  "title": "Is RSS the better social network?",
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "accounts": { "socialspec": "@alice.example.com" }
  }
}
```

Bob submits a comment from his comment feed `https://bob.example.net/appdata/app.example/comments/b7b/8ec59ab09c61ab7b.json`, with `inReplyTo` set to Alice's post. His client sends the RFC-4 webmention to the group, and no mention. The group's server, `rm.example.org`, accepts the comment and republishes it in the post's comment feed, `https://rm.example.org/groups/twowaves/comments/a8Kd03XpQz/comments.json`:

```json
{
  "id": "a8Kd03XpQz-k2m9x",
  "content_html": "<p>Agreed, but discovery is hard.</p>",
  "authors": [{ "name": "Bob" }],
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "accounts": { "socialspec": "@bob.example.net" },
    "inReplyTo": "https://rm.example.org/groups/twowaves/posts.json#a8Kd03XpQz"
  }
}
```

Now the server sends the mention. The answered item is its own post `a8Kd03XpQz`, whose account is `@alice.example.com`. It discovers Alice's endpoint, `https://alice.example.com/mention`, and sends:

```
source=https://rm.example.org/groups/twowaves/comments/a8Kd03XpQz/comments.json#a8Kd03XpQz-k2m9x
target=https://rm.example.org/groups/twowaves/posts.json#a8Kd03XpQz
```

Alice's receiver:

1. Fetches the source and finds the comment. Its `inReplyTo` equals the target.
2. Sees that the target isn't on `alice.example.com`. It fetches the group feed, which announces `_submissions`, and finds the post `a8Kd03XpQz` with the account `@alice.example.com`.
3. Adds a `reply` to Alice's inbox: *Bob (@bob.example.net) in Two Waves RSS Discussion*, answering her post.

When Alice answers Bob, her client submits her comment to the group the same way, and the server sends a reply mention to `bob.example.net`, with Bob's comment as the target.

## Conformance

**Senders**

- Send mentions only for posts they publish; for RFC-4 submissions, only as the server that republishes them, after acceptance and moderation, with the republished item as `source`. Never send mentions for an item in an author feed submitted under RFC-4.
- Find handles as in [Handles](#handles), and reply authors as in [Replies](#replies). Notify each account at most once per post, never the post's own author, and at most about 20 accounts per post.
- Discover endpoints through `host-meta.json` only, over HTTPS, with caching; send nothing to domains without the link.
- Send `source` and `target` as defined, after the post is publicly visible, asynchronously, with retries for network errors, `5xx` and `429` only.
- Resend after edits and deletes to every account mentioned before or after.

**Receivers**

- Publish the endpoint in `host-meta.json` with the relation `https://bigwide.net/ns/mention`.
- Accept form-encoded webmentions, answer `400` for invalid requests and `202` otherwise, and send `Access-Control-Allow-Origin: *`.
- Verify asynchronously by fetching the source: the handle for mentions by name; `inReplyTo` equal to the target, and the target being the owner's item, for replies. Accept RSS 2.0 and JSON Feed sources, and should accept HTML.
- Keep one mention per source and target; update or remove it when a repeated webmention shows a change, a `404`, a `410` or a missing item.
- Fetch with limits, refuse private addresses, and limit the rate per source host.

## Open questions

- **The link relation.** `https://bigwide.net/ns/mention` is a placeholder. Should it be a registered relation, or should the relation `webmention` itself be allowed in `host-meta.json`?
- **Fallback to Webmention discovery.** Should senders, for domains without the link, fall back to standard Webmention discovery on `https://<host>/` or on the answered item's page? That would reach many existing sites, but notify pages, not people.
- **Permalinks as `inReplyTo`.** rss.chat and others reply to permalinks, not to feed items. Should senders find the author of an HTML page, e.g. through its `h-entry` or a `source:account` in the feed that contains it, and send reply mentions for those too?
- **Proving the sender.** Should a sender be able to attach an identity token (RFC-10), so that a receiver can filter by the sending domain before it fetches anything?
- **Mentions of groups.** Should a group handle, e.g. `@twowavesandadot.blog`, notify the group's moderators, or its members?
