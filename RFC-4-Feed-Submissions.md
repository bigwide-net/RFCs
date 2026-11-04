# RFC: Feed Submissions - Posting to Group Feeds with Webmention

Nov 4, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-4-Feed-Submissions.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-4-Feed-Submissions.md)

## Summary

This proposal lets a feed accept posts and comments from other feeds. A **group feed** works like a subreddit or a Facebook group, except that every post and comment is first published in its author's own RSS or JSON feed. The author then sends a [Webmention](https://www.w3.org/TR/webmention/) to the group, and the group republishes the item in its own feed.

The proposal adds two things to RSS (namespace `https://rssmention.bigwide.net/`, prefix `rm:`) and JSON Feed:

- **`submissions`**, on the feed: where to send the webmention, the terms of service, and the rules a submission must follow (title, length, allowed HTML, comment limits, editing).
- **`commenting`**, on an item: whether the item accepts comments.

Everything else comes from existing specs. Replies, comment feeds, counts, Markdown and accounts use the [source namespace](https://source.scripting.com/) in RSS and [RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md) in JSON Feed. The allowed HTML is a subset of [Feed HTML (RFC-2)](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md). Sending a submission is a plain webmention.

[rssmention](https://github.com/bigwide-net/rssmention) is the reference implementation.

**Why now.** Social-RSS systems like rss.chat already link posts and replies across feeds with `source:inReplyTo` and `source:comments`. A reply in one feed can point at a post in another, but nobody collects the replies in one place: the author of a post has to find the replies, and readers of a reply can't see the rest of the conversation. A group feed fills that gap without a new account system. The author's feed is their identity, the group feed is the shared space, and webmention is the notification. What is missing is a way for a client to learn, from the group feed alone, where to submit and which rules apply. Without it, a client fails at the first post that is too long or has no title, and the author learns why only from an error message.

## Roles

- **Group server**: publishes one or more group feeds, receives webmentions, checks them and republishes accepted items.
- **Author feed**: an RSS or JSON feed that the author controls. It contains the item to submit.
- **Client**: the software that writes the item into the author feed and sends the webmention. It can be a blog engine, a feed-based social app or a script.
- **Reader**: any feed reader. It shows group feeds like other feeds; the extensions in this proposal only matter to it if it offers a "reply" button.

## Overview

1. The client reads the group feed and finds `submissions`: the endpoint and the rules.
2. The client publishes an item in the author feed. The item's `inReplyTo` names the target: the group feed for a new post, or a post or comment for a reply.
3. The client sends a webmention with `source` = author feed URL + `#guid` and `target` = the same value as `inReplyTo`.
4. The group server answers `201 Created` with a status page, fetches the author feed, checks the item against the rules and republishes it under a new guid.
5. The republished post points to its comment feed (`source:comments`). Replies to it repeat steps 2–4 with the post as their target.

## Feed: `submissions`

`submissions` describes how to submit to the feed. It may appear in every feed of a group: the posts feed, its archives and the comment feeds. All of them carry the same rules, so a client can submit from whatever feed it is showing.

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

- `about` should be this document's URL. **This is not its final location**: the URL will change once the spec has a permanent home. Readers must therefore **ignore the value of `about`**: they recognize the extension by the name `_submissions` alone. A group server may also write the namespace URI (`https://rssmention.bigwide.net/`) there.
- Every field except `url` is optional. A reader ignores fields it doesn't know and values of the wrong type.

### Fields

| JSON `_submissions` | RSS `rm:submissions` | Meaning |
| --- | --- | --- |
| `url` | `<rm:url>` | **Required.** The webmention endpoint for posts and comments. Absolute `http(s)` URL. |
| `tos` | `<rm:tos>` | The terms of service. Clients should show the link before the first submission. |
| `policies.posts` | `<rm:policies type="posts">` | Rules for new posts. See [Policies](#policies). |
| `policies.comments` | `<rm:policies type="comments">` | Rules for comments and replies to comments. |
| `republish` | `<rm:republish>` | `true` if accepted submissions appear as items in the group's feeds. See the open questions. |

**Discovery.** The endpoint is also announced the standard webmention way, so generic webmention senders find it: `<atom:link rel="webmention" href="…"/>` in RSS and an HTTP `Link: <…>; rel="webmention"` header where the server can send one. `url` and the webmention link must name the same endpoint.

## Policies

A policy is a flat list of rules. In RSS each rule is an attribute of `<rm:policies>`, with booleans written as `true` or `false`. In JSON it is a property with a native JSON type. A missing rule means the server doesn't announce it, not that it is unlimited: the client should then expect the server to enforce a sensible default.

| Rule | Type | Meaning |
| --- | --- | --- |
| `title` | `required` \| `allowed` \| `none` | Whether the item must, may or must not have a title. Comments usually use `none`. |
| `minLength` | integer | Minimum length of the visible text in characters, after rendering and with whitespace collapsed. Markup doesn't count. |
| `maxLength` | integer | Maximum length of the content in characters: the Markdown if the item has `markdown`, otherwise the HTML. |
| `allowStyle` | boolean | `strong`, `em` |
| `allowLinks` | boolean | `a` with `href` |
| `allowCode` | boolean | `code`, `pre` |
| `allowQuote` | boolean | `q`, `blockquote` |
| `allowImages` | boolean | `img` with `src` |
| `allowTables` | boolean | `table`, `thead`, `tbody`, `tr`, `th`, `td` |
| `allowLists` | boolean | `ul`, `ol`, `li` |
| `allowHeadlines` | boolean | `h1`–`h6` |
| `urlAutoLinking` | boolean | The server turns bare URLs in the text into links, even if `allowLinks` is `false`. |
| `requireSocialSpec` | boolean | The item or its feed must have a `socialspec` account whose host is the author feed's host. See [Authorship](#authorship). |
| `allowEdit` | boolean | Re-sending the webmention updates the republished item. |
| `allowDelete` | boolean | Re-sending it after the item was removed from the author feed (or the feed answers 404/410) deletes the republished item. |
| `commentLimit` | integer | For posts: the maximum number of comments on all levels. For comments: the maximum number of direct replies to one comment. |
| `maxDepth` | integer | Comments only: how deep replies can be nested. Replies to a post have depth 1. |

**Allowed HTML.** `p` and `br` are always allowed. The `allow*` rules add the elements listed above, all of them from Feed HTML. Any other element makes the server reject the submission rather than silently drop it, so that the author is not surprised by what gets published. Attributes other than `a@href`, `img@src` and `img@alt` are removed. A server may add more limits, for example on the number of paragraphs or the length of link text. It should describe them in its terms of service.

**Markdown.** If the item has `markdown` (`source:markdown`, `_source.markdown`), the server renders it and ignores the HTML. Raw HTML inside the Markdown is escaped, not interpreted. The same element rules apply to the rendered result.

**Extending.** Servers may add rules. Clients ignore rules they don't know, and should then simply submit and let the server decide.

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

It is used in both directions:

- **In a group feed**, on posts: `false` means the post is closed, for example by a moderator. Clients should hide the reply button, and the server rejects comments with `400`. A missing `commenting` means the item accepts comments if the feed has `submissions`.
- **In an author feed**, on a submitted post: `false` asks the group server to publish the post with comments disabled. A missing value means comments are allowed.

`allowComments` must be a boolean in JSON and `true` or `false` in RSS. Readers ignore other values.

## Submitting

The client publishes the item in the author feed, then sends a webmention as the [Webmention spec](https://www.w3.org/TR/webmention/#sending-webmentions) describes: a `POST` to `url` with `source` and `target`, form-encoded. Servers should accept a JSON body with the same two fields as well.

```
POST /groups/twowaves/mention HTTP/1.1
Host: rm.example.org
Content-Type: application/x-www-form-urlencoded

source=https%3A%2F%2Falice.example.com%2Ffeed.xml%23post-2026-11-04&target=https%3A%2F%2Frm.example.org%2Fgroups%2Ftwowaves%2Fposts.xml
```

### Source

`source` is the author feed's URL plus `#` and the item's guid (RSS `<guid>`, JSON `id`), percent-encoded where needed. If the feed has exactly one item, the fragment may be left out. The server fetches the feed, not an HTML page, and finds the item by its guid.

### Target and `inReplyTo`

Items in a group feed are referenced by **the feed that contains them plus `#guid`**. This works for items that have no page of their own and makes the containing feed easy to find.

| Submission | `target` | Example |
| --- | --- | --- |
| New post | the group's posts feed | `https://rm.example.org/groups/twowaves/posts.xml` |
| Comment on a post | the posts feed + `#post-guid` | `…/posts.xml#a8Kd03XpQz` |
| Reply to a comment | the comment feed that contains the comment + `#comment-guid` | `…/comments/a8Kd03XpQz/comments.xml#a8Kd03XpQz-k2m9x` |

- The item in the author feed must have `inReplyTo` (`source:inReplyTo`, `_source.inReplyTo`) with the same value as `target`. This is the link from source to target that webmention requires, and it lets readers of the author feed show the reply in context.
- The RSS and JSON versions of a group feed (`posts.xml` and `posts.json`) count as the same target. The server compares them after decoding the fragment.
- A client finds the comment feed of a post or comment in its `comments.feedUrl` (`source:comments feedUrl`). Adding `#guid` gives the target for a reply.
- A post that is in an archive may also be targeted through the archive feed: `…/archive/posts-3.xml#a8Kd03XpQz`. New posts must target the posts feed.

### Authorship

Anyone can send a webmention for any feed. To show that the author controls the source, an item or its feed carries a `socialspec` account (`<source:account service="socialspec">@alice.example.com</source:account>`, `_source.accounts.socialspec`). When `requireSocialSpec` is set, its host must be the author feed's host name, `alice.example.com` for `https://alice.example.com/feed.xml`. Without `requireSocialSpec`, the server republishes the account only if it matches and leaves it out otherwise, so a group feed never shows an unverified account.

### What is republished

The server copies the title, the content (both HTML and, if given, Markdown), the author's name (`dc:creator`, `author`, `authors[0].name`), the item's `link`/`url` and the verified account. It assigns a new guid. The publication date is the item's own if that is close to the time of submission (rssmention allows 30 minutes) and the time of submission otherwise, so authors can't backdate or pin posts.

## Response and status

The server does the cheap checks synchronously: syntax, whether the target exists and accepts comments, limits and nesting depth. If one fails, it answers `400` and a JSON body with a `message`. If they pass, it answers `201 Created` with the status page in `Location`, and processes the submission asynchronously:

```
HTTP/1.1 201 Created
Location: https://rm.example.org/groups/twowaves/status/a8Kd03XpQz
Content-Type: application/json

{"status":"waiting","guid":"a8Kd03XpQz","statusUrl":"https://rm.example.org/groups/twowaves/status/a8Kd03XpQz"}
```

`guid` is the guid the item will have in the group feed. The status page returns one of these objects:

| `status` | Other fields | Meaning |
| --- | --- | --- |
| `waiting` | | Not processed yet. Poll again after a few seconds. |
| `success` | `pubDate`, `pubUrl` | Published. `pubUrl` is the containing feed + `#guid`. |
| `deleted` | | The resubmission deleted the item. |
| `failure` | `code`, `text` | Rejected. `text` is a human-readable reason to show the author. |

Status pages are short-lived: rssmention keeps them for at least an hour. Clients should not rely on them after that, and should read the group feed instead.

**Failure codes.** Clients can show `text` without knowing `code`. Codes let them react, for example by highlighting the title field:

| Code | Meaning |
| --- | --- |
| `fetch-failed` | The author feed could not be fetched or did not return 2xx. |
| `invalid-feed` | The author feed is neither RSS 2.0 nor JSON Feed. |
| `item-not-found`, `ambiguous-source` | No item with the guid, or several items and no `#guid`. |
| `in-reply-to-missing`, `in-reply-to-mismatch` | `inReplyTo` is missing or differs from `target`. |
| `socialspec-missing`, `socialspec-mismatch` | See [Authorship](#authorship). |
| `title-required`, `title-not-allowed`, `title-too-long` | The `title` rule. |
| `empty-content`, `too-short`, `too-long`, `malformed-html`, `forbidden-element`, `bad-nesting`, `missing-attribute`, `invalid-url`, `url-too-long`, `link-text-too-long`, `weird-formatting` | Content rules. Servers may define more content codes. |
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
| `accounts.socialspec` | `<source:account service="socialspec">` | Authorship of a submission; the group's own account on the feed. |

## Notes for group servers and clients

**Group servers**

- Put `submissions` in every feed of a group, with the rules that are actually enforced. A client that follows them should not be rejected.
- Treat the author feed as untrusted: fetch it with a size and time limit, refuse private network addresses, and never fetch your own URLs.
- Process submissions one at a time per group, or lock, so that comment limits and counts stay correct.
- Republished items carry `source:inReplyTo` (comments) and `source:comments` (posts and comments), so a reader can walk the thread without knowing this proposal.

**Clients**

- Read the rules before publishing: check the title, length and elements locally, and show the terms of service.
- Write `inReplyTo` exactly as the target, and keep the item in the author feed. If it drops out of the feed and the group allows deletes, the next webmention for it deletes the post.
- Poll the status page with a growing interval, and stop after a few minutes. A `waiting` status that never changes means the server restarted; resubmit.
- Prefer the `.json` or `.xml` URLs consistently, but don't depend on it: servers treat them as the same.

## Full example

The group feed `https://rm.example.org/groups/twowaves/posts.json` with one post:

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

Bob's author feed `https://bob.example.net/feed.json`, with a comment on that post:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Bob",
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "accounts": { "socialspec": "@bob.example.net" }
  },
  "items": [{
    "id": "c-17",
    "authors": [{ "name": "Bob" }],
    "date_published": "2026-11-04T10:20:00Z",
    "content_html": "<p>Agreed, but discovery is hard.</p>",
    "_source": {
      "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
      "markdown": "Agreed, but discovery is hard.",
      "inReplyTo": "https://rm.example.org/groups/twowaves/posts.json#a8Kd03XpQz"
    }
  }]
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
- **`republish`.** It is always `true` today. Is there a use for `false`, such as an endpoint that only notifies a person or collects submissions for a newsletter? If not, it can go.
- **Status pages.** The status object is not part of the Webmention spec, which leaves the status page open. Should this proposal standardize it, or only recommend it?
- **Stronger authorship.** A `socialspec` account on the feed's own host proves control of the host, not of one user on a shared host. Shared hosts (rss.chat, micro.blog) need something better, for example a token in the feed or a signature.
- **Feed HTML.** Should the `allow*` rules be replaced by one list of allowed elements, taken from RFC-2, so the rules can grow with Feed HTML?
- **Atom.** Author feeds in Atom could be supported with `thr:in-reply-to` (RFC 4685) instead of `inReplyTo`. Is that worth the extra parsing?
