# RFC: JSON Feed Archives - Keeping Every Item of a Feed

Nov 22, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-13-JSON-Feed-Archives.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-13-JSON-Feed-Archives.md)

## Summary

This proposal defines how a [JSON Feed](https://www.jsonfeed.org/version/1.1/) keeps **every item it ever published**, in archive files that never lose an item and never move one, so that each item has an address that lasts: the URL of its archive plus `#` and its id. It adds one extension, `_archive`, and uses JSON Feed's own `next_url` for the chain of archives.

In short:

- The **feed file** is the feed readers subscribe to. It holds copies of the newest items, and names the newest archive in `_archive.archive_url`.
- The **archives** hold every item of the feed, **each in exactly one archive**, newest first. They are JSON Feeds too, linked from the newest to the oldest with `next_url`, and each names the feed file in `_archive.updates_url`.
- **Items never move.** An item stays in the archive it was first added to, so `https://tim.example.com/feed/feed.000002.json#p-81` names it for as long as it exists.
- **Archives have numbers** from 0, the oldest, and a fixed maximum size. With `_archive.archive_template`, a reader can compute the URL of any archive without walking the chain.
- Every item has an **index**, `_archive.index`: 0 for the first item, then counting up, never changed and never used again. Readers can number the items, page through them and link to them without reading every archive.
- Archives change only when an item in them is **edited or deleted**. They are never rebalanced and never deleted.
- Readers that don't know `_archive` see an ordinary feed with the newest items, and, if they follow `next_url`, all the others.

It has three audiences:

- **Publishers**: CMSs, static site generators, and servers that keep feeds for apps, such as a Personal Upload Server ([RFC-7](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md)).
- **Readers**: feed readers that show a feed's history, open a post from a link, or number posts.
- **Other specs** that reference feed items by URL: [RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) comments, [RFC-11](https://github.com/bigwide-net/RFCs/blob/main/RFC-11-Mentions.md) mentions, post pages of a site.

**Why now.** RSS and JSON Feed were made to syndicate websites. The website was the original; the feed carried its newest posts, and older ones simply fell off the end, because the site still had them. In the Big Wide network, it is the other way round: the feed is the original, written by apps on the owner's domain, and a web page is only one way to show it. When an item falls off the end of such a feed, it is gone, and with it every comment that answers it, every mention that points to it, and every page that shows it. Feeds therefore need what websites always had: a permanent place for every post. Pagination with `next_url` alone doesn't give that, because the pages shift whenever a new item arrives, and nothing tells a reader whether it has reached the end of a feed's history or only the end of what the publisher keeps.

## Goals and non-goals

**Goals**

- Every item of a feed stays available, at a URL that never changes.
- The feed file stays small enough to poll often.
- Readers can show the full history, find a single item, and number items, each with as few requests as possible.
- Readers that don't know this extension lose nothing.
- Static files are enough: no server logic, no query parameters.
- Several writers can share a feed, given a way to make conditional writes, such as ETags.

**Non-goals**

- **RSS and Atom.** They have [RFC 5005](https://www.rfc-editor.org/rfc/rfc5005) (Feed Paging and Archiving) already, see [Relation to RFC 5005](#relation-to-rfc-5005). This RFC can be extended to RSS if there is demand.
- **How archives are written** in a particular system. This document describes the files and the rules that keep them consistent; [RFC-7](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md) describes how a PUPS server keeps them for the apps that post to a feed.
- **Searching.** Archives are pages in a chain, not an index of words or dates.
- **Deleting history.** A publisher can delete items, but not hide that an archive existed: archives are never removed from the chain.

## Relation to RFC 5005

[RFC 5005](https://www.rfc-editor.org/rfc/rfc5005) defines *archived feeds* for Atom (and, through `atom:link`, for RSS): a subscription document with the newest entries, and archive documents linked with `prev-archive`, each of which links back to the subscription document with `current`. This RFC is the same idea for JSON Feed, and adds what the Big Wide network needs on top:

| RFC 5005 | This RFC |
| --- | --- |
| Subscription document | Feed file |
| `prev-archive` (the next older archive) | `next_url`, as JSON Feed already pages with it |
| `current` (the subscription document) | `_archive.updates_url` |
| Link from the subscription document to the archives | `_archive.archive_url` |
| `next-archive` (the next newer archive) | Not used: the archive template and the numbers serve the same purpose without writing older archives again |
| Archives should not change | Archives change only when an item in them is edited or deleted, and items never move between them |
| — | Item addresses that last: archive URL + `#id` |
| — | Archive numbers and an archive template, for direct access |
| — | Item indexes |

## Terms

- **Feed file**: the JSON Feed that readers subscribe to and poll, e.g. `https://tim.example.com/feed.json`.
- **Archive**: a JSON Feed that holds a part of the feed's items, permanently. Every archive has a **number**, from 0 for the oldest.
- **Newest archive**: the archive with the highest number. New items go into it.
- **Item address**: the URL of the archive that holds an item, plus `#` and its id.
- **Index**: an item's number in the order in which items were added to the feed.
- **Writer**: whatever writes the files: a CMS, a static site generator, an app.

## The `_archive` object

Every file of an archived feed carries an `_archive` object, following JSON Feed's [extension rules](https://www.jsonfeed.org/version/1.1/#extensions-a-name-extensions-a), and so does every item.

In the feed file:

```json
"_archive": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md",
  "archive_url": "https://tim.example.com/feed/feed.000002.json",
  "archive_template": "https://tim.example.com/feed/feed.{number}.json",
  "archive_size": 20
}
```

In an archive (only the newest one has `next_index`):

```json
"_archive": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md",
  "updates_url": "https://tim.example.com/feed.json",
  "number": 2,
  "next_index": 42
}
```

In an item:

```json
"_archive": { "index": 41 }
```

| Field | Where | Meaning |
| --- | --- | --- |
| `archive_url` | Feed file | **Required** in a feed with archives: the URL of the newest archive. A feed file without it has no archives. |
| `archive_template` | Feed file | Optional: the URL of every archive, with `{number}` for its number, written in decimal with at least six digits (`000002`, `1000000`). See [Numbers](#numbers-and-the-template). |
| `archive_size` | Feed file | Optional: the most items an archive holds. Default `20`. |
| `updates_url` | Archive | **Required**: the URL of the feed file. |
| `number` | Archive | **Required**: the archive's number, from 0. |
| `next_index` | Newest archive, or the feed file of a feed without archives | The index the next new item gets. See [Indexes](#indexes). Other files may carry an outdated one, which everybody ignores. |
| `index` | Item | The item's index. |
| `about` | Feed file, archive | Should be this document's URL. Readers recognize `_archive` by its name and ignore the value of `about`, which will change once the spec has a permanent home. An item's `_archive` has no `about`. |

All URLs are absolute. Writers keep fields of `_archive` that they don't know, and readers ignore them, along with values of the wrong type.

A feed may number its items without keeping archives (yet): its feed file then has `_archive` with `next_index` and no `archive_url`, and its items have indexes. Everything about [indexes](#indexes) applies to it, but its items don't last, and their address is the feed file's URL + `#id` only as long as they are in it.

## Archives

The archives hold **every item of the feed, each in exactly one archive**. Together with the feed file, they form a chain:

```
feed.json  ⇢  feed.000002.json  →  feed.000001.json  →  feed.000000.json
     _archive.archive_url      next_url          next_url
```

- **Order.** Each archive lists its items newest first, like any feed. Every item of an archive is newer than every item of the archives with lower numbers.
- **Size.** An archive holds at most `archive_size` items, 20 by default. New items always go into the newest archive; when it is full, the next item starts a new archive with the next number. Older archives change only when an item in them is edited or deleted.
- **Items never move.** Once an item is in an archive, it stays there until it is deleted. Its address therefore never changes.
- **Links.** `next_url` of each archive points to the archive with the next lower number. The oldest archive has no `next_url`, unless the chain continues into older pages that are not archives, e.g. the pages of a feed from before it had archives (see [Starting the archives](#starting-the-archives)).
- **Never rebalanced, never deleted.** An archive that loses items keeps the rest, and no items move into it from other archives. In the worst case, an archive ends up empty; it keeps its place, its number and its URL, and it is never deleted. If the newest archive is emptied, the next item goes into it as usual.
- **Never empty at the start.** A new archive is created with the item that doesn't fit into the one before. A feed with no items has no archives.

### Numbers and the template

Archive numbers start at 0 for the oldest archive and go up by one for each new archive, without gaps. The newest archive has the highest number.

`archive_template`, if given, is the URL of every archive, with `{number}` in place of its number. The number is written in decimal, padded with zeros to six digits, and longer once it needs more: `https://tim.example.com/feed/feed.{number}.json` gives `…/feed.000000.json`, `…/feed.000017.json`, and so on. The template contains `{number}` exactly once and nothing else in braces. Every archive's URL, including `archive_url`, must match it.

The template lets a reader go directly to an archive, e.g. page 3 of the history or the archive of a given index, instead of walking the chain from the newest. A writer that can't give archives such URLs leaves it out; readers then walk the chain.

`archive_size` doesn't change once the feed has archives: readers use it to find items by index.

### Feed fields of an archive

An archive is a complete JSON Feed. A new archive gets a copy of the top-level fields of the feed file, with these differences:

- `feed_url` is the archive's own URL.
- `next_url` points to the next older archive.
- `_archive` has `updates_url`, `number` and, in the newest archive, `next_index`, instead of the feed file's fields.
- `items` holds the archive's items.
- Fields that describe the live feed rather than its items are left out: `hubs`, RFC-4 `_submissions`, RFC-3 `_podcast.liveItems`. What an archive would say about them would only be true as long as it was not outdated.
- `user_comment` may say that the file is an archive, e.g.: *This is an archive of https://tim.example.com/feed.json. New items appear in the feed at _archive.updates_url, older ones are in the archive at next_url.*

Archives keep the fields they were created with. A writer may update them, but doesn't have to: readers take the title, icon, authors and other settings of the feed from the feed file.

## The feed file

The feed file is the feed readers subscribe to. In a feed with archives:

- It holds **copies of the newest items**, newest first, as many as the writer likes. It should hold at least as many as an archive (`archive_size`), so that a reader that polls it sees every item that the newest archive gained in the meantime.
- Its items are the same as in their archives, with the same `id`, `_archive.index` and content. When an item is edited, both copies change; if they differ for a moment, because a write was interrupted, the **archive's copy** is the item.
- It has `_archive.archive_url`, and doesn't carry `next_index`: the counter is in the newest archive.
- It may also have `next_url`, equal to `archive_url`, for readers that page through feeds with `next_url` but don't know `_archive`. Those readers then see the newest items twice, once in the feed file and once in the newest archive, and must tell them apart by `id`, as JSON Feed readers do anyway.

## Items

- **`id`** is unique across the feed file and all archives, and never changes. Writers should use ids that need no percent-encoding in a URL fragment, e.g. random strings of `A–Z`, `a–z`, `0–9`, `-` and `_`.
- **The item's address** is the URL of its archive + `#` + its id, percent-encoded where needed: `https://tim.example.com/feed/feed.000002.json#p-8f2kQ9xLm4Tz7wRb`. Writers use it wherever another system must find the item later: as the `source` of an RFC-4 comment or an RFC-11 mention, in links to the item from other feeds, or in a site that shows the item.
- **Order.** Writers add new items at the top and never reorder items, also not by `date_published`. The order of the files is the order in which items were added.
- **`url`** remains the item's web page, if it has one. An archive doesn't change it.

## Indexes

Every item has an **index**, `_archive.index`, a whole number from 0: the first item of a feed gets 0, every new item the next number. An item keeps its index forever, also when it is edited, and the index of a deleted item is never used again. Indexes are unique across the feed, like ids, and they grow with the order in which items were added: newer items have higher indexes, in the feed file and in every archive, and every index in an archive is higher than every index in the archives with lower numbers.

Indexes let readers work with the archives without reading all of them:

- **Counting.** The highest index in the feed file tells how many items the feed has had, so a reader can number them, e.g. as posts 0 to 46, before it has read a single archive.
- **Stable references.** The numbers don't change when items are added or deleted, so readers and sites can use them in bookmarks and URLs, e.g. `https://tim.example.com/p/feed/post/47`. Ids are stable too, but they have no order, and a reader that only has an id must read the archives until it finds it.
- **Direct access.** Every archive takes at least `archive_size` indexes before the next one starts, so the item with index `i` is in archive `⌊i / archive_size⌋` or a lower one (see [Reading](#finding-an-item-by-index)).

**The counter.** `next_index` is the index the next new item gets. It lives in the file that new items are added to: the newest archive, or the feed file of a feed without archives. A writer gives a new item that index and increases the counter in the same write that adds the item, so that two writers with conditional writes never give out the same index. A new archive gets the counter of the archive before it, after the new item's index has been taken from it.

**Items without an index**, e.g. written by a writer that doesn't know indexes, get one when a writer that knows them next writes the file that holds the counter: every item there without an index gets the next index, from the oldest to the newest, and the copies in the feed file get the same. Readers treat an item without an index as newer than every indexed item of the same file.

## Writing

This section describes the files after each change. A system with one writer can apply it directly, as a PUPS server does for the apps that post to a feed ([RFC-7](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md)). Where several writers share the files themselves, they need conditional writes, such as ETags, and a common procedure.

**The archive comes first.** The archives are the record of the feed; the feed file is a copy of its newest items. Every change is written to the archive before the feed file. An interrupted writer then leaves an item that is archived but not yet in the feed file, which the next write repairs, but never an item in the feed file that no archive holds.

### Adding an item

1. Give the item the index `next_index` of the newest archive, and increase the counter.
2. If the newest archive holds fewer than `archive_size` items, insert the item at its top. Otherwise create the next archive, with the item as its only item, `next_url` pointing to the previous archive, and the counter. Write the archive.
3. Insert the item at the top of the feed file, drop items beyond the size the writer keeps, point `archive_url` to the newest archive, and write the feed file.

**Repairing.** Whenever a writer writes the feed file after adding an item, it makes sure that every item of the newest archive is in it, in the archive's order, at the top. When it adds an item to an archive that already holds an item with the same `id`, it replaces that copy instead of adding a second one; the copy keeps its index and doesn't count as a new item.

### Editing and deleting

- **Editing.** The writer finds the item's archive, starting with `archive_url` and following `next_url`, or by its index. It changes the item there, sets `date_modified`, and writes the archive; then it changes the copy in the feed file, if there is one. The item keeps its id, index and archive.
- **Deleting.** The writer removes the item from its archive and then from the feed file, and fills the feed file up again from the archives, so that it keeps holding the newest items. The index becomes a gap.

### Starting the archives

A feed that has no archives yet, e.g. a new feed or one that kept only its newest items so far, starts them like this:

1. Give the items the feed file already has indexes, if they have none, from the oldest to the newest.
2. Split them, from the oldest to the newest, into archives of `archive_size`: archive 0 gets the oldest, archive 1 the next, and so on. The newest archive may hold fewer. If the feed file has a `next_url` to older pages, it becomes the `next_url` of archive 0: those pages are older than the archives and remain what they were, without indexes and without lasting addresses.
3. Write the archives, from the oldest to the newest, the newest with `next_index`. Then write the feed file with `archive_url`, and without `next_index`, and with `next_url` only if it equals `archive_url`.

A feed without items starts its archives with its first item. Once a feed has archives, it keeps them: a writer that doesn't support archives must not add, edit or delete items of such a feed, because the archives would then miss the item or keep an outdated copy. It may still change the feed's settings, which live in the feed file.

### Numbering an existing feed

A feed with archives whose newest archive has no `next_index` hasn't been numbered yet. Before a writer that knows indexes adds an item, it numbers the feed: it reads all archives, gives every item without an index one, from the oldest item to the newest (the first one above the highest index found, or 0), and writes every archive that changed, from the oldest to the newest, the newest with `next_index`, and then the feed file with the same indexes on its copies.

## Reading

**Subscribing.** A reader subscribes to the feed file and polls it, as any JSON Feed. It finds every new item there.

**The whole history.** A reader that wants all items follows `archive_url` to the newest archive, and from there `next_url` until it ends. Every item is in exactly one archive; the newest are also in the feed file, so readers identify items by `id` and show each once. A reader that shows the history page by page may use the archives as pages: with the template, page `n` is archive `n`, and links to it last.

**From an archive to the feed.** A reader that comes across an archive, e.g. through the address of an item in a comment, a mention or a link, finds the feed to subscribe to, and its current title, icon and settings, at the archive's `updates_url`.

**Finding an item by its address.** The reader fetches the archive and finds the item by `id`. If the archive answers `404` or `410`, or no longer has the item, the item has been deleted.

### Finding an item by index

1. Read the feed file. If an item with the index is there, that's it. If the index is above the highest index in the feed file, there is no such item yet.
2. Without a template, follow the chain from `archive_url` until an archive holds indexes at or below the one sought. With a template, read archive `min(N, ⌊i / archive_size⌋)`, where `N` is the number of the newest archive (its `_archive.number`).
3. If the archive's lowest index is above `i`, go to the next lower archive; if its highest index is below `i` and it isn't the newest, go to the next higher one. The item is usually in the first archive read, or one or two below it. Only in a feed that was numbered after items had been deleted may it be higher.
4. An index that falls between two items, or below the lowest, is a deleted item. Readers skip it, and a page for it says that the post doesn't exist.

## Using archives with other RFCs

- **RFC-4 comments and RFC-11 mentions.** An author whose feed has archives uses the item's address as `source`. It never changes, so the receiver still finds the item when it checks an edit or a delete months later, without following the chain.
- **Targets.** A comment or reply may name the answered item by its address in an archive or by the feed file's URL + `#id`. Receivers accept both.
- **`_submissions`** is in the feed file only, and applies to all items of the feed and its archives. A receiver or reader that has an archive finds it at `updates_url`.
- **Post pages.** A site that shows every post on a page of its own can name the pages by index, e.g. `https://tim.example.com/p/feed/post/47` ([RFC-6, Post URLs](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#post-urls)), and find the item as described above.

## Security considerations

- **Following links.** `archive_url`, `next_url`, `updates_url` and the template are URLs that the publisher controls. Readers follow them only to `https` (or `http`) URLs, never to `localhost`, `*.local` or private networks, never to a URL they have already read in the same walk, and only up to a limit, e.g. 10 archives for a search, or as many as the user pages through.
- **Who an archive belongs to.** Anyone can publish a file whose `updates_url` names someone else's feed. A reader or receiver that relies on that relation, e.g. to take `_submissions` or an account from the feed file, checks that the archive is on the same host as the feed file, or that it is in the feed's chain.
- **Size.** Archives are bounded by `archive_size`, but items can be large. Readers apply the same limits to archives as to feed files.

## Example

`https://tim.example.com/feed.json`, a feed with three archives:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Tim's Notes",
  "home_page_url": "https://tim.example.com/",
  "feed_url": "https://tim.example.com/feed.json",
  "_archive": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md",
    "archive_url": "https://tim.example.com/feed/feed.000002.json",
    "archive_template": "https://tim.example.com/feed/feed.{number}.json",
    "archive_size": 20
  },
  "items": [
    {
      "id": "c-Lq7Wm2xP9dRt4yZa",
      "content_text": "Agreed, archives don't need a CMS.",
      "date_published": "2026-11-15T10:03:00Z",
      "_archive": { "index": 41 }
    },
    {
      "id": "p-8f2kQ9xLm4Tz7wRb",
      "title": "Archives without a CMS",
      "content_html": "<p>My feed now archives itself.</p>",
      "date_published": "2026-11-15T09:12:00Z",
      "_archive": { "index": 40 }
    }
  ]
}
```

The feed file continues with 28 more items: the rest of archive 2, then the newest items of archive 1. The feed has had 42 items, numbered 0 to 41.

`https://tim.example.com/feed/feed.000002.json`, the newest archive. It holds the two items above, and gets every new item until it holds 20:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Tim's Notes",
  "home_page_url": "https://tim.example.com/",
  "feed_url": "https://tim.example.com/feed/feed.000002.json",
  "user_comment": "This is an archive of https://tim.example.com/feed.json. New items appear in the feed at _archive.updates_url, older ones are in the archive at next_url.",
  "next_url": "https://tim.example.com/feed/feed.000001.json",
  "_archive": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md",
    "updates_url": "https://tim.example.com/feed.json",
    "number": 2,
    "next_index": 42
  },
  "items": [
    { "id": "c-Lq7Wm2xP9dRt4yZa", "content_text": "…", "date_published": "2026-11-15T10:03:00Z", "_archive": { "index": 41 } },
    { "id": "p-8f2kQ9xLm4Tz7wRb", "title": "Archives without a CMS", "content_html": "…", "date_published": "2026-11-15T09:12:00Z", "_archive": { "index": 40 } }
  ]
}
```

`feed.000001.json` holds the 20 items with indexes 20 to 39, `"number": 1` and `next_url` → `feed.000000.json`; `feed.000000.json` holds indexes 0 to 19, `"number": 0` and no `next_url`. The post with index 40 has the address `https://tim.example.com/feed/feed.000002.json#p-8f2kQ9xLm4Tz7wRb`. A reader looking for index 25 reads `feed.000001.json` (`⌊25 / 20⌋ = 1`) directly.

## Conformance

A **publisher** conforms if:

- every item of the feed is in exactly one archive, and never moves to another;
- the archives are linked with `next_url` from the newest to the oldest, are numbered from 0 without gaps, hold at most `archive_size` items, and are never deleted;
- the feed file names the newest archive in `archive_url`, and every archive names the feed file in `updates_url` and has its `number`;
- if it gives indexes, they follow [Indexes](#indexes);
- it writes every change to the archive before the feed file.

A **reader** conforms if it identifies items by `id` across the feed file and the archives, takes the feed's settings from the feed file, and follows links only as described in [Security considerations](#security-considerations). Everything else (showing the history, using indexes, the template) is optional.

## Open questions

- **RSS.** RSS feeds can use RFC 5005 for the chain. Should this RFC define item addresses and indexes for RSS as well, e.g. with `atom:link` and a namespace element for the index?
- **Dates.** Should the feed file or each archive name the range of dates it covers, so that readers can jump to a month without reading archives?
- **Name.** `_archive` and the URI of `about` are placeholders until the extension has a permanent home on bigwide.net.
