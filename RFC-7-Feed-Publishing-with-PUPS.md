# RFC: PUPS Feeds - Publishing a Blog Feed with a Personal Upload Server

Nov 23, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-7-Feed-Publishing-with-PUPS.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-7-Feed-Publishing-with-PUPS.md)

## Summary

This proposal describes how a blog feed, or any similar stream of posts, is kept on the owner's domain by a [Personal Upload Server (PUPS, RFC-6)](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md), and how apps write it: how they set up a feed, which fields they may change, how they post, edit and delete items, and how they register feeds in `host-meta.json`, so that several apps can share a feed without breaking it. The server keeps every item in archives that follow [JSON Feed Archives (RFC-13)](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md). This document adds one small extension, `_pups`, and one request to PUPS: `POST` of an item to a feed.

In short:

- The owner's **main feed** is `/feed.json`, a [JSON Feed 1.1](https://www.jsonfeed.org/version/1.1/) document. Its media and archives live in `/feed/`. Further **named feeds** live in `/feed/` too: the feed `photos` is `/feed/photos.json`, with its files in `/feed/photos/`.
- **JSON Feed only.** No RSS, no Atom. An owner who needs RSS needs a real CMS.
- **The server keeps the feed.** A feed file that carries `"_pups": { "pupsControlled": true }` is a **kept feed**: the server writes its archives, gives new items their index and the URL of their page, and writes the feed file with copies of the newest 30 items. A feed without the marker is an ordinary file, and never touched.
- **Apps post items.** An app sends a new item, or a changed one, with `POST` to the feed file; it deletes one by posting its id with a delete marker. The server answers with the item as stored and its address.
- **Apps may also write the feed file**, with `PUT` and ETags as for any file: the server compares its items with the ones it had, and adds, changes or deletes items in the archives accordingly. This is how apps change the feed's settings, such as its title.
- **Archives** keep every item, each in exactly one archive, as RFC-13 defines: `feed.000000.json`, `feed.000001.json` and so on, with at most 20 items each, linked from the newest to the oldest with `next_url`. Items never move between archives, so the archive's URL plus `#id` addresses an item forever. Only the server writes them.
- Every item gets an **index**, `_archive.index` (RFC-13): 0 for the first item, then counting up, never changed and never used again. Readers can number and page through the items without reading every archive.
- Every feed is **registered in `host-meta.json`**, so apps and readers can find all of the owner's feeds.
- A feed **accepts comments** if its feed file has RFC-4 `_submissions`, e.g. naming the PUPS server's built-in receiver. Archives never carry it. The owner approves comments in their inbox, and the server publishes them in a **comment feed** per post, which the post's `_source.comments` names, with the number of comments.
- A site that shows every post on a page of its own declares its **post URLs** with a template in `host-meta.json`; the server gives new items that URL.
- How a feed becomes a web page is **out of scope**.

**Why the server keeps the feed.** RSS and JSON Feed were made to syndicate websites; in the Big Wide network, the feed is the original, and it needs archives that never lose an item (RFC-13). Writing them takes several files per post: the newest archive, sometimes a new one, and the feed file, each with its own precondition. If every app did that over plain `PUT`, every app would need the same archive logic, an interrupted app would leave half-written feeds behind, and two apps writing at the same moment would have to repair each other's work. The server already has the files and a lock; it does the work in one place, at once, for every app.

## Goals and non-goals

**Goals**

- Any app that holds a feed's scopes can set up the feed, post to it, and edit or delete posts, without knowing which other apps write it.
- An owner can have several feeds, e.g. a blog, a photo stream and a podcast, and apps can find them.
- A feed that the owner maintains by other means is never touched.
- Feed files stay small enough to fetch often, and every post stays available, at a URL that never changes.
- Writing a post is a single request; apps need no archive logic.
- Concurrent writes by several apps, or by the owner, never lose a post.

**Non-goals**

- **RSS and Atom.** Producing two formats from one set of posts, and keeping them in sync, is the job of a CMS. JSON Feed is easy to produce, easy to parse, and supports everything the bigwide.net extensions need.
- **HTML.** No app writes web pages; PUPS doesn't allow it. See [Rendering](#rendering).
- **Real-time notifications.** PUPS servers don't support [WebSub](https://www.w3.org/TR/websub/) yet, so feeds have no `hubs`.
- **App-private feeds.** A feed that an app keeps in its own directory, `/appdata/<client-host>/`, has only one writer, and the app may organize it differently. The exception is the author feeds of RFC-4 comments, which follow [Comment feeds](#comment-feeds-rfc-4). These are ordinary files; the server doesn't keep them.

## Roles

- **Owner**: the person who controls the domain and runs the PUPS server, as in RFC-6. The feeds are theirs.
- **Server**: the PUPS server, which keeps the feeds.
- **App** (client): an app that writes feeds through PUPS: a blogging app, a photo app, a podcast publisher, a feed-based social app.
- **Reader**: any feed reader or other software that reads the feeds. Readers need nothing beyond JSON Feed 1.1.

## Overview

1. **Authorize.** The app discovers the owner's PUPS server and asks for the scopes of a feed, e.g. `write:/feed.json` and `write:/feed/` for the main feed.
2. **Set up.** If the feed doesn't exist, the app asks the owner for a title and a description, creates the feed file with the marker, and registers it in `host-meta.json`. If it exists, the app writes it only if it carries the marker.
3. **Post.** The app sends the new item with `POST` to the feed file. The server gives it its index, adds it to the newest archive, or starts a new archive if the newest is full, and puts it at the top of the feed file.
4. **Edit and delete.** The app sends the changed item, or the item's id with a delete marker, in the same way. The server changes the item in its archive and in the feed file.

## Feeds and their files

Every feed consists of a **feed file**, the JSON Feed that readers subscribe to and poll, and a **feed directory**, which holds its archives, icons and media.

| | Main feed | Named feed `photos` |
| --- | --- | --- |
| Feed file | `/feed.json` | `/feed/photos.json` |
| Feed directory | `/feed/` | `/feed/photos/` |
| Archives | `/feed/feed.000000.json`, `/feed/feed.000001.json`, … | `/feed/photos/feed.000000.json`, … |
| Comment feeds | `/feed/comments/<index>.json` | `/feed/photos/comments/<index>.json` |
| Icons | `/feed/images/icons/` | `/feed/photos/images/icons/` |
| Media | `/feed/images/`, `/feed/media/` | anywhere in `/feed/photos/`, e.g. `/feed/photos/images/` |
| `feed_url` | `https://<owner host>/feed.json` | `https://<owner host>/feed/photos.json` |
| Scopes | `write:/feed.json write:/feed/` | `write:/feed/photos.json write:/feed/photos/` |

`<owner host>` is the host of `me` from the token response. All rules in this document apply to every feed in the same way. Where the text says `/feed.json` and `/feed/`, read `/feed/<name>.json` and `/feed/<name>/` for a named feed.

### The main feed

`/feed.json` is the owner's main feed, e.g. their blog. Readers find it on the domain without any discovery. Most owners need nothing else.

### Named feeds

An owner who wants more than one feed, e.g. one for photos and one for a podcast, gives each further feed a **name**.

- A name consists of 1 to 40 lowercase ASCII letters, digits and `-`, and doesn't start or end with `-`: `photos`, `podcast`, `notes-de`.
- The owner chooses the name when the feed is created. The app may suggest one derived from the title. The name is part of every URL of the feed and its archives, so it never changes.
- `images`, `media` and `comments` are not available as names: in `/feed/`, they belong to the main feed.
- **`/feed/` is shared** by the main feed and the named feeds. At its top level, it holds only the main feed's archives (`feed.<NNNNNN>.json`), the main feed's media directories `images/` and `media/`, its comment feeds in `comments/`, and the named feeds' files `<name>.json` and directories `<name>/`. Apps store nothing else there.
- A named feed's `home_page_url` is the domain's root too. Its title, not its name, is what readers show.

## Server requirements

The app needs a PUPS server whose configuration document says that it keeps feeds, and declares the [well-known paths](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#the-list-of-well-known-paths) of the feeds it writes:

```json
"kept_feeds": true,
"well_known_paths": [
  { "path": "/feed.json", "url": "https://tim.example.com/feed.json", "rels": ["https://bigwide.net/ns/feed"] },
  { "path": "/feed/", "url": "https://tim.example.com/feed/", "rels": ["https://bigwide.net/ns/feed"] }
]
```

- **`kept_feeds: true`** says that the server keeps feeds as this document describes. Without it, apps don't create or write feeds there.
- **Feed files** are the declared files with the feed relation, `/feed.json`, and the files `<name>.json` directly in a declared directory with the feed relation, `/feed/<name>.json`. Only these can be kept feeds; their archives are `feed.<NNNNNN>.json` in the feed directory.
- For the main feed, the server declares `/feed.json` and `/feed/`. For named feeds, `/feed/` is enough: they have no well-known paths of their own, but are files and directories inside it. The `rels` of `/feed/` allow apps to register named feeds in `host-meta.json`.
- **Servers declare both paths with the feed relation**, `"rels": ["https://bigwide.net/ns/feed"]`, as in the list of well-known paths. Without it, the host-meta endpoint refuses to register the feed (`403`), and readers and websites can't find the owner's named feeds. An app that finds the paths declared without the relation still publishes the feed, and tells the owner which link to add to `host-meta.json` by hand, as it does without a `host_meta_endpoint`.
- The app requests both scopes of a feed, its file and its directory, and needs both: without the directory it can't upload media or icons. If the owner grants only one of them, the app treats the grant as denied and tells the owner why. Posting items needs the scope of the feed file.
- An app that works with one named feed asks for that feed's scopes, `write:/feed/photos.json write:/feed/photos/`. The main feed's scopes include `write:/feed/`, and therefore every named feed as well: an app that writes the main feed can write all of the owner's feeds, and the consent screen should say so. An app with per-feed scopes that wants to create another feed runs a new authorization with the new feed's scopes.
- If the server doesn't declare the paths, e.g. because the domain's website already serves a feed there, the app can't publish that feed. It tells the owner, and may offer to keep a feed in its own directory instead.
- All feed scopes need step-up confirmation (RFC-6).

The app reads files from `storage_base` and writes them there, as RFC-6 requires for ETags. It reads them with its token, as [fresh reads](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#fresh-reads), so that no cache hands it an old version. It uses the **public URLs** (based on the `url` of the well-known entries) in everything it writes into a feed, e.g. icons and media; the server does the same for `feed_url`, `next_url`, `archive_url` and `updates_url`.

## The feed file

The feed file is a JSON Feed 1.1 document, served as `application/json`.

### Fields

Some top-level fields are set by the server. All others belong to the owner, who sets them through any app.

| Field | Value | Set by |
| --- | --- | --- |
| `version` | `https://jsonfeed.org/version/1.1` | server |
| `title` | The feed's name for humans. **Required.** | owner |
| `home_page_url` | `https://<owner host>/`, the root of the owner's domain | server |
| `feed_url` | The feed file's public URL, e.g. `https://<owner host>/feed.json` | server |
| `description` | What the feed is about. Recommended. | owner |
| `user_comment` | A note that the feed is kept by PUPS, see [below](#user_comment) | server |
| `next_url` | Not set in a feed with archives; readers follow `_archive.archive_url`, see [Reading](#reading) | server |
| `icon`, `favicon` | Images in `/feed/images/icons/`, or `/feed/<name>/images/icons/` for a named feed, see [Icons](#icons) | owner |
| `authors`, `language`, `expired` | As in JSON Feed | owner |
| `hubs` | **Never set.** PUPS servers don't support WebSub yet. | — |
| `items` | The newest 30 items, newest first | server |
| `_pups` | The marker, see [The `_pups` object](#the-_pups-object) | owner |
| `_archive` | RFC-13: the newest archive, the archive template and size. See [Archives](#archives). | server |
| other extensions | E.g. `_podcast` ([RFC-3](https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md)), `_source` ([RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md)), `_submissions` ([RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md)) | owner |

- **The home page is the domain.** A feed belongs to the domain, not to an app, so `home_page_url` is always the domain's root, never the app's website. What the domain shows there is up to the owner (see [Rendering](#rendering)).
- **The server sets its fields** whenever it writes the feed file, whatever an app sent for them. A JSON Feed 1.0 file is upgraded to 1.1 (`author` becomes `authors`).
- **Unknown fields are kept.** An app changes only the fields it means to change and writes everything else back unchanged, including fields and extensions it doesn't know. [RFC-6](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#well-known-paths) requires this for all shared well-known files, and it is what allows several apps to share a feed. The server keeps them too, in the feed file and in every item.
- **No `hubs`.** A `hubs` entry without a server that notifies the hub would make readers wait for updates that never come. Apps don't add it, and remove it when the owner asks.

### The `_pups` object

A kept feed's file, and each of its archives, carries a `_pups` object, following JSON Feed's [extension rules](https://www.jsonfeed.org/version/1.1/#extensions-a-name-extensions-a):

```json
"_pups": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
  "pupsControlled": true
}
```

- `pupsControlled: true` says that the server keeps the feed according to this document.
- **An app never writes a feed file without the marker**, except to create it, with `If-None-Match: *` and the marker. A feed file without the marker is an ordinary file for the server too.
- Archive links, archive numbers and item indexes are not part of `_pups`: they are in the `_archive` objects of RFC-13, in the feed file, the archives and the items.
- `about` should be this document. **This is not its final location**: the URL will change once the spec has a permanent home. Apps and servers therefore ignore the value of `about`.
- Apps and servers keep the other fields of `_pups` unchanged, including fields they don't know.

The marker protects feeds that the owner maintains by hand or with another tool: an app that finds such a feed leaves it alone (see [Setting up a feed](#setting-up-a-feed)), and the server treats it as an ordinary file. The owner hands a feed over to the server by adding the marker, and takes it back by removing it.

### `user_comment`

`user_comment` tells people who open the file how it is maintained. The server sets it, e.g. to:

> This feed is kept by a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. It contains the newest items; all items are in the archives, starting with the one at _archive.archive_url.

The text is for humans. Apps recognize a kept feed by `_pups`, never by this text.

### Icons

`icon` (512 × 512 pixels) and `favicon` (192 × 192, large enough for touch UIs) are optional and chosen by the owner. They are stored in `/feed/images/icons/` for the main feed, e.g. `https://tim.example.com/feed/images/icons/icon-k3x9.png`, and in `/feed/<name>/images/icons/` for a named feed, e.g. `https://tim.example.com/feed/photos/images/icons/icon-m2q7.png`.

- The app lets the owner pick or upload the image. It may offer to crop or scale it, but doesn't create an icon on its own, e.g. from its own logo.
- Every new image gets a **new file name**, so that readers and caches that keep the old URL never show a stale image under a new name. After the feed has been written with the new URL, the app deletes the old file.
- Only icons go into these directories. Apps don't store anything else there.

## Setting up a feed

After authorization, the app `GET`s the feed file from `storage_base`.

**`404`: no feed yet.** The app offers to create one. It asks the owner for:

- the **name**, for a named feed;
- the **title** (required) and the **description**;
- optionally the icon and favicon, the authors (name, URL, avatar), and the language.

It may offer advanced options, but should hide them by default: for a podcast, `_podcast.blocks` (`podcast:block`, to keep the feed out of podcast directories), `explicit`, `medium`, `locked` and the show's `guid`; for RFC-4, the owner's SocialSpec account in `_source.accounts`. The app says clearly that the feed is public.

It then writes the feed file with these fields, `_pups` and an empty `items` array, with `If-None-Match: *`. The server adds its fields (see [Writing the feed file](#writing-the-feed-file)). If that fails with `412`, someone else has created a feed under that name in the meantime; the app loads it and continues as below, or, for a named feed, asks the owner for another name. Finally, it [registers the feed](#registering-feeds-in-host-metajson) in `host-meta.json`.

**`200`: the feed exists.** The app checks that the file is a JSON Feed (a `version` starting with `https://jsonfeed.org/version/1`) and that it carries `"pupsControlled": true`. If either is missing, **the app never writes the file**. It tells the owner that the feed is maintained by other means, and that they can hand it over as described below.

**Handing over an existing feed.** An owner who wants the server to keep a feed that was made by hand or by another tool adds the `_pups` object to it, e.g. with a text editor. An app may show the owner the exact lines to add, but never adds them itself. From then on, the feed is treated like any other. On the next write, the server starts its archives (RFC-13 "Starting the archives"): the items the feed has get indexes, from the oldest to the newest, and go into archives of 20, the oldest first. An existing `next_url` becomes the `next_url` of the oldest archive, the end of the chain. The pages it points to are not archives and are never written. An app may register the feed if it isn't registered yet.

## Registering feeds in host-meta.json

PUPS has no directory listing, so there is no other way to learn which feeds a domain has. Every feed should therefore be registered in the domain's `host-meta.json`, with the link relation `https://bigwide.net/ns/feed`, through PUPS's [host-meta endpoint](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#editing-host-metajson):

```json
{ "action": "add", "rel": "https://bigwide.net/ns/feed", "href": "https://tim.example.com/feed/photos.json", "type": "application/feed+json" }
```

- **When.** The app that creates a feed registers it right afterwards. This includes the main feed, although readers also find it at `/feed.json` without discovery.
- **No duplicates.** Before it registers a feed, the app reads `host-meta.json` and skips the request if a link with this relation and `href` exists already, e.g. because the owner added it by hand.
- **Only feed files.** `href` is always a feed file, `/feed.json` or `/feed/<name>.json`, never an archive or another file in a feed directory.
- **Without the endpoint.** If the configuration document has no `host_meta_endpoint`, the app tells the owner which link to add to `host-meta.json` by hand.
- **Finding feeds.** An app that wants to post to one of the owner's existing feeds reads the links with this relation from `host-meta.json`, fetches each feed for its title, and lets the owner choose. It still checks `_pups.pupsControlled` before writing.

`host-meta.json` then looks like this, with the ownership property that PUPS adds to every link it writes:

```json
{
  "links": [
    { "rel": "https://bigwide.net/ns/personal-storage", "type": "application/json", "href": "https://tim.example.com/personal-storage.json" },
    {
      "rel": "https://bigwide.net/ns/feed", "type": "application/feed+json", "href": "https://tim.example.com/feed.json",
      "properties": { "https://bigwide.net/ns/pups#client": "https://app.example/oauth/client.json" }
    },
    {
      "rel": "https://bigwide.net/ns/feed", "type": "application/feed+json", "href": "https://tim.example.com/feed/photos.json",
      "properties": { "https://bigwide.net/ns/pups#client": "https://photos.example/oauth/client.json" }
    }
  ]
}
```

When the owner revokes an app's grant, its feed links stay and become the owner's, as RFC-6 specifies for well-known paths. The relation `https://bigwide.net/ns/feed` is a placeholder until it has a permanent home.

## Items

An item is a JSON Feed 1.1 item. The conventions:

- **`id`** is unique across the feed and all its archives, and **never changes**. Apps should use a random string of at least 16 characters from `A–Z`, `a–z`, `0–9`, `-` and `_`, e.g. `p-8f2kQ9xLm4Tz7wRb`, so that the id can be used in a URL fragment without encoding (an item's address is the URL of its archive + `#id`). An item posted without an id gets one from the server.
- **`date_published`** is always set; the server sets it to the time of the post if the app doesn't. `date_modified` is set whenever the item is edited, by the server unless the app sets it.
- **Content.** JSON Feed requires `content_html` or `content_text`, and the server refuses items without both. For `content_html`, apps should consider [Feed HTML (RFC-2)](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md): HTML within that subset looks the same in every reader that supports it, anything beyond it may be dropped. Markdown is an alternative: the app writes the text as Markdown in `_source.markdown` ([RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md#item-markdown)), and the same text as `content_html` or `content_text` for readers that don't know the extension.
- **`url`** is the item's web page. An app sets it only if it knows that the page exists, e.g. because the owner entered it. Otherwise the server sets it from the site's post URL template, see [Post URLs](#post-urls). `external_url` can be used for link posts.
- **Media** (images, podcast episodes, other attachments) go into the feed directory, under the PUPS file-type allowlist, and are referenced by their absolute public URL. For the main feed, that means `/feed/images/` and `/feed/media/`, because everything else in `/feed/` belongs to the named feeds. A named feed's directory is organized as apps like, except for the names reserved below; `images/` and `media/` are recommended there too. Like icons, media files get new names when they change, and are never overwritten. The app uploads an item's media before it posts the item.
- **Order.** `items` is sorted newest first, in the feed file and in every archive: a new item is inserted at the top. Nobody reorders items, also not by `date_published`.
- **`_archive.index`** numbers the items, see [Indexes](#indexes). The server sets it; apps send it back unchanged.
- **`_source.comments`** ([RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md#item-comments)) is set by the server while the feed's comments go to its built-in receiver, see [Publishing comments](#publishing-comments). Elsewhere, it is the app's.
- **Who may change an item.** Any app may edit or delete any item, whichever app created it, but only when the owner asks it to. It keeps all fields of the item that it doesn't change.

**Reserved names in a feed directory**: the archive files `feed.<NNNNNN>.json`, the directory `comments/` for the [comment feeds](#publishing-comments), and the directory `images/icons/`, i.e. `/feed/images/icons/` and `/feed/<name>/images/icons/`. Apps store nothing else under these names.

## Posting items

An app adds, changes and deletes items with `POST` of a JSON object to the feed file at `storage_base`, with its bearer token, e.g. `POST https://tim.example.com/feed.json` with `Content-Type: application/json`. Behind a [POST fallback](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#post-fallback), it is `POST <post_endpoint>?method=POST&path=%2Ffeed.json`. The body is at most `max_upload_bytes`.

**Adding.** The body is the new item, without `_archive.index`:

```json
{ "id": "p-8f2kQ9xLm4Tz7wRb", "title": "Archives without a CMS", "content_html": "<p>My feed now archives itself.</p>" }
```

The server answers `201 Created` with the item as stored and its address, which is also the `Location`:

```json
{
  "item": {
    "id": "p-8f2kQ9xLm4Tz7wRb", "title": "Archives without a CMS", "content_html": "<p>My feed now archives itself.</p>",
    "date_published": "2026-11-15T09:12:00Z", "_archive": { "index": 40 }, "url": "https://tim.example.com/p/feed/post/40"
  },
  "address": "https://tim.example.com/feed/feed.000002.json#p-8f2kQ9xLm4Tz7wRb"
}
```

**Changing.** The body is the whole item as it should be, with the `_archive.index` the server gave it, which names the item. It replaces the stored item; the item keeps its index, its archive and, if the body has none, its `date_published`. The server answers `200 OK` with the item as stored and its address. An item sent unchanged changes nothing.

**Deleting.** The body is the item's id with a delete marker, and if the app knows it, the index:

```json
{ "id": "p-8f2kQ9xLm4Tz7wRb", "_pups": { "delete": true }, "_archive": { "index": 40 } }
```

The server answers `200 OK` with the deleted item and its former address, so that the app can e.g. delete the item's media files, unless another item still uses them, and tell those who were mentioned. The index becomes a gap. A delete marker is explicit on purpose: an item that merely lost its content is refused, not deleted.

**Which item.** An item with `_archive.index` names the item with that index and id; if there is none, e.g. because it was deleted, the server answers `404`. An item without an index is new, unless the feed file or the newest archive has an item with its id: then it replaces that item. This makes a repeated request harmless, e.g. after the answer to a post was lost, as long as the app sent an id. A delete without an index is looked for in all archives.

**Errors**, with RFC-6's error body: `404` if there is no feed file or no such item; `409` with `not_pups_controlled` if the feed file has no marker; `400` for an item without content, a malformed id, or a body that is not a JSON object; `405` for a `POST` to a file that is not a feed file; `415` unless the body is `application/json`; and RFC-6's `401`, `403`, `413`, `429` and `507`.

Each request is done completely or not at all, under the server's lock, so concurrent posts from several apps never lose an item, and never give out an index twice. `POST` takes no ETags: an edit replaces the item as a whole, and the last edit wins.

## Writing the feed file

Apps may also write the feed file with `PUT`, as RFC-6 allows for any file, e.g. to change the feed's settings. When the body carries the marker, the server doesn't store it as it is, but applies it:

1. The preconditions (`If-Match`, `If-None-Match`) are checked as for any file. Apps read the feed file, change it, and write it with `If-Match`; on `412`, they read it again and repeat the change.
2. The owner's fields of the body replace those of the feed file. The server's fields come from the server (see [Fields](#fields)).
3. The body's items are compared with those of the stored feed file, by id. An item that is new or changed is posted, as in [Posting items](#posting-items), new items from the bottom of the list to the top, so that the topmost gets the highest index. An item that was in the stored feed file and is missing in the body is deleted. Items that are unchanged stay as they are. Two items with the same id are refused with `400`.
4. The server writes the archives and the feed file, and answers `200` (`201` for a new feed) with the feed file as stored in the body, and its URL as `Content-Location`. The response has an `ETag` only if the stored file is exactly the body that was sent, which is rare; apps that want to write again read the file first.

An app that only changes the settings therefore sends the items back as it read them, and nothing happens to them. It never drops items to keep the feed file short: the server does that, and for the server, a missing item is a deleted item. A body without the marker is stored as an ordinary file, as for a feed that the owner takes back.

## Editing a feed

Apps that set up a feed should also let the owner edit its fields later: title, description, icons, authors, language, `_submissions` and the extensions they support. An app may support only some of them; it leaves the others unchanged. Every change is a `PUT` of the feed file with `If-Match`, as in [Writing the feed file](#writing-the-feed-file).

Archives keep the feed fields they were created with: readers take the feed's title, icon and so on from the feed file.

## Archives

The archives follow [JSON Feed Archives (RFC-13)](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md): they hold **every item of the feed, each in exactly one archive**, newest first, linked into a chain with `next_url` from the newest to the oldest, and the feed file holds copies of the newest 30 items and points to the newest archive with `_archive.archive_url`:

```
/feed.json  ⇢  /feed/feed.000002.json  →  /feed/feed.000001.json  →  /feed/feed.000000.json
   _archive.archive_url               next_url                 next_url
```

RFC-13 defines the format: the `_archive` fields, the feed fields an archive copies and leaves out (among them `_submissions`, see [Receiving comments](#receiving-comments-rfc-4)), item addresses, and that items never move, archives are never rebalanced and never deleted. For kept feeds, this document fixes what RFC-13 leaves open:

- **Only the server writes archives.** It refuses `PUT` and `DELETE` of a kept feed's archives, and of its comment feeds, with `403`. Apps change items with `POST`, or through the feed file.
- **Names.** Archives are called `feed.<NNNNNN>.json`, in the feed directory, with a six-digit number starting at `000000`: `/feed/feed.000000.json` for the main feed, `/feed/photos/feed.000000.json` for the feed `photos`. The feed file's `_archive.archive_template` is therefore always `https://<owner host>/feed/feed.{number}.json`, or `https://<owner host>/feed/<name>/feed.{number}.json`, and every archive's `_archive.number` is the number in its name.
- **Size.** An archive holds at most **20 items**, and `_archive.archive_size` is `20`.
- **The feed file** holds copies of the newest **30 items**, and no `next_url`.
- **When archives start.** A new feed gets its first archive with its first item. A handed-over feed gets archives on the next write, with the items it has (see [Setting up a feed](#setting-up-a-feed)).
- **Order of writes.** The server writes the changed archives first, from the oldest to the newest, and the feed file last, so that an interruption never leaves an item in the feed file that no archive holds. Each time it writes the feed file, it takes the newest items from the archives, which repairs a feed file that an interrupted write didn't reach.
- **Item addresses.** Apps use the archive's URL plus `#<id>`, as returned by `POST`, wherever an item must be referenced, e.g. as the `source` of an RFC-4 webmention.
- **Marker.** Archives carry `_pups` with `"pupsControlled": true`, like the feed file.

The archive's `user_comment` is, e.g.:

> This is an archive of https://tim.example.com/feed.json, kept by a Personal Upload Server (PUPS) following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. New items appear in the feed at _archive.updates_url, older ones are in the archive at next_url.

Example: the feed file has 30 items and `archive_url` → `feed.000001.json`, which holds 20. A new post doesn't fit, so the server creates `feed.000002.json` with the new item and `next_url` → `feed.000001.json`, then writes the feed file with the new item at the top, without its oldest item, and `archive_url` → `feed.000002.json`.

## Indexes

Every item has an **index**, `_archive.index`, as [RFC-13](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md#indexes) defines it: a whole number from 0 that grows with the order in which items were added, never changes and is never used again. Readers use it to number items and to find them without reading every archive; sites use it in post URLs; apps send it back to name the item they change.

The server gives out the indexes, with the counter `_archive.next_index` in the newest archive. Items of a handed-over feed get theirs when the server starts its archives, from the oldest to the newest. A deleted item's index becomes a gap.

**Reading with indexes** is described in RFC-13, [Finding an item by index](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md#finding-an-item-by-index).

## Post URLs

A site whose renderer shows each post on a page of its own declares where, with the site property `https://bigwide.net/ns/post-url-template` in `host-meta.json` ([RFC-6, Post URLs](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#post-urls)):

```json
"https://bigwide.net/ns/post-url-template": "https://tim.example.com/p/{feed}/post/{index}"
```

When the server stores an item without `url`, and the property is there, it sets the item's `url` to the template with the variables filled in: `{feed}` is the path of the feed file without `.json` (`feed`, `feed/photos`), `{index}` the item's `_archive.index`, `{id}` its `id`, percent-encoded. For the item with index 47 of the main feed, that is `https://tim.example.com/p/feed/post/47`.

- **The owner's URL comes first.** An item with a `url` keeps it.
- **Changed templates.** The server doesn't change a `url` an item has, also not when the template changed.
- **The renderer finds the item** by its index (see RFC-13, [Finding an item by index](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md#finding-an-item-by-index)) or its id. A page for an index or id that no item has, e.g. a deleted one, says that the post doesn't exist.

## Reading

A reader subscribes to the feed file and polls it, as any JSON Feed. It contains the newest 30 items, so readers that don't know `_archive` simply see those. Reading the archives, the whole history or single items, is described in [JSON Feed Archives (RFC-13)](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md). Readers need nothing from this document beyond it: `_pups` concerns only the apps and the server that write the feed.

Readers find the main feed at `https://<owner host>/feed.json` without any discovery, and all feeds of a domain in `host-meta.json`. A website or profile that wants to point to a feed links to its feed file.

## Rendering

This document doesn't say how a feed becomes a web page, and the server never produces HTML from a feed. Possible ways:

- A **static site generator**, running on the owner's server or elsewhere, that rebuilds the site whenever a feed file or an archive changes.
- A **static page** at `/index.html`, put there by the owner (apps can't upload HTML), that fetches `/feed.json` in the browser and renders it, following `_archive.archive_url` and `next_url` for older posts. The feed is served with `Access-Control-Allow-Origin: *`, so the page could even be hosted elsewhere. [webpup](https://github.com/bigwide-net/webpup) is such a page: it reads `host-meta.json` for the registered feeds and the [site properties](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#site-properties) (name, title, summary, avatar, banner, theme color, language), the SocialSpec files and the feeds, and shows them as a profile with a full reader, with archive pages addressed by their numbers (`feed.000003.json` is page 3) and a page for every post (`/p/feed/post/47`), which it declares with the post URL template.
- **No page at all.** The feeds are meant for readers, and the domain's home page shows something else.

Whatever is used, it only reads the files. Items that have a page of their own name it in `url`, e.g. from the [post URL template](#post-urls).

## Comment feeds (RFC-4)

Comments, and posts to discussion groups, are [RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) submissions: the author publishes them in an author feed, and the target's server republishes them. **Apps never use the main feed or a named feed as the author feed.** Its readers follow the owner's posts, not every comment the owner writes elsewhere, and comments would push the posts out of the newest 30. Instead, an app keeps submissions in **comment feeds** in its own directory. How it names and organizes them is up to the app. This layout is recommended, with one file per target site and hour:

```
/appdata/<client-host>/comments/<hash3>/<hash>.json
```

- **`<hash>`** is the first 16 characters of the lowercase hexadecimal SHA-256 of `<target host> <hour>` (with one space between them, UTF-8):
  - `<target host>` is the host of the target feed, the part of `inReplyTo` before `#`: lowercase, in its ASCII form (as `URL.hostname` returns it), without a port.
  - `<hour>` is the hour of the item's `date_published` in UTC, as `yyyymmddhh`.
- **`<hash3>`** is the last three characters of `<hash>`. It spreads the files over at most 4,096 directories, since some file systems and hosts get slow with many files in one directory.

  A comment on `https://rm.example.org/groups/twowaves/posts.json#a8Kd03XpQz` published at `2026-11-04T10:20:00Z` hashes `rm.example.org 2026110410` to `8ec59ab09c61ab7b`, and goes to `/appdata/app.example/comments/b7b/8ec59ab09c61ab7b.json`.
- **Why this layout.** Each file holds the comments on one site within one hour, so it stays small: it needs no archives, read-modify-write is cheap, and the RFC-4 server fetches it quickly. The path follows from the comment itself, so every device of the owner, and every version of the app, finds the file of a comment from its target and `date_published`, without having to remember it. The hash keeps the names short and free of special characters.
- **The file** is a JSON Feed 1.1 with a `title` (e.g. "Comments by tim.example.com"), `home_page_url` (the owner's domain, `https://<me host>/`), `feed_url` (its own public URL) and the owner's socialspec account in `_source.accounts` (`@<me host>`), so that targets with RFC-4's `requireSocialSpec` accept it. It has no `_pups` and is not registered in `host-meta.json`: it is not a feed for readers.
- **Posting.** The app creates the item with a new `id` (16 random URL-safe characters), `_source.inReplyTo` set to the target, `date_published` set to now, and content within the target's policy. It finds the comment feed for it (with the recommended layout, from the target and `date_published`), reads the file with a [fresh read](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#fresh-reads), adds the item at the top of `items`, and writes it with `If-Match`; if the file doesn't exist, it creates it with `If-None-Match: *`. On `412`, it starts again.
- **Submitting.** `source` is the file's public URL plus `#<id>`, e.g. `https://bob.example.net/appdata/app.example/comments/b7b/8ec59ab09c61ab7b.json#c-Lq7Wm2xP9dRt4yZa`. The app sends the webmention once the item is visible at that URL, so that the server doesn't fetch an older copy from a cache. It sends no [RFC-11](https://github.com/bigwide-net/RFCs/blob/main/RFC-11-Mentions.md) mentions for the item; the RFC-4 server does that when it republishes it.
- **Editing and deleting.** The app changes the item in its file, keeping `id` and `date_published` and setting `date_modified`, or removes it, and resends the webmention with the same `source`. It deletes a file whose last item it removed; RFC-4 servers treat the `404` as a delete.
- **Keeping.** The files stay as long as the author may want to edit or delete the comments (RFC-4 "Author feeds"). An app never moves an item to another file, since its `source` would change.

`public_base` must be on the owner's host for `requireSocialSpec` to work, as for every author feed (see RFC-6, [Using PUPS for Feed Submissions](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#using-pups-for-feed-submissions-rfc-4)).

## Receiving comments (RFC-4)

A feed can also be the target of RFC-4 comments: readers comment on its posts, and a receiver delivers or republishes the comments. The feed announces this with RFC-4 `_submissions` in the **feed file**, with a `comments` policy and the receiver's endpoint as `url`. The receiver may be the PUPS server's own ([RFC-6 "Built-in receiver"](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#built-in-receiver), whose `comment_endpoint` is then the `url`, and `republish` is `full` or `summary` if the owner approves and publishes comments, see [Publishing comments](#publishing-comments), or `no` if they only read them), or any other RFC-4 server. `_submissions` is one of the owner's fields: an app sets or removes it only when the owner asks, and asks before replacing one that names another receiver.

- **Only the feed file.** Archives are created with a copy of the feed file's fields, so a copy of `_submissions` would keep saying what was true when the archive was created. The server therefore leaves `_submissions` out of new archives (see [Archives](#archives)), and the feed file alone says whether the feed accepts comments, for all its items.
- **Readers** take `_submissions` from the feed file. A reader that shows an item from an archive finds the feed file at the archive's `_archive.updates_url`. The `target` of a comment is the feed file's URL plus `#<id>`; RFC-4 also allows the archive's URL plus `#<id>`.
- **Receivers** that get a comment whose target is an archive (it has `_archive.updates_url`) use the `_submissions` of the feed file at `updates_url`, if it is on the same host, and ignore any in the archive. Items that are no longer in the feed file are found in the archives, from `_archive.archive_url` (or `next_url`) on.
- **Single items** are closed with RFC-4's `_commenting: { "allowComments": false }`.
- **Discovery.** RFC-4 also announces the endpoint with an HTTP `Link` header with `rel="webmention"` where the server can send one. Feed files are static files, so their `_submissions.url` is usually the only announcement; servers that serve them with that header may add it.

## Publishing comments

When a kept feed's `_submissions` names the PUPS server's `comment_endpoint` with `"republish": "full"` or `"summary"`, the server publishes the comments that the owner approves, in one **comment feed** per post, and the replies to them in one comment feed per comment. With `"no"`, or a value it doesn't know, the comments only go into the inbox. Replies are accepted as deep as the policy's `maxDepth` (RFC-4, default 5) allows, but at most 10 levels; deeper ones are refused with `max-depth`.

**Moderation.** A comment that passes the policy's checks goes into the owner's inbox ([RFC-12](https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md)), with the link to the comment first, then a *Publish Comment* button ([RFC-9](https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md), `"style": "major"`) next to *Dismiss*. Its status page says `pending` (RFC-4 "Response and status") until the owner decides. *Publish Comment* publishes the comment and removes it from the inbox; its status becomes `success`, with the comment's address as `pubUrl`. *Dismiss* removes it; its status becomes `failure` with `moderation-rejected`. The owner's own comments need no approval: a comment whose author feed is on the owner's host, or served by the server for the owner, is published at once (edits too) and never goes into the inbox; in a feed that doesn't publish comments, it is accepted without further effect.

**Comment feeds.** The comments on a post are in `<feed directory>/comments/<index>.json`, named after the post's `_archive.index`: `/feed/comments/47.json` for the post with index 47 of the main feed, `/feed/photos/comments/47.json` for the feed `photos`. The replies to a comment are in `<feed directory>/comments/<index>/<comment id>.json`, whatever the comment's depth: all comment feeds of a post's thread share the directory of the post's index, and the comment ids, which the server assigns, are unique. A comment feed is a JSON Feed 1.1 document, newest first, written only by the server:

- `title` is "Comments on “<post title>”", or for replies "Replies to a comment by <author> on “<post title>”"; `home_page_url` is the `url` of the comment (or post) it answers, or the domain's root; `feed_url` is its own URL.
- `_pups.inReplyTo` is the address of the post or comment whose replies it holds. It ties the thread together: from any comment feed, the server finds the comment feeds above it, up to the post's.
- Each item is the comment as RFC-4 "What is republished" describes: its `id` is the guid from the submission, then `url` (the comment's link), `title`, `content_html` (cleaned to a safe subset of HTML), `authors`, `date_published`, and `_source` with `inReplyTo` (the post or comment it answers), the verified `accounts.socialspec`, and `comments` (see Counts).
- **`summary`.** Instead of `content_html`, the item has an abstract as `content_text`: the comment's plain text, without any markup, cut off after 150 characters (the last one an ellipsis `…`). Its `url` links to the full text, so the server refuses comments without one (`link-missing`).
- **Which mode.** A comment is published as its author agreed when submitting it, and never more fully: if the feed changed to `summary` since, it is published as an abstract; if the feed no longer publishes comments, *Publish Comment* fails.
- An item's address is the comment feed's URL plus `#<id>`. A reply to a comment uses that address as its `target` and `inReplyTo`, e.g. `https://alice.example/feed/comments/47.json#s-m8NdnkLz7ri4Aws9`; the server applies the policy of the post's feed, and refuses replies to comments it hasn't published (`target-gone`).

**Counts.** While the feed's `_submissions` names the server's `comment_endpoint`, the server keeps every post's `_source.comments` (RFC-1), and that of every published comment:

- A post without comments that accepts them has `{ "count": 0, "total": 0 }`, without `feedUrl`, as there is no comment feed yet. So does a comment without replies that can get them, because it is less deep than `maxDepth`. A comment at the deepest level has none.
- A post or comment with replies has `{ "feedUrl": "<its comment feed>", "count": <direct replies>, "total": <with all replies to them> }`. When a comment is published or removed, the server updates the counts of every comment above it and of the post.
- A post closed with `_commenting` that has no comments gets none. A post that has comments keeps its `feedUrl` and counts when the feed is closed, so readers still find them.
- Apps don't set it in such a feed; the server replaces what they send. When the feed is opened or closed for comments, the server updates the posts in all archives. A comment changes nothing else in the post, and gives it no `date_modified`.

**Edits and deletes** follow the feed's policy (RFC-4 "Edits and deletes"). An edit of a published comment goes into the inbox again, marked as an edit; the published version stays until the owner approves the new one, which keeps its `id` and `date_published` and gets a `date_modified`. A resubmission without changes is accepted at once. A deleted comment is removed from the comment feed and the counts, together with all replies to it, as RFC-4 requires. When a post is deleted, all comment feeds of its thread go with it.

**Limits.** The server refuses to approve a comment when the post has as many comments, on all levels, as `commentLimit` (RFC-4, default 1000) allows (`comment-limit`), and a reply that is deeper than `maxDepth` allows by then (`max-depth`). A comment feed is a single file; very long threads may need paging later (see [Open questions](#open-questions)).

## Example

`/feed.json` with three archives, and an RFC-4 comment:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Tim's Notes",
  "home_page_url": "https://tim.example.com/",
  "feed_url": "https://tim.example.com/feed.json",
  "description": "Short notes on feeds, the web and everything else.",
  "user_comment": "This feed is kept by a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. It contains the newest items; all items are in the archives, starting with the one at _archive.archive_url.",
  "icon": "https://tim.example.com/feed/images/icons/icon-k3x9.png",
  "favicon": "https://tim.example.com/feed/images/icons/favicon-k3x9.png",
  "authors": [{ "name": "Tim", "url": "https://tim.example.com/" }],
  "language": "en",
  "_pups": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
    "pupsControlled": true
  },
  "_archive": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md",
    "archive_url": "https://tim.example.com/feed/feed.000002.json",
    "archive_template": "https://tim.example.com/feed/feed.{number}.json",
    "archive_size": 20
  },
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "accounts": { "socialspec": "@tim.example.com" }
  },
  "items": [
    {
      "id": "c-Lq7Wm2xP9dRt4yZa",
      "content_text": "Agreed, archives don't need a CMS.",
      "date_published": "2026-11-15T10:03:00Z",
      "_source": {
        "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
        "inReplyTo": "https://alice.example.com/feed.xml#post-42",
        "markdown": "Agreed, archives don't need a CMS."
      },
      "_archive": { "index": 41 }
    },
    {
      "id": "p-8f2kQ9xLm4Tz7wRb",
      "title": "Archives without a CMS",
      "content_html": "<p>My feed now archives itself.</p><p><img src=\"https://tim.example.com/feed/images/2026/chain-w8d2.png\" alt=\"A chain of feeds\"></p>",
      "date_published": "2026-11-15T09:12:00Z",
      "_archive": { "index": 40 }
    }
  ]
}
```

The feed file continues with 28 more items: the rest of `feed.000002.json`, then the newest items of `feed.000001.json`. The two items above are the newest, with indexes 41 and 40: the feed has had 42 items, numbered 0 to 41.

`/feed/feed.000002.json`, the newest archive, with the copied feed fields. It holds the two items above, and gets every new item until it holds 20:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Tim's Notes",
  "home_page_url": "https://tim.example.com/",
  "feed_url": "https://tim.example.com/feed/feed.000002.json",
  "description": "Short notes on feeds, the web and everything else.",
  "user_comment": "This is an archive of https://tim.example.com/feed.json, kept by a Personal Upload Server (PUPS) following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. New items appear in the feed at _archive.updates_url, older ones are in the archive at next_url.",
  "next_url": "https://tim.example.com/feed/feed.000001.json",
  "icon": "https://tim.example.com/feed/images/icons/icon-k3x9.png",
  "favicon": "https://tim.example.com/feed/images/icons/favicon-k3x9.png",
  "authors": [{ "name": "Tim", "url": "https://tim.example.com/" }],
  "language": "en",
  "_pups": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
    "pupsControlled": true
  },
  "_archive": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md",
    "updates_url": "https://tim.example.com/feed.json",
    "number": 2,
    "next_index": 42
  },
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "accounts": { "socialspec": "@tim.example.com" }
  },
  "items": [
    { "id": "c-Lq7Wm2xP9dRt4yZa", "content_text": "Agreed, archives don't need a CMS.", "date_published": "2026-11-15T10:03:00Z", "_source": { "…": "…" }, "_archive": { "index": 41 } },
    { "id": "p-8f2kQ9xLm4Tz7wRb", "title": "Archives without a CMS", "content_html": "…", "date_published": "2026-11-15T09:12:00Z", "_archive": { "index": 40 } }
  ]
}
```

`/feed/feed.000001.json` looks the same with 20 items, indexes 20 to 39, `"number": 1`, no `next_index` and `next_url` → `feed.000000.json`; `/feed/feed.000000.json`, the oldest archive, holds indexes 0 to 19, has `"number": 0` and no `next_url`.

A named feed `photos` looks the same, with `"feed_url": "https://tim.example.com/feed/photos.json"`, its own title, and its archives and images in `/feed/photos/`.

## Open questions

- **WebSub.** Since the server writes every kept feed, it knows when one changes: a later version could notify a hub, or be one, and `hubs` would become a field the server sets. Not yet.
- **Feed titles in `host-meta.json`.** An app that lists the owner's feeds has to fetch each one to show its title. Should the host-meta endpoint accept a `titles` object for links, as the JSON form of host-meta allows?
- **Long comment feeds.** A post with hundreds of comments may need its comment feed split into RFC-13 archives.
- **Removing published comments.** So far a comment leaves its comment feed only when its author deletes it. The owner will need a way to take one down, e.g. an action in a feed of the published comments.
- **Names and URIs.** The link relation `https://bigwide.net/ns/feed` is a placeholder, like the URIs of RFC-6, to be published on bigwide.net.
