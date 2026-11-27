# RFC: Inbox Feeds - Mentions and Moderation Queues in the Reader

Nov 19, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-12-Inbox-Feeds.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-12-Inbox-Feeds.md)

## Summary

The **inbox feed** is a private feed of incoming things that the owner of a domain may want to see: people who mentioned or answered them ([RFC-11](https://github.com/bigwide-net/RFCs/blob/main/RFC-11-Mentions.md)), and comments on their feeds that wait for their approval ([RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md)).

It is a single JSON Feed at a fixed path on the owner's [Personal Upload Server (RFC-6)](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md), in its private area:

```
/pups/private/inbox.json
```

- **One writer**: the system that receives the owner's mentions and submissions, such as an [rssmention](https://github.com/bigwide-net/rssmention) installation, writes it.
- **Any number of readers**: the owner's feed readers and social apps read it with the scope `read:/pups/private/inbox.json`, and show it as the owner's inbox.
- **Protected by PUPS**: like every file in `/pups/private/`, it is never public, never cached, and readable only with a token whose scope covers it.

The feed has one extension, `_inbox`, which says what each item is: a mention, a reply or a submission. Items may carry [Feed Actions (RFC-9)](https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md), e.g. to approve or reject a comment, or to block its author. Which actions the writer offers is up to it. This RFC only defines where the reader finds the inbox, what it contains, and how the reader presents it.

**Why now.** RFC-11 delivers mentions to the owner's receiver, and an RFC-4 server collects comments that wait for moderation, but neither says how the owner gets to see them. Each server could build its own web page, and the owner would have to visit it. The reader already shows feeds, RFC-9 lets feeds carry buttons, and RFC-6 has a private area that only the owner's apps can read. What is missing is an agreement on where the inbox is, and what it looks like, so that any reader can show it.

## Goals and non-goals

**Goals**

- **Nothing to configure.** A reader that is connected to the owner's PUPS server finds the inbox at a fixed path.
- **Private, without new mechanisms.** The inbox is protected by the rules of RFC-6's private area, and read with an ordinary PUPS token.
- **An ordinary feed.** The inbox is a JSON Feed. A reader shows it with little more than its usual code.
- **Simple.** One file, one writer.

**Non-goals**

- **Several writers.** Exactly one system writes the inbox. An owner who wants mentions, comments and moderation queues from many systems in one inbox needs a system that collects them, e.g. a CMS, and lets it write the inbox.
- **Moderating groups on other servers.** A group hosted elsewhere, which the owner moderates, can't write the owner's inbox. Such servers can offer their own personal feeds ([RFC-10](https://github.com/bigwide-net/RFCs/blob/main/RFC-10-Identity-Tokens.md)).
- **How items get into the inbox.** That is RFC-11 for mentions and RFC-4 for submissions, and otherwise the writer's own business.
- **Which actions exist.** Approving, rejecting, blocking or marking as spam are RFC-9 actions that the writer defines as it likes.
- **Read state across devices.** Which items the owner has seen is kept by each reader.
- **Push notifications.** Readers poll, as they do for any feed.

## What is new and what is reused

**Used as they are.**

| Spec | Used for |
| --- | --- |
| RFC-6 private area, scopes and private reads | Storing the inbox and protecting it. |
| [JSON Feed 1.1](https://www.jsonfeed.org/version/1.1/) | The format of the inbox. |
| RFC-9 actions, with capability URLs or RFC-10 `identity` | Approving, rejecting, blocking. |
| RFC-1 `_source.accounts` | The socialspec account of an item's author. |

**New in this RFC.**

- **The private path `/pups/private/inbox.json`**, added to RFC-6's list of well-known paths. See [The inbox file](#the-inbox-file).
- **The `_inbox` extension** on the feed and its items. See [The feed](#the-feed).
- **The categories** `mentions` and `moderation`. See [Categories](#categories).
- **How readers present the inbox.** See [Presenting the inbox](#presenting-the-inbox).

## Roles

- **Owner**: the person who controls a domain, e.g. `tim.example.com`, and runs a PUPS server for it.
- **Writer**: the one system that receives things for the owner and writes them into the inbox: an RFC-11 receiver, an RFC-4 server that hosts the comments of the owner's blog or a group the owner runs, or one system that does all of this.
- **Reader**: the owner's feed reader or social app, connected to their PUPS server.

## Overview

1. **Grant.** The owner connects the writer to their PUPS server with the scope `write:/pups/private/inbox.json`, and their reader with `read:/pups/private/inbox.json`. A writer that is part of the PUPS installation itself needs no grant.
2. **Write.** When a mention arrives, or a comment needs approval, the writer adds an item to the inbox. When a mention is withdrawn or a comment decided, it removes it.
3. **Read.** The reader fetches `storage_base` + `pups/private/inbox.json` with its PUPS token, using conditional requests.
4. **Present.** It shows the items as the owner's inbox, grouped by category, with a count of new items.
5. **Act.** The owner opens a mention in its thread, or approves a comment with one of the item's actions.

## The inbox file

### The path

The inbox is the file `/pups/private/inbox.json` on the owner's PUPS server. This RFC adds it to RFC-6's [list of well-known paths](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#the-list-of-well-known-paths):

| Path | Contents | Defined by | Consent screen | Link relations | Properties |
| --- | --- | --- | --- | --- | --- |
| `/pups/private/inbox.json` | The owner's inbox: mentions, replies and comments awaiting approval | this RFC | your inbox | — | — |

A server that offers it declares it in `well_known_paths`, like every private path, without `url`, `rels`, `properties` or `link_properties`:

```json
{ "path": "/pups/private/inbox.json" }
```

All rules of RFC-6's [private area](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#private-areas) apply: the file has no public URL, is read only with a token whose scope covers it, is stored where nothing serves it without PUPS, and every response carries `Cache-Control: no-store` and `Vary: Authorization`. It is never linked from `host-meta.json`; readers don't need a link, because the path is fixed.

### Scopes

| Who | Scope | Consent screen |
| --- | --- | --- |
| The writer | `write:/pups/private/inbox.json` | "manage your private data", with step-up confirmation |
| Readers | `read:/pups/private/inbox.json` | "read your private data", with step-up confirmation |

A writer that is part of the PUPS installation, e.g. an RFC-11 receiver or an RFC-4 server built into it, writes the file directly and needs no grant.

### One writer

The inbox has exactly one writer. RFC-6 lets the owner grant the scope to several apps, so the file says who writes it, in `_inbox.writer`:

- **Creating.** A writer that finds no inbox creates it with `If-None-Match: *`, with its own `client_id` in `writer`. A writer that is part of the PUPS installation uses `pups`.
- **Writing.** It writes only an inbox whose `writer` is its own, always as a read-modify-write with `If-Match`, as RFC-6 describes.
- **Someone else's inbox.** If `writer` names another system, it doesn't write the file, and tells the owner that their inbox is managed by that system. The owner can delete the file to let the new writer take over.

### Size

The file must stay below the server's `max_upload_bytes`, and small enough to fetch every few minutes:

- **Mentions** are kept for a limited time, e.g. 30 days or the newest 200. The writer drops older ones. There are no archive pages.
- **Submissions** stay until they are decided, since the moderation queue must be complete. A writer whose queue grows too large closes submissions, or tells the owner, rather than drop pending ones.
- **Excerpts.** For mentions, an excerpt of the post is enough; the reader opens the source for the rest.

## The feed

The inbox is a JSON Feed 1.1 document. JSON Feed, because RFC-9 actions exist only there.

### Feed: `_inbox`

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Inbox",
  "_inbox": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md",
    "writer": "https://tim.example.com/rm/client.json",
    "categories": ["mentions", "moderation"]
  },
  "items": []
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `writer` | string | **Required.** The `client_id` of the writer, or `pups` for a writer that is part of the PUPS installation. See [One writer](#one-writer). |
| `categories` | array of strings | **Required.** The categories that the writer fills, even when they have no items at the moment. Readers show a section for each, e.g. *To approve (0)*. See [Categories](#categories). |

`about` should be this document's URL. As with the other extensions, readers recognize `_inbox` by its name alone and ignore the value of `about`, which will change.

`title` is for readers that show the file as an ordinary feed. `home_page_url` may point to the writer's own web interface. The feed has no `feed_url`, since it has no public URL, and no `next_url`.

### Items

Every item is one incoming thing. It has the usual JSON Feed fields, and an `_inbox` object:

```json
{
  "id": "m-5Kd82",
  "url": "https://alice.example.com/2026/11/feeds-talk",
  "title": "Feeds are back",
  "content_html": "<p>Great talk by @tim.example.com yesterday.</p>",
  "date_published": "2026-11-19T09:00:12Z",
  "authors": [{ "name": "Alice" }],
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "accounts": { "socialspec": "@alice.example.com" }
  },
  "_inbox": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md",
    "type": "mention",
    "source": "https://alice.example.com/feed/feed.000004.json#p-81",
    "target": "https://tim.example.com/",
    "postDate": "2026-11-19T09:00:00Z"
  }
}
```

| Field | Meaning |
| --- | --- |
| `id` | Stable for the item, so that readers can track what the owner has seen. |
| `url` | The web page of the post, if there is one. Readers open it when they can't show the post themselves. |
| `title`, `content_html` | The post's title and text, or an excerpt of it. For `submission`, the full text, since the owner must see what they approve. Like any feed content, it is untrusted and shown with the subset of [RFC-2](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md). |
| `date_published` | When the item arrived in the inbox. Readers sort by it. |
| `authors[0].name`, `_source.accounts.socialspec` | The author of the post and their account, as the writer verified it. RFC-11 describes how to attribute an account on another host than the post's. |
| `_inbox.type` | **Required.** What the item is: `mention`, `reply` or `submission`. See [Categories](#categories). |
| `_inbox.source` | **Required.** The post: the `source` of the RFC-11 mention or of the RFC-4 submission, e.g. a feed URL + `#id`. |
| `_inbox.target` | **Required.** What it mentions, answers or was submitted to: the `target` of the mention or submission. |
| `_inbox.postDate` | The post's own publication date, if known. |

Readers ignore fields they don't know and values of the wrong type. Items with a `type` they don't know are shown like ordinary feed items.

### Categories

| Category | Item types | Contents | Filled by |
| --- | --- | --- | --- |
| `mentions` | `mention`, `reply` | Posts that name the owner by their handle, or answer one of their posts or comments. | An RFC-11 receiver. |
| `moderation` | `submission` | Comments and posts submitted to a feed or group of the owner, which wait for the owner's decision. | An RFC-4 server with moderation. |

**`mentions`.** One item per mention that the receiver accepted. When RFC-11 removes a mention, because the post was deleted or no longer mentions the owner, the writer removes the item.

**`moderation`.** One item per pending submission. `target` names the post, comment or group it was submitted to, so that the reader can show what it answers. The `submission` items are **the complete queue**: every pending submission is in the inbox, and nothing else. When a submission is approved or rejected, through the inbox or any other way, the writer removes it. A reader therefore removes submissions from its view that are no longer in the file.

**Published without review.** An RFC-4 server that publishes a submission without the owner's decision, e.g. after automatic moderation, may tell the owner with a `reply` item in `mentions`, with the `source` and `target` of the submission. It needs no decision, so it is not part of the queue, and readers don't count it as such. It may carry actions, e.g. to reject the submission after all, which removes it from the site again.

**More categories.** Writers may define further categories and item types, named with a URI if they are not defined by an RFC. Readers show items of unknown categories in a general section of the inbox, or not at all.

### Actions

Items, and the feed, may carry [Feed Actions (RFC-9)](https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md). Typical ones:

- On a `submission`: approve, reject, reject with a reason, approve and close the thread.
- On a `reply` that was published without review: reject, which removes it from the site again.
- On any item: block the author, report spam, remove the item from the inbox.
- On the feed: pause submissions, mark all as seen.

Which actions exist, and what they do, is the writer's choice. Action URLs point to the writer, not to the PUPS server, so the reader's PUPS token isn't sent with them. Two ways of authorizing them fit:

- **Capability URLs**, as in RFC-9: the writer puts a secret key into the action URLs, e.g. `https://tim.example.com/rm/approve/{id}?key=9c1f…`. The inbox is private, so only the owner's apps can see the key. This is the simplest choice.
- **Identity tokens**: controls with `"identity": true`, as in RFC-10, for writers that prefer to check who acts.

An action on a `submission` usually ends with the item leaving the inbox. The writer removes it from the file before it answers, and answers with new controls, e.g. a display *Approved by you* instead of the buttons, so that the owner sees the result at once. On its next fetch, the reader no longer finds the item and removes it.

A writer without actions still has a useful inbox. For moderation it then links to its own web interface, in `home_page_url` or in each item's `url`.

## Reading the inbox

1. **Find the PUPS server.** The reader discovers it from the owner's domain, as RFC-6 describes, and reads its configuration document.
2. **Check for an inbox.** If `well_known_paths` doesn't declare `/pups/private/inbox.json`, the server offers no inbox, and the reader shows none.
3. **Ask for the scope.** The reader requests `read:/pups/private/inbox.json`, with the scopes it needs anyway. The consent screen shows it as reading the owner's private data.
4. **Fetch.** It reads `storage_base` + `pups/private/inbox.json` with `Authorization: Bearer <access token>`, as an RFC-6 private read. `404` means that there is no inbox yet: nothing has arrived, or no writer is connected. `403` means that the token lacks the scope.
5. **Poll.** While the reader is open, it may fetch the inbox every few minutes, otherwise as often as other feeds. It sends `If-None-Match` with the last `ETag`, and the server answers `304` when nothing changed.

The reader sends the token only to `storage_base`, as RFC-6 requires for access tokens, never to the action URLs or the links in the items.

## Presenting the inbox

**A place of its own.** The reader shows the inbox apart from the feeds the owner subscribed to: a separate section or tab, with a count of new items. It doesn't ask the owner to subscribe to it.

**Grouped by category.** It shows a section for each category in `_inbox.categories`, e.g. *Mentions* and *To approve*, with the items sorted by `date_published`, newest first.

**What each item shows.**

- **`mention`**: the author, the post's title and text, and *mentioned you*.
- **`reply`**: the author, the reply, and the item that it answers, e.g. *replied to your comment "Agreed, but discovery is hard."*. The reader may fetch `target`, a feed URL + `#id`, to show it.
- **`submission`**: the author, the full text, where it was submitted (`target`), and the item's actions.

**Opening an item.** A reader that can show feed items in their thread opens `source`, which for RFC-11 and RFC-4 is a feed URL + `#id`, and shows the post among its comments, where the owner can answer it. Otherwise it opens `url`. A pending submission has no thread yet; the reader shows it from the inbox.

**New items.** The reader remembers which items the owner has seen, by `id`, and counts the others. For `moderation`, the count is the size of the queue: it shows what still needs a decision, not what is new.

**Several domains.** A reader that is connected to several domains shows the inbox of each, labelled with its handle, or one merged inbox with the handle on each item.

## Security considerations

**Who can write.** Whoever holds `write:/pups/private/inbox.json` can put anything into the inbox, including items that impersonate mentions, and actions that point anywhere. The owner grants it to one system they trust, with step-up confirmation. The `writer` field keeps a second app from overwriting the inbox by mistake, not on purpose: an app that holds the scope can do anything with the file.

**Who can read.** Every app with `read:/pups/private/inbox.json` sees all mentions, pending comments, and the capability URLs of the actions, and can therefore act on them. Owners grant it to their readers only.

**Items are untrusted.** Inbox items quote posts by strangers, and moderation items quote posts nobody has approved. Readers show them with the same care as any feed content, and never run their scripts or load their content with the owner's credentials.

**Actions.** The usual RFC-9 rules apply: no cookies, the reader's own words for anything done as the owner, and identity tokens only where RFC-10 allows them.

## Privacy considerations

- **Nothing is public.** The inbox has no public URL, and nothing in `host-meta.json` or elsewhere tells others which systems feed it.
- **Not encrypted.** As RFC-6 says of every private file, whoever runs the server or the hosting can read the inbox.
- **Reading is visible to the server.** The PUPS server sees when, and through which app, the inbox is read.

## Relation to other specs

- **RFC-11** delivers mentions to a receiver; the inbox is where the receiver puts them, and how the owner sees them.
- **RFC-4** submissions that wait for moderation appear here, if the RFC-4 server is the inbox's writer.
- **RFC-6** reserves `/pups/private/` for private files that extensions define, and names an inbox as an example. This RFC is that extension.
- **RFC-10** describes personal feeds on RFC-4 servers, such as replies and moderation queues, read with identity tokens. They are the way to follow servers that can't write the owner's inbox. The inbox keeps what arrives for the owner on the owner's own server.
- **ActivityPub** has an inbox for each actor too, but it is the endpoint where others deliver, not something the owner reads. Here, delivering is RFC-11, and the inbox feed is what the owner's reader reads.

## Example

Tim runs rssmention for his domain `tim.example.com`. It receives his mentions (RFC-11) and the comments on his blog (RFC-4), which he approves before they appear. He connected it to his PUPS server, whose `storage_base` is `https://tim.example.com/files/`, with the scope `write:/pups/private/inbox.json`. His reader holds `read:/pups/private/inbox.json` and fetches:

```http
GET /files/pups/private/inbox.json HTTP/1.1
Host: tim.example.com
Authorization: Bearer eyJhbGciOi…
If-None-Match: "a91f-3"
```

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Inbox",
  "home_page_url": "https://tim.example.com/rm/",
  "_inbox": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md",
    "writer": "https://tim.example.com/rm/client.json",
    "categories": ["mentions", "moderation"]
  },
  "_actions": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md",
    "templates": {
      "approve": { "type": "button", "label": "Approve", "style": "major", "action": "https://tim.example.com/rm/approve/{id}?key=9c1f4e07b2" },
      "reject": { "type": "button", "label": "Reject", "action": "https://tim.example.com/rm/reject/{id}?key=9c1f4e07b2" }
    }
  },
  "items": [
    {
      "id": "s-Hq72Lm",
      "content_html": "<p>Nice write-up. Does this work with Atom too?</p>",
      "date_published": "2026-11-19T10:02:00Z",
      "authors": [{ "name": "Carol" }],
      "_source": {
        "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
        "accounts": { "socialspec": "@carol.example.org" }
      },
      "_inbox": {
        "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md",
        "type": "submission",
        "source": "https://carol.example.org/comments.json#q-3",
        "target": "https://tim.example.com/feed.json#p-12"
      },
      "_actions": {
        "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md",
        "actions": [{ "template": "approve" }, { "template": "reject" }]
      }
    },
    {
      "id": "r-8Wq1Zt",
      "content_html": "<p>Agreed, but discovery is hard.</p>",
      "date_published": "2026-11-19T09:40:05Z",
      "authors": [{ "name": "Bob" }],
      "_source": {
        "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
        "accounts": { "socialspec": "@bob.example.net" }
      },
      "_inbox": {
        "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md",
        "type": "reply",
        "source": "https://rm.example.org/groups/twowaves/comments/a8Kd03XpQz/a8Kd03XpQz-k9t2c/comments.json#a8Kd03XpQz-r4w8n",
        "target": "https://rm.example.org/groups/twowaves/comments/a8Kd03XpQz/comments.json#a8Kd03XpQz-k9t2c",
        "postDate": "2026-11-19T09:40:00Z"
      }
    },
    {
      "id": "m-5Kd82",
      "url": "https://alice.example.com/2026/11/feeds-talk",
      "title": "Feeds are back",
      "content_html": "<p>Great talk by @tim.example.com yesterday.</p>",
      "date_published": "2026-11-19T09:00:12Z",
      "authors": [{ "name": "Alice" }],
      "_source": {
        "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
        "accounts": { "socialspec": "@alice.example.com" }
      },
      "_inbox": {
        "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md",
        "type": "mention",
        "source": "https://alice.example.com/feed/feed.000004.json#p-81",
        "target": "https://tim.example.com/",
        "postDate": "2026-11-19T09:00:00Z"
      }
    }
  ]
}
```

The reader shows Tim *To approve (1)*: Carol's comment on his post `p-12`, with *Approve* and *Reject*. Under *Mentions*, it shows Bob's reply to Tim's comment in the Two Waves group, which opens in its thread, and Alice's post that names him. When Tim approves Carol's comment, rssmention publishes it, removes it from the inbox, answers with a display *Approved*, and sends an RFC-11 mention to Carol's domain if her comment names anyone. On its next fetch, the reader no longer finds the item.

## Conformance

**PUPS servers**

- Declare `/pups/private/inbox.json` in `well_known_paths` to offer an inbox, and serve it under the rules of the private area.

**Writers**

- Write only an inbox that they created or whose `_inbox.writer` names them, with `If-None-Match: *` or `If-Match`.
- Write `_inbox.writer` and `_inbox.categories` on the feed, and `_inbox.type`, `source` and `target` on every item, with stable `id`s.
- Keep the `submission` items complete; remove items once a submission is decided or a mention withdrawn; keep the file small.
- Point actions to themselves, authorized by capability URLs or RFC-10 identity. Never pass on the actions of a received post, its `_actions` or its [RFC-9](https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md) fallback HTML (`class="without-extension-actions"`): in the inbox, the reader can't tell them from the writer's own.

**Readers**

- Read `storage_base` + `pups/private/inbox.json` with a token with `read:/pups/private/inbox.json`, with conditional requests, and send the token nowhere else.
- Show the inbox apart from the owner's subscriptions, grouped by category; treat the `submission` items as the complete queue; show actions under the rules of RFC-9 and RFC-10.

## Open questions

- **Messages.** RFC-4 submissions to a feed with `republish: no` are delivered to the owner and never published. Should they be a category of their own, e.g. `messages`?
- **The author's view.** RFC-10 suggests personal feeds that show authors the state of their own submissions. Should that be a category, e.g. `submissions`, in the author's inbox?
- **Read state.** Should readers be able to share which items were seen, e.g. through an action on the feed or a file in `/pups/private/`?
- **Owners without PUPS.** A domain without a PUPS server has no inbox. Should a writer be able to publish the inbox elsewhere, e.g. as an RFC-10 personal feed on its own server?
