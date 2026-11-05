# RFC: Feed Submissions - Comments and Discussions with Webmention

Nov 4, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-4-Feed-Submissions.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-4-Feed-Submissions.md)

## Summary

This proposal lets a feed accept comments, and optionally posts, from other feeds. The author first publishes the comment in their own RSS or JSON feed, then sends a [Webmention](https://www.w3.org/TR/webmention/) to the target. The server checks it and republishes it in a comment feed. It works for two kinds of targets:

- **A blog or any other feed with posts**: readers comment on its posts, and reply to each other's comments. The feed accepts comments only.
- **A discussion group**, like a subreddit or a Facebook group: the group feed accepts new posts as well as comments. Anyone can start a thread.

The proposal adds two things to RSS (namespace `https://rssmention.bigwide.net/`, prefix `rm:`) and JSON Feed:

- **`submissions`**, on the feed: where to send the webmention, the terms of service, and the rules a submission must follow (title, length, allowed HTML, comment limits, editing).
- **`commenting`**, on an item: whether the item accepts comments.

Everything else comes from existing specs. Replies, comment feeds, counts, Markdown and accounts use the [source namespace](https://source.scripting.com/) in RSS and [RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md) in JSON Feed. The allowed HTML is [Comment HTML (RFC-5)](https://github.com/bigwide-net/RFCs/blob/main/RFC-5-Comment-HTML.md). Sending a submission is a plain webmention.

[rssmention](https://github.com/bigwide-net/rssmention) is the reference implementation.

**Why now.** [rss.chat](https://rss.chat) shows that a feed post can carry a conversation: readers browse the comments on a post and reply to them, linked with `source:inReplyTo` and `source:comments`. But the current implementation relies on a central server, and users must register on that server before they can write a comment. This proposal allows commenting without registering anywhere. The author only needs to prove that they own their domain, and they do that by publishing the comment on it. Their domain is their identity, as established by [socialspec.org](https://socialspec.org); their feed is where they publish, and webmention is how the comment reaches the post. What is missing is a way for a client to learn, from the target feed alone, where to submit and which rules apply. Without it, a client fails at the first comment that is too long or uses a link where links are not allowed, and the author learns why only from an error message, or from finding that the comment was not published at all, or was published with features missing.

## Roles

- **Server**: receives webmentions for one or more feeds, checks them and republishes accepted items in comment feeds. For a blog it hosts only the comments, and the blog itself can live anywhere. For a discussion group it also publishes the group feed with the posts.
- **Target feed**: a feed that announces `submissions`: a blog's feed, or a group feed.
- **Author feed**: an RSS or JSON feed that the author controls. It contains the item to submit.
- **Client**: the software that writes the item into the author feed and sends the webmention. It can be a blog engine, a feed-based social app or a script.
- **Reader**: any feed reader. It shows target and comment feeds like other feeds; the extensions in this proposal only matter to it if it offers a "reply" button.

## Overview

1. The client reads the target feed and finds `submissions`: the endpoint and the rules.
2. The client publishes an item in the author feed. The item's `inReplyTo` names the target: a post in the target feed (a blog post or a group post), a comment, or, for a new post in a group, the group feed itself.
3. The client sends a webmention with `source` = author feed URL + `#guid` and `target` = the same value as `inReplyTo`.
4. The server answers `201 Created` with a status page, fetches the author feed, checks the item against the rules and republishes it under a new guid.
5. Every post and comment points to its comment feed (`source:comments`). Replies repeat steps 2–4 with that post or comment as their target.

## Feed: `submissions`

`submissions` describes how to submit to the feed. It may appear in the target feed and in every feed the server publishes for it: archives and comment feeds. All of them carry the same `submissions` block, so a client can submit from whatever feed it is showing. The block has separate policies for new posts and for comments, which may differ: the client applies the policy for what the author is creating, not for the feed it found the block in. See [Policies](#policies).

### RSS

```xml
<rss version="2.0" xmlns:rm="https://rssmention.bigwide.net/">
<channel>
  …
  <rm:submissions>
    <rm:url>https://rm.example.org/groups/twowaves/mention</rm:url>
    <rm:tos>https://twowavesandadot.blog/discussion/tos.html</rm:tos>
    <rm:policies type="posts" minLength="10" maxLength="8000" title="required" allowLinks="true" … commentLimit="1000"/>
    <rm:policies type="comments" minLength="10" maxLength="4000" title="none" allowLinks="true" … commentLimit="20" maxDepth="5"/>
    <rm:republish>true</rm:republish>
  </rm:submissions>
</channel>
</rss>
```

The namespace URI is `https://rssmention.bigwide.net/`. Readers should also accept `http://rssmention.bigwide.net/`, and match the namespace by URI, not by prefix.

### JSON Feed

`_submissions` follows JSON Feed's [extension rules](https://www.jsonfeed.org/version/1.1/#extensions-a-name-extensions-a) and RFC-1's conventions:

```json
"_submissions": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md",
  "url": "https://rm.example.org/groups/twowaves/mention",
  "tos": "https://twowavesandadot.blog/discussion/tos.html",
  "policies": {
    "posts": { "minLength": 10, "maxLength": 8000, "title": "required", "allowLinks": true, "commentLimit": 1000 },
    "comments": { "minLength": 10, "maxLength": 4000, "title": "none", "allowLinks": true, "commentLimit": 20, "maxDepth": 5 }
  },
  "republish": true
}
```

- `about` should be this document's URL. **This is not its final location**: the URL will change once the spec has a permanent home. Readers must therefore **ignore the value of `about`**: they recognize the extension by the name `_submissions` alone. A server may also write the namespace URI (`https://rssmention.bigwide.net/`) there.
- A reader ignores fields it doesn't know and values of the wrong type.

### Fields

| JSON `_submissions` | RSS `rm:submissions` | Meaning |
| --- | --- | --- |
| `url` | `<rm:url>` | **Required.** The webmention endpoint for posts and comments. Absolute `http(s)` URL. |
| `republish` | `<rm:republish>` | **Required.** Whether accepted submissions are published. See [Republishing](#republishing). |
| `tos` | `<rm:tos>` | The terms of service. Clients should show the link before the first submission. |
| `policies.posts` | `<rm:policies type="posts">` | Rules for new posts. Missing: the feed accepts no posts. See [Policies](#policies). |
| `policies.comments` | `<rm:policies type="comments">` | Rules for comments and replies to comments. Missing: the feed accepts no comments. |

**Posts, comments or both.** A blog announces only `comments`: readers comment on the blog's posts, but can't add posts to the blog. A discussion group announces both. A feed with `submissions` but neither policy accepts nothing; that only makes sense temporarily, for example while a group is closed.

**Discovery.** The endpoint is also announced the standard webmention way, so generic webmention senders find it: `<atom:link rel="webmention" href="…"/>` in RSS and an HTTP `Link: <…>; rel="webmention"` header where the server can send one. `url` and the webmention link must name the same endpoint.

### Republishing

`republish` says what happens to an accepted submission:

- **`true`**: the server publishes it in its feeds, next to the other comments, under the author's name.
- **`false`**: the server only delivers it to the owner of the target feed, like an email to the author of a blog. Nothing is published, and there are no comment feeds. Some blogs prefer this: they want to hear from their readers without hosting a public discussion.

`republish` is also a statement to the client. When it is `true`, sending the webmention means that the author agrees to the item being republished. The interface makes the consequence of calling it explicit: a client must make clear to the author, before the first submission, that the comment will be published, and show the terms of service along with it.

## Policies

A policy is a flat list of rules. In RSS each rule is an attribute of `<rm:policies>`, with booleans written as `true` or `false`. In JSON it is a property with a native JSON type.

A missing rule has the default value from the table below. Clients and servers must use these defaults; they never guess. Servers should still announce every rule they enforce.

| Rule | Type | Default | Meaning |
| --- | --- | --- | --- |
| `title` | `required` \| `allowed` \| `none` | `none` | Whether the item must, may or must not have a title. Comments usually use `none`; discussion groups usually require titles for posts. |
| `minLength` | integer | `2` | Minimum length of the visible text in characters. See [Content](#content). |
| `maxLength` | integer | `2000` | Maximum length of the content in characters. See [Content](#content). |
| `allowStyle`, `allowLinks`, `allowCode`, `allowQuote`, `allowImages`, `allowTables`, `allowLists`, `allowHeadlines`, `urlAutoLinking` | boolean | `false` | The Comment HTML features that are enabled. See [Content](#content). |
| `requireSocialSpec` | boolean | `false` | The item or its feed must have a `socialspec` account whose host is the author feed's host. See [Authorship](#authorship). |
| `allowEdit` | boolean | `false` | Re-sending the webmention updates the republished item. |
| `allowDelete` | boolean | `false` | Re-sending it after the item was removed from the author feed (or the feed answers 404/410) deletes the republished item. |
| `commentLimit` | integer | `1000` | For posts: the maximum number of comments on all levels. For comments: the maximum number of direct replies to one comment. |
| `maxDepth` | integer | `5` | Comments only: how deep replies can be nested. Replies to a post have depth 1. |

**Extending.** Servers may add rules. Clients ignore rules they don't know, and should then simply submit and let the server decide.

## Content

The content of a submission must be **Comment HTML**, defined in [RFC-5](https://github.com/bigwide-net/RFCs/blob/main/RFC-5-Comment-HTML.md). In short:

- Comment HTML is a small, strict subset of HTML. Paragraphs and line breaks are always allowed; the `allow*` rules above switch on optional features such as links, code, quotes or lists.
- Markdown (`source:markdown`, `_source.markdown`) is preferred over HTML, and plain text is the fallback.
- A submission that breaks a rule is rejected with a reason, not repaired. The client is responsible for keeping the author within the rules: it offers only the enabled features and enforces the limits.
- RFC-5 also defines how `minLength` and `maxLength` are measured, and how the server normalizes the HTML it republishes.

## Item: `commenting`

`commenting` says whether an item accepts comments.

```xml
<rm:commenting><rm:allowComments>false</rm:allowComments></rm:commenting>
```

```json
"_commenting": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md",
  "allowComments": false
}
```

**`allowComments` defaults to `true`.** If the feed has `submissions` with a `comments` policy, every item accepts comments unless it says `false`. Items don't need `commenting` to be open for comments.

It is used in both directions:

- **In a target feed**, on posts: `false` means the post is closed, for example by its author or a moderator. Clients should hide the reply button, and the server rejects comments with `400`.
- **In an author feed**, on a submitted post: `false` asks the server to publish the post with comments disabled.

`allowComments` must be a boolean in JSON and `true` or `false` in RSS. Readers ignore other values, and then use the default.

## Submitting

The client publishes the item in the author feed, then sends a webmention as the [Webmention spec](https://www.w3.org/TR/webmention/#sending-webmentions) describes: a `POST` to `url` with `source` and `target`, form-encoded. Servers should accept a JSON body with the same two fields as well.

```
POST /groups/twowaves/mention HTTP/1.1
Host: rm.example.org
Content-Type: application/x-www-form-urlencoded

source=https%3A%2F%2Fbob.example.net%2Ffeed.xml%23c-17&target=https%3A%2F%2Frm.example.org%2Fgroups%2Ftwowaves%2Fposts.xml%23a8Kd03XpQz
```

### Author feeds

Author feeds are **RSS 2.0 or JSON Feed**. Atom is not supported: two formats are enough. RSS because it is by far the most widespread, JSON Feed because it is the easiest to produce and to parse.

### Source

`source` is the author feed's URL plus `#` and the item's guid (RSS `<guid>`, JSON `id`), percent-encoded where needed. If the feed has exactly one item, the fragment may be left out. The server fetches the feed, not an HTML page, and finds the item by its guid.

### Target and `inReplyTo`

Posts and comments are referenced by **the feed that contains them plus `#guid`**. This works for items that have no page of their own and makes the containing feed easy to find.

| Submission | `target` | Example |
| --- | --- | --- |
| Comment on a blog post | the blog's feed + `#post-guid` | `https://alice.example.com/feed.xml#post-42` |
| Comment on a group post | the group feed + `#post-guid` | `https://rm.example.org/groups/twowaves/posts.xml#a8Kd03XpQz` |
| Reply to a comment | the comment feed that contains the comment + `#comment-guid` | `https://rm.example.org/…/comments/a8Kd03XpQz/comments.xml#a8Kd03XpQz-k2m9x` |
| New post in a group | the group feed, without fragment | `https://rm.example.org/groups/twowaves/posts.xml` |

- The item in the author feed must have `inReplyTo` (`source:inReplyTo`, `_source.inReplyTo`) with the same value as `target`. This is the link from source to target that webmention requires, and it lets readers of the author feed show the reply in context.
- If a feed is published as both RSS and JSON Feed, the two count as the same target. The server compares them after decoding the fragment.
- A client finds the comment feed of a post or comment in its `comments.feedUrl` (`source:comments feedUrl`). Adding `#guid` gives the target for a reply. For a blog, the server publishes the comment feeds, and the blog's posts point to them with `source:comments`.
- A post that is in an archive may also be targeted through the archive feed: `…/archive/posts-3.xml#a8Kd03XpQz`. New posts must target the group feed itself.

### Authorship

Anyone can send a webmention for any feed. To show that the author controls the source, an item or its feed carries a `socialspec` account (`<source:account service="socialspec">@alice.example.com</source:account>`, `_source.accounts.socialspec`). When `requireSocialSpec` is set, its host must be the author feed's host name, `alice.example.com` for `https://alice.example.com/feed.xml`. Without `requireSocialSpec`, the server republishes the account only if it matches and leaves it out otherwise, so a feed never shows an unverified account.

**Every author needs their own domain or subdomain.** As in [socialspec.org](https://socialspec.org), the domain is the author's identity. Shared hosts, where many users publish under one host name, can't prove authorship this way. This is deliberate: it is what allows servers not to identify users any further. Whoever gave the author a domain or subdomain has already identified them, and the server doesn't need to repeat that with accounts, passwords or e-mail addresses.

### What is republished

The server copies the title, the content (both HTML and, if given, Markdown), the author's name (`dc:creator`, `author`, `authors[0].name`), the item's `link`/`url` and the verified account. It assigns a new guid. The publication date is the item's own if that is close to the time of submission (rssmention allows 30 minutes) and the time of submission otherwise, so authors can't backdate or pin posts.

## Response and status

The server does the cheap checks synchronously: syntax, whether the target exists and accepts comments, limits and nesting depth. If one fails, it answers `400` and a JSON body with a `message`. If they pass, it answers `201 Created` with the status page in `Location`, and processes the submission asynchronously:

```
HTTP/1.1 201 Created
Location: https://rm.example.org/groups/twowaves/status/a8Kd03XpQz-k2m9x
Content-Type: application/json

{"status":"waiting","guid":"a8Kd03XpQz-k2m9x","statusUrl":"https://rm.example.org/groups/twowaves/status/a8Kd03XpQz-k2m9x"}
```

`guid` is the guid the item will have when it is republished. The status page returns one of these objects:

| `status` | Other fields | Meaning |
| --- | --- | --- |
| `waiting` | | Not processed yet. Poll again after a few seconds. |
| `success` | `pubDate`, `pubUrl` | Accepted. `pubUrl` is the containing feed + `#guid`; it is missing if `republish` is `false`. |
| `deleted` | | The resubmission deleted the item. |
| `failure` | `code`, `text` | Rejected. `text` is a human-readable reason to show the author. |

Status pages are short-lived: rssmention keeps them for at least an hour. Clients should not rely on them after that, and should read the comment feed instead.

**Failure codes.** Clients can show `text` without knowing `code`. Codes let them react, for example by highlighting the title field:

| Code | Meaning |
| --- | --- |
| `fetch-failed` | The author feed could not be fetched or did not return 2xx. |
| `invalid-feed` | The author feed is neither RSS 2.0 nor JSON Feed. |
| `item-not-found`, `ambiguous-source` | No item with the guid, or several items and no `#guid`. |
| `in-reply-to-missing`, `in-reply-to-mismatch` | `inReplyTo` is missing or differs from `target`. |
| `socialspec-missing`, `socialspec-mismatch` | See [Authorship](#authorship). |
| `title-required`, `title-not-allowed`, `title-too-long` | The `title` rule. |
| `empty-content`, `too-short`, `too-long`, `malformed-html`, `forbidden-element`, `bad-nesting`, `missing-attribute`, `invalid-url`, `url-too-long`, `link-text-too-long`, `weird-formatting` | Content rules, see RFC-5. Servers may define more content codes. |
| `comments-disabled`, `comment-limit`, `max-depth`, `target-gone` | The target no longer accepts the comment. |
| `edit-not-allowed` | The source was submitted before and `allowEdit` is not set. |
| `moderation-rejected`, `moderation-unavailable` | The server's moderation rejected the item, or could not run. |

## Edits and deletes

The server remembers which source became which guid. Sending a webmention for the same `source` again:

- **updates** the republished item if `allowEdit` is set. The target must stay the same; the item keeps its guid and publication date, and gets a `date_modified`.
- **deletes** it, together with all replies, if `allowDelete` is set and the author feed answers 404 or 410, or no longer contains the item.
- is rejected with `400` if neither is set.

## Mapping between RSS and JSON Feed

| JSON | RSS (`rm:` = `https://rssmention.bigwide.net/`) |
| --- | --- |
| feed `_submissions` | `<rm:submissions>` in the channel |
| `_submissions.url` | `<rm:url>` |
| `_submissions.tos` | `<rm:tos>` |
| `_submissions.policies.posts` | `<rm:policies type="posts" …/>`, one attribute per rule |
| `_submissions.policies.comments` | `<rm:policies type="comments" …/>` |
| `_submissions.republish` | `<rm:republish>` |
| item `_commenting.allowComments` | `<rm:commenting><rm:allowComments>` in the item |

Fields used from RFC-1 and the source namespace:

| JSON `_source` | RSS | Used for |
| --- | --- | --- |
| `inReplyTo` | `<source:inReplyTo>` | The target of a submission; the parent of a republished comment. |
| `comments.feedUrl`, `count` | `<source:comments feedUrl count>` | Where the replies to a post or comment are, and how many. |
| `comments.total` | — | Comments on all levels of a post. |
| `markdown` | `<source:markdown>` | The submitted text; republished alongside the HTML. |
| `accounts.socialspec` | `<source:account service="socialspec">` | Authorship of a submission; the target's own account on the feed. |

## Notes for servers and clients

**Servers**

- Put `submissions` in the target feed and every feed you publish for it, with the rules that are actually enforced. A client that follows them must not be rejected for breaking a rule.
- Treat the author feed as untrusted: fetch it with a size and time limit, refuse private network addresses, and never fetch your own URLs.
- Process submissions one at a time per target feed, or lock, so that comment limits and counts stay correct.
- Republished items carry `source:inReplyTo` (comments) and `source:comments` (posts and comments), so a reader can walk the thread without knowing this proposal.

**Clients**

- Read the rules before the author starts writing. Offer only the enabled features and enforce the limits in the editor (see RFC-5). Show the terms of service, and that the comment will be published if `republish` is `true`.
- Write `inReplyTo` exactly as the target, and keep the item in the author feed. If it drops out of the feed and the server allows deletes, the next webmention for it deletes the comment.
- Poll the status page with a growing interval, and stop after a few minutes. A `waiting` status that never changes means the server restarted; resubmit.

## Example: comments on a blog

Alice's blog feed `https://alice.example.com/feed.json` uses a comment server at `comments.example.org`. It accepts comments only, and announces it in `_submissions`. Her post points to its comment feed:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Alice's blog",
  "feed_url": "https://alice.example.com/feed.json",
  "_submissions": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md",
    "url": "https://comments.example.org/groups/alice/mention",
    "policies": {
      "comments": { "minLength": 2, "maxLength": 2000, "allowStyle": true, "urlAutoLinking": true, "title": "none", "requireSocialSpec": true }
    },
    "republish": true
  },
  "items": [{
    "id": "post-42",
    "url": "https://alice.example.com/2026/11/rss-social",
    "title": "Is RSS the better social network?",
    "content_html": "<p>I think feeds are all we need.</p>",
    "_source": {
      "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
      "comments": { "feedUrl": "https://comments.example.org/groups/alice/external/post-42/comments.json", "count": 0 }
    }
  }]
}
```

Bob publishes his comment in `https://bob.example.net/feed.json`, with `inReplyTo` set to the post:

```json
{
  "id": "c-17",
  "authors": [{ "name": "Bob" }],
  "content_html": "<p>Agreed, but discovery is hard.</p>",
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "inReplyTo": "https://alice.example.com/feed.json#post-42"
  }
}
```

and sends the webmention:

```
source=https://bob.example.net/feed.json#c-17
target=https://alice.example.com/feed.json#post-42
```

The comment then appears in `https://comments.example.org/groups/alice/external/post-42/comments.json`. With rssmention, this setup is a group with `externalFeeds: ["https://alice.example.com/feed.json"]`, and the blog copies `_submissions` from `https://comments.example.org/groups/alice/submissions.json`.

## Example: a discussion group

The group feed `https://rm.example.org/groups/twowaves/posts.json`, as rssmention publishes it, with one post:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Two Waves RSS Discussion",
  "description": "Discussions about RSS, podcasts, open social networks and similar technology",
  "feed_url": "https://rm.example.org/groups/twowaves/posts.json",
  "_submissions": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md",
    "url": "https://rm.example.org/groups/twowaves/mention",
    "tos": "https://twowavesandadot.blog/discussion/tos.html",
    "policies": {
      "posts": {
        "minLength": 10, "maxLength": 8000, "urlAutoLinking": true, "allowStyle": true, "allowLinks": true,
        "allowCode": true, "allowQuote": false, "allowImages": false, "allowTables": false,
        "allowLists": true, "allowHeadlines": true, "requireSocialSpec": true, "title": "required",
        "allowEdit": false, "allowDelete": false, "commentLimit": 1000
      },
      "comments": {
        "minLength": 10, "maxLength": 4000, "urlAutoLinking": true, "allowStyle": true, "allowLinks": true,
        "allowCode": true, "allowQuote": false, "allowImages": false, "allowTables": false,
        "allowLists": false, "allowHeadlines": false, "requireSocialSpec": true, "title": "none",
        "allowEdit": false, "allowDelete": false, "commentLimit": 20, "maxDepth": 5
      }
    },
    "republish": true
  },
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "accounts": { "socialspec": "@twowavesandadot.blog" }
  },
  "items": [{
    "id": "a8Kd03XpQz",
    "url": "https://alice.example.com/2026/11/rss-social",
    "title": "Is RSS the better social network?",
    "content_html": "<p>I think <strong>feeds</strong> are all we need.</p>",
    "date_published": "2026-11-04T10:00:00.000Z",
    "authors": [{ "name": "Alice" }],
    "_source": {
      "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
      "markdown": "I think **feeds** are all we need.",
      "accounts": { "socialspec": "@alice.example.com" },
      "comments": {
        "feedUrl": "https://rm.example.org/groups/twowaves/comments/a8Kd03XpQz/comments.json",
        "count": 3,
        "total": 7
      }
    },
    "_commenting": {
      "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md",
      "allowComments": true
    }
  }]
}
```

Bob's comment in his author feed `https://bob.example.net/feed.json`:

```json
{
  "id": "c-17",
  "authors": [{ "name": "Bob" }],
  "date_published": "2026-11-04T10:20:00Z",
  "content_html": "<p>Agreed, but discovery is hard.</p>",
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "markdown": "Agreed, but discovery is hard.",
    "inReplyTo": "https://rm.example.org/groups/twowaves/posts.json#a8Kd03XpQz"
  }
}
```

The webmention, and the comment as it appears afterwards in `…/comments/a8Kd03XpQz/comments.json`:

```
source=https://bob.example.net/feed.json#c-17
target=https://rm.example.org/groups/twowaves/posts.json#a8Kd03XpQz
```

```json
{
  "id": "a8Kd03XpQz-k2m9x",
  "content_html": "<p>Agreed, but discovery is hard.</p>",
  "date_published": "2026-11-04T10:20:00.000Z",
  "authors": [{ "name": "Bob" }],
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "markdown": "Agreed, but discovery is hard.",
    "accounts": { "socialspec": "@bob.example.net" },
    "inReplyTo": "https://rm.example.org/groups/twowaves/posts.json#a8Kd03XpQz",
    "comments": {
      "feedUrl": "https://rm.example.org/groups/twowaves/comments/a8Kd03XpQz/a8Kd03XpQz-k2m9x/comments.json",
      "count": 0
    }
  }
}
```

## Open questions

- **The namespace URI.** `https://rssmention.bigwide.net/` names the implementation, not the idea. Should the namespace be renamed (for example to `https://bigwide.net/ns/submissions/`) before other servers adopt it?
- **Status pages.** The status object is not part of the Webmention spec, which leaves the status page open. Should this proposal standardize it, or only recommend it?
