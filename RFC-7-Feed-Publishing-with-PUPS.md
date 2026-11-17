# RFC: PUPS Feeds - Publishing a Blog Feed with a Personal Upload Server

Nov 15, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-7-Feed-Publishing-with-PUPS.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-7-Feed-Publishing-with-PUPS.md)

## Summary

This proposal describes how apps maintain a blog feed, or any similar stream of posts, on the owner's domain through a [Personal Upload Server (PUPS, RFC-6)](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md). It defines no new protocol, and only one small extension, `_pups`. It is a list of conventions for clients: how to set up a feed, which fields they may change, how to post, edit and delete items, how to keep every item in archive feeds, and how to register feeds in `host-meta.json`, so that several apps can share a feed without breaking it.

In short:

- The owner's **main feed** is `/feed.json`, a [JSON Feed 1.1](https://www.jsonfeed.org/version/1.1/) document. Its media and archives live in `/feed/`. Further **named feeds** live in `/feed/` too: the feed `photos` is `/feed/photos.json`, with its files in `/feed/photos/`.
- **JSON Feed only.** No RSS, no Atom. An owner who needs RSS needs a real CMS.
- **Clients only write feeds marked as theirs.** Every file they create carries `"_pups": { "pupsControlled": true }`. A feed without it is never overwritten.
- The **feed file** contains the **newest 30 items**. A client **posts** by adding the item at the top and dropping the oldest beyond 30. Every write is a read-modify-write cycle protected by ETags.
- **Archives are optional.** A feed with archives keeps every item in exactly one archive, `feed.000000.json`, `feed.000001.json` and so on, with at most 20 items each, linked from the newest to the oldest with `next_url`. The feed file then holds copies of the newest 30 and points to the newest archive with `_pups.archive_url`; every archive points back with `_pups.updates_url`. Items never move between archives, so the archive's URL plus `#id` addresses an item forever.
- **Simple clients don't need to support archives**, but they write only feeds without them.
- Every feed is **registered in `host-meta.json`**, so apps and readers can find all of the owner's feeds.
- How a feed becomes a web page is **out of scope**.

**Why now.** PUPS gives apps a place to write the owner's feeds, and says that several apps may write the same feed. It doesn't say what they should write. Without shared conventions, every app would set up the feed differently, overwrite fields set by another app, and let the file grow forever or cut it off where it likes. A feed that one app archives into `feed-page-2.json` and another into `/feed/old/1.json` is no longer one feed. The rules here are few, but every client that writes a feed through PUPS must follow them.

## Goals and non-goals

**Goals**

- Any app that holds a feed's scopes can set up the feed, post to it, and edit or delete posts, without knowing which other apps write it.
- An owner can have several feeds, e.g. a blog, a photo stream and a podcast, and apps can find them.
- A feed that the owner maintains by other means is never touched.
- Feed files stay small enough to fetch often. With archives, every post stays available, at a URL that never changes.
- A simple client can publish a feed without implementing archives.
- Concurrent writes by several apps, or by the owner, never lose a post.
- Nothing is needed on the server beyond PUPS.

**Non-goals**

- **RSS and Atom.** Producing two formats from one set of posts, and keeping them in sync, is the job of a CMS. JSON Feed is easy to produce, easy to parse, and supports everything the bigwide.net extensions need.
- **HTML.** No client writes web pages; PUPS doesn't allow it. See [Rendering](#rendering).
- **Real-time notifications.** PUPS servers don't support [WebSub](https://www.w3.org/TR/websub/) yet, so feeds have no `hubs`.
- **App-private feeds.** A feed that an app keeps in its own directory, `/appdata/<client-host>/`, e.g. the author feed of RFC-4 comments, has only one writer. The app may organize it differently.

## Roles

- **Owner**: the person who controls the domain and runs the PUPS server, as in RFC-6. The feeds are theirs.
- **Client**: an app that writes feeds through PUPS: a blogging app, a photo app, a podcast publisher, a feed-based social app.
- **Reader**: any feed reader or other software that reads the feeds. Readers need nothing beyond JSON Feed 1.1.

## Overview

1. **Authorize.** The client discovers the owner's PUPS server and asks for the scopes of a feed, e.g. `write:/feed.json` and `write:/feed/` for the main feed.
2. **Set up.** If the feed doesn't exist, the client asks the owner for a title and a description, creates it, and registers it in `host-meta.json`. If it exists, the client writes it only if it is marked with `_pups.pupsControlled`.
3. **Post.** The client adds the new item to the top of the feed file, which keeps the newest 30 items. If the feed has archives, it first adds the item to the newest archive, or starts a new archive if the newest is full.
4. **Edit and delete.** The client changes the item in the feed file and, if the feed has archives, in the item's archive.

## Feeds and their files

Every feed consists of a **feed file**, the JSON Feed that readers subscribe to and poll, and a **feed directory**, which holds its archives, icons and media.

| | Main feed | Named feed `photos` |
| --- | --- | --- |
| Feed file | `/feed.json` | `/feed/photos.json` |
| Feed directory | `/feed/` | `/feed/photos/` |
| Archives | `/feed/feed.000000.json`, `/feed/feed.000001.json`, … | `/feed/photos/feed.000000.json`, … |
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
- The owner chooses the name when the feed is created. The client may suggest one derived from the title. The name is part of every URL of the feed and its archives, so it never changes.
- `images` and `media` are not available as names: in `/feed/`, they belong to the main feed.
- **`/feed/` is shared** by the main feed and the named feeds. At its top level, it holds only the main feed's archives (`feed.<NNNNNN>.json`), the main feed's media directories `images/` and `media/`, and the named feeds' files `<name>.json` and directories `<name>/`. Clients store nothing else there.
- A named feed's `home_page_url` is the domain's root too. Its title, not its name, is what readers show.

## Server requirements

The client needs a PUPS server whose configuration document declares the [well-known paths](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#the-list-of-well-known-paths) of the feeds it writes:

```json
"well_known_paths": [
  { "path": "/feed.json", "url": "https://tim.example.com/feed.json", "rels": ["https://bigwide.net/ns/feed"] },
  { "path": "/feed/", "url": "https://tim.example.com/feed/", "rels": ["https://bigwide.net/ns/feed"] }
]
```

- For the main feed, `/feed.json` and `/feed/`. For named feeds, `/feed/` is enough: they have no well-known paths of their own, but are files and directories inside it. The `rels` of `/feed/` allow apps to register named feeds in `host-meta.json`.
- **Servers declare both paths with the feed relation**, `"rels": ["https://bigwide.net/ns/feed"]`, as in the list of well-known paths. Without it, the host-meta endpoint refuses to register the feed (`403`), and readers and websites can't find the owner's named feeds. A client that finds the paths declared without the relation still publishes the feed, and tells the owner which link to add to `host-meta.json` by hand, as it does without a `host_meta_endpoint`.
- The client requests both scopes of a feed, its file and its directory, and needs both: without the directory it can neither upload media nor write archives. If the owner grants only one of them, the client treats the grant as denied and tells the owner why.
- A client that works with one named feed asks for that feed's scopes, `write:/feed/photos.json write:/feed/photos/`. The main feed's scopes include `write:/feed/`, and therefore every named feed as well: a client that writes the main feed can write all of the owner's feeds, and the consent screen should say so. A client with per-feed scopes that wants to create another feed runs a new authorization with the new feed's scopes.
- If the server doesn't declare the paths, e.g. because the domain's website already serves a feed there, the client can't publish that feed. It tells the owner, and may offer to keep a feed in its own directory instead.
- All feed scopes need step-up confirmation (RFC-6).

The client reads files from `storage_base` and writes them there, as RFC-6 requires for ETags. It reads them with its token, as [fresh reads](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#fresh-reads), so that no cache hands it an old version. It uses the **public URLs** (based on the `url` of the well-known entries) in everything it writes into a feed: `feed_url`, `next_url`, `archive_url`, `updates_url`, icons and media.

## The feed file

The feed file is a JSON Feed 1.1 document, served as `application/json`.

### Fields

Some top-level fields are fixed or managed by the conventions in this document. All others belong to the owner, who sets them through any client.

| Field | Value | Set by |
| --- | --- | --- |
| `version` | `https://jsonfeed.org/version/1.1` | fixed |
| `title` | The feed's name for humans. **Required.** | owner |
| `home_page_url` | `https://<owner host>/`, the root of the owner's domain | fixed |
| `feed_url` | The feed file's public URL, e.g. `https://<owner host>/feed.json` | fixed |
| `description` | What the feed is about. Recommended. | owner |
| `user_comment` | A note that the feed is PUPS-controlled, see [below](#user_comment) | fixed |
| `next_url` | Optional: the newest archive, like `archive_url`, see [Reading](#reading). Usually not set. | managed |
| `icon`, `favicon` | Images in `/feed/images/icons/`, or `/feed/<name>/images/icons/` for a named feed, see [Icons](#icons) | owner |
| `authors`, `language`, `expired` | As in JSON Feed | owner |
| `hubs` | **Never set.** PUPS servers don't support WebSub yet. | — |
| `items` | The newest 30 items, newest first, see [Posting](#posting) | managed |
| `_pups` | The PUPS marker, and, for a feed with archives, `archive_url`, the newest archive, see [The `_pups` object](#the-_pups-object) | managed |
| other extensions | E.g. `_podcast` ([RFC-3](https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md)), `_source` ([RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md)) | owner |

- **The home page is the domain.** A feed belongs to the domain, not to an app, so `home_page_url` is always the domain's root, never the client's website. What the domain shows there is up to the owner (see [Rendering](#rendering)).
- **Unknown fields are kept.** A client changes only the fields it means to change and writes everything else back unchanged, including fields and extensions it doesn't know, in the feed and in every item. [RFC-6](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#well-known-paths) requires this for all shared well-known files, and it is what allows several clients to share a feed.
- **No `hubs`.** A `hubs` entry without a server that notifies the hub would make readers wait for updates that never come. Clients don't add it, and remove it when the owner asks.

### The `_pups` object

Every file that a client creates under this document, feed files and archives, carries a `_pups` object, following JSON Feed's [extension rules](https://www.jsonfeed.org/version/1.1/#extensions-a-name-extensions-a). In the feed file:

```json
"_pups": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
  "pupsControlled": true,
  "archive_url": "https://tim.example.com/feed/feed.000002.json"
}
```

In an archive:

```json
"_pups": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
  "pupsControlled": true,
  "updates_url": "https://tim.example.com/feed.json"
}
```

- `pupsControlled: true` says that the file is maintained by PUPS clients according to this document.
- **A client never overwrites or deletes a feed file or an archive unless it carries `"pupsControlled": true`.** The only files a client writes without the marker are the ones it creates, with `If-None-Match: *`, and those get the marker.
- `archive_url`, in the feed file only, is the public URL of the newest archive. It is missing if the feed has no archives. See [Archives](#archives-optional).
- `updates_url`, in archives only, is the public URL of the feed file. It tells readers which file to poll for new items, and lets a reader that found an archive, e.g. through an RFC-4 `source`, find the feed.
- `about` should be this document. **This is not its final location**: the URL will change once the spec has a permanent home. Clients therefore ignore the value of `about`.
- Clients keep the other fields of `_pups` unchanged, including fields they don't know.

The marker protects feeds that the owner maintains by hand or with another tool: a client that finds such a feed leaves it alone (see [Setting up a feed](#setting-up-a-feed)). The owner hands a feed over to PUPS clients by adding the marker, and takes it back by removing it.

### `user_comment`

`user_comment` tells people who open the file how it is maintained. Clients set it to:

> This feed is maintained by apps through a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. It contains the newest items. If it has _pups.archive_url, all items are in the archives, starting with the one there.

The text is for humans. Clients recognize a PUPS feed by `_pups`, never by this text.

### Icons

`icon` (512 × 512 pixels) and `favicon` (64 × 64) are optional and chosen by the owner. They are stored in `/feed/images/icons/` for the main feed, e.g. `https://tim.example.com/feed/images/icons/icon-k3x9.png`, and in `/feed/<name>/images/icons/` for a named feed, e.g. `https://tim.example.com/feed/photos/images/icons/icon-m2q7.png`.

- The client lets the owner pick or upload the image. It may offer to crop or scale it, but doesn't create an icon on its own, e.g. from its own logo.
- Every new image gets a **new file name**, so that readers and caches that keep the old URL never show a stale image under a new name. After the feed has been written with the new URL, the client deletes the old file.
- Only icons go into these directories. Clients don't store anything else there.

## Setting up a feed

After authorization, the client `GET`s the feed file from `storage_base`.

**`404`: no feed yet.** The client offers to create one. It asks the owner for:

- the **name**, for a named feed;
- the **title** (required) and the **description**;
- optionally the icon and favicon, the authors (name, URL, avatar), and the language.

It may offer advanced options, but should hide them by default: for a podcast, `_podcast.blocks` (`podcast:block`, to keep the feed out of podcast directories), `explicit`, `medium`, `locked` and the show's `guid`; for RFC-4, the owner's SocialSpec account in `_source.accounts`. The client says clearly that the feed is public.

It then writes the feed with the fixed fields from the table above, including `_pups`, an empty `items` array and no archive, with `If-None-Match: *`. A client that supports archives may start them with the first post. If that fails with `412`, someone else has created a feed under that name in the meantime; the client loads it and continues as below, or, for a named feed, asks the owner for another name. Finally, it [registers the feed](#registering-feeds-in-host-metajson) in `host-meta.json`.

**`200`: the feed exists.** The client checks that the file is a JSON Feed (a `version` starting with `https://jsonfeed.org/version/1`) and that it carries `"pupsControlled": true`. If either is missing, **the client never writes the file**. It tells the owner that the feed is maintained by other means, and that they can hand it over to PUPS clients as described below. A JSON Feed 1.0 file with the marker is upgraded to 1.1 when it is next written (`author` becomes `authors`).

**Handing over an existing feed.** An owner who wants PUPS clients to take over a feed that was made by hand or by another tool adds the `_pups` object to it, e.g. with a text editor. A client may show the owner the exact lines to add, but never adds them itself. From then on, the feed is treated like any other:

- The first client that writes it sets the fixed fields (`home_page_url`, `feed_url`, `user_comment`) to their values, and registers the feed if it isn't registered yet.
- A client that supports archives may [start the archives](#starting-the-archives) with the items the feed already has. An existing `next_url` then moves to the oldest new archive and becomes the end of the archive chain. The pages it points to are not PUPS archives and are never written, unless the owner marks them too.

## Registering feeds in host-meta.json

PUPS has no directory listing, so there is no other way to learn which feeds a domain has. Every feed should therefore be registered in the domain's `host-meta.json`, with the link relation `https://bigwide.net/ns/feed`, through PUPS's [host-meta endpoint](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#editing-host-metajson):

```json
{ "action": "add", "rel": "https://bigwide.net/ns/feed", "href": "https://tim.example.com/feed/photos.json", "type": "application/feed+json" }
```

- **When.** The client that creates a feed registers it right afterwards. This includes the main feed, although readers also find it at `/feed.json` without discovery.
- **No duplicates.** Before it registers a feed, the client reads `host-meta.json` and skips the request if a link with this relation and `href` exists already, e.g. because the owner added it by hand.
- **Only feed files.** `href` is always a feed file, `/feed.json` or `/feed/<name>.json`, never an archive or another file in a feed directory.
- **Without the endpoint.** If the configuration document has no `host_meta_endpoint`, the client tells the owner which link to add to `host-meta.json` by hand.
- **Finding feeds.** A client that wants to post to one of the owner's existing feeds reads the links with this relation from `host-meta.json`, fetches each feed for its title, and lets the owner choose. It still checks `_pups.pupsControlled` before writing.

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

When the owner revokes a client's grant, its feed links stay and become the owner's, as RFC-6 specifies for well-known paths. The relation `https://bigwide.net/ns/feed` is a placeholder until it has a permanent home.

## Editing a feed

Clients that set up a feed should also let the owner edit its fields later: title, description, icons, authors, language and the extensions they support. A client may support only some of them; it leaves the others unchanged.

Every change is a read-modify-write of the feed file with `If-Match`, as in [Posting](#posting), and on `412` the client reloads the file and applies the change again.

Archives keep the feed fields they were created with. A client may update them as well, but doesn't have to: readers take the feed's title, icon and so on from the feed file.

## Items

An item is a JSON Feed 1.1 item. The conventions:

- **`id`** is unique across the feed and all its archives, and **never changes**. Clients should use a random string of at least 16 characters from `A–Z`, `a–z`, `0–9`, `-` and `_`, e.g. `p-8f2kQ9xLm4Tz7wRb`, so that the id can be used in a URL fragment without encoding (an item's URL is the URL of its file + `#id`).
- **`date_published`** is always set. `date_modified` is set whenever the item is edited.
- **Content.** JSON Feed requires `content_html` or `content_text`. For `content_html`, clients should consider [Feed HTML (RFC-2)](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md): HTML within that subset looks the same in every reader that supports it, anything beyond it may be dropped. Markdown is an alternative: the client writes the text as Markdown in `_source.markdown` ([RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md#item-markdown)), and the same text as `content_html` or `content_text` for readers that don't know the extension.
- **`url`** is the item's web page. The client sets it only if it knows that the page exists, e.g. because the owner entered it; it never invents one. `external_url` can be used for link posts.
- **Media** (images, podcast episodes, other attachments) go into the feed directory, under the PUPS file-type allowlist, and are referenced by their absolute public URL. For the main feed, that means `/feed/images/` and `/feed/media/`, because everything else in `/feed/` belongs to the named feeds. A named feed's directory is organized as clients like, except for the names reserved below; `images/` and `media/` are recommended there too. Like icons, media files get new names when they change, and are never overwritten.
- **Order.** `items` is sorted newest first, in the feed file and in every archive: a new item is inserted at index 0. Clients never reorder items, also not by `date_published`.
- **Who may change an item.** Any client may edit or delete any item, whichever client created it, but only when the owner asks it to. It keeps all fields of the item that it doesn't change.

**Reserved names in a feed directory**: the archive files `feed.<NNNNNN>.json`, and the directory `images/icons/`, i.e. `/feed/images/icons/` and `/feed/<name>/images/icons/`. Clients store nothing else under these names.

## Posting

Posting is a read-modify-write of the feed file, with ETags so that concurrent writes by another client, or by the owner, are never lost ([RFC-6 Concurrency](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#concurrency)):

1. `GET` the feed file from `storage_base` and remember its `ETag`. If it doesn't carry `"pupsControlled": true`, stop. If it has `_pups.archive_url`, the feed has archives: post as described in [Posting with archives](#posting-with-archives), or, if the client doesn't support archives, stop and tell the owner.
2. Insert the new item at the top of `items`, and drop the items beyond the newest 30. In a feed without archives, they are gone.
3. `PUT` the feed file with `If-Match: "<etag>"`.

If the write fails with `412`, the client starts again at step 1. It uploads the item's media files before step 1, so that the item never references a file that doesn't exist yet.

**Feed size.** The recommended size of the feed file is 30 items. It is not fixed: a client may use another size, e.g. 50 for a feed with short posts, or let the owner configure it. Clients that use different sizes on the same feed don't break anything; each applies its own when it writes the feed file.

**Size in bytes (optional).** Items can be large, e.g. long posts with full HTML. A client may keep fewer items in the feed file when it would otherwise be larger than a size limit it chooses, e.g. 256 KB, but keeps at least 10. Archives are not affected; they always hold 20 items.

## Archives (optional)

Archives keep every item of a feed, not only the newest 30. They are an optional feature: a simple client may publish feeds without them. **A feed has archives if its feed file has `_pups.archive_url`**, and then it keeps them.

- **A client that doesn't support archives never writes the items of a feed with archives**: it doesn't post to it, and doesn't edit or delete its items, because the archives would then miss the post or keep an outdated copy. It may still change the feed's settings (see [Editing a feed](#editing-a-feed)), which live in the feed file only.
- **A client that supports archives may start them** for a feed that has none, when it posts to it, e.g. because the owner turned archiving on (see [Starting the archives](#starting-the-archives)). From then on, only clients that support archives can write the feed's items.

The archives hold **every item of the feed, each in exactly one archive**. They are JSON Feeds too, stored in the feed directory and linked into a chain with `next_url`, from the newest to the oldest. The feed file holds copies of the newest 30 items and points to the newest archive with `_pups.archive_url`:

```
/feed.json  ⇢  /feed/feed.000002.json  →  /feed/feed.000001.json  →  /feed/feed.000000.json
   _pups.archive_url                  next_url                 next_url
```

- **Names.** Archives are called `feed.<NNNNNN>.json`, with a six-digit number starting at `000000`. Each new archive gets the next number, so the newest archive has the highest one. The name is the same for every feed; the feed directory tells them apart: `/feed/photos/feed.000000.json`.
- **Size.** An archive holds at most **20 items**. New items always go into the newest archive; when it holds 20, the next item starts a new one. Older archives only change when an item in them is edited or deleted.
- **Items never move.** Once an item is in an archive, it stays there. Its URL, the archive's URL plus `#<id>`, therefore never changes, e.g. `https://tim.example.com/feed/feed.000002.json#p-8f2kQ9xLm4Tz7wRb`. Clients use it wherever an item must be referenced, e.g. as the `source` of an RFC-4 webmention.
- **Links.** `next_url` of each archive points to the next older one. The oldest archive has no `next_url`, unless the chain continues into older pages of a feed that was handed over.
- **Feed fields.** A new archive gets a copy of the top-level fields of the feed file, with these exceptions: `feed_url` is the archive's own URL, `user_comment` says that it is an archive, `next_url` points to the next older archive, `_pups` has `updates_url` instead of `archive_url`, `items` holds the archived items, and `_podcast.liveItems` is left out.
- **Marker.** Like the feed file, an archive is written only if it carries `"pupsControlled": true`.

The archive's `user_comment`:

> This is an archive of https://tim.example.com/feed.json, maintained by apps through a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. New items appear in the feed at _pups.updates_url, older ones are in the archive at next_url.

### Posting with archives

In a feed with archives, a post writes two files, the newest archive and the feed file, each with a read-modify-write cycle protected by ETags:

1. `GET` the feed file from `storage_base` and remember its `ETag`. If it doesn't carry `"pupsControlled": true`, stop. If it has no `archive_url` and the client starts the archives now, [start the archives](#starting-the-archives) first.
2. `GET` the newest archive, the one `archive_url` names, and remember its `ETag`.
3. **Add the item to the archive.** If the archive holds fewer than 20 items, insert the item at the top of its `items` and `PUT` it with `If-Match`. Otherwise **start a new archive**: number `N + 1`, with the new item as its only item and `next_url` pointing to archive `N`. `PUT` it with `If-None-Match: *`, and set `archive_url` of the feed file to its URL.
4. **Add the item to the feed file**, as in [Posting](#posting): insert it at the top of `items`, drop the items beyond the newest 30, and `PUT` the feed file with `If-Match`.

If any write fails with `412`, the client starts again at step 1.

The order matters: **the archive is always written before the feed file**. The archives are the record of the feed; the feed file is a copy of its newest items. If the client is interrupted between the two writes, the item is in the archive but not yet in the feed file, which the next post repairs (see below), but it is never in the feed file without being archived.

Example: the feed file has 30 items and `archive_url` → `feed.000001.json`, which holds 20. A new post doesn't fit, so the client creates `feed.000002.json` with the new item and `next_url` → `feed.000001.json`, then writes the feed file with the new item at the top, without its oldest item, and `archive_url` → `feed.000002.json`.

### Starting the archives

A feed without `archive_url` has no archives: a new feed, a feed that simple clients have written so far, or a feed the owner [handed over](#setting-up-a-feed). A client that supports archives and starts them does so when it posts, before step 2 of [Posting with archives](#posting-with-archives):

1. It takes the items the feed file already has, if any, and splits them, from the oldest to the newest, into archives of 20: `feed.000000.json` gets the oldest 20, `feed.000001.json` the next 20, and so on. The newest archive may hold fewer. A feed without items gets no archive here; the post then creates `feed.000000.json` in step 3.
2. It writes each archive with `If-None-Match: *`, from the oldest to the newest, each with `next_url` pointing to the one before. The oldest archive's `next_url` is the feed file's `next_url`, if it has one.
3. It continues with step 2 of Posting with archives, with `archive_url` pointing to the newest of these archives, and removes `next_url` from the feed file when it writes it in step 4.

### Interrupted and concurrent writes

Because a post to a feed with archives writes two files, the client has to deal with writes that only half happened:

- **Duplicates.** When the client adds an item to an archive that already contains an item with the same `id`, it replaces the old copy instead of adding a second one, and the copy doesn't count as a new item for the size limit. This cleans up after a retry.
- **Repairing the feed file.** The newest archive holds the newest items. Whenever a client writes the feed file after a post, it makes sure that every item of the newest archive is in it, in the archive's order, at the top, before it drops the items beyond 30. This brings back items that an interrupted post wrote only into the archive.
- **`412` on the archive.** Another client has changed it. The client starts the whole post again at step 1.
- **`412` when creating an archive.** The file exists already: left behind by an interrupted client, or created by another client at the same moment. The client loads it. If it carries the marker, its `next_url` is the expected one and it has room for the item, the client adds the item to it, as in step 3, and points `archive_url` to it. Otherwise it reloads the feed file: if that has changed, it starts the whole post again at step 1; if not, it tries the next number.
- **`412` on the feed file** after the archive was written. The client starts again at step 1. The item is already in the archive; the duplicate rule takes care of it.
- **A missing archive.** If the archive that `archive_url` names returns `404`, e.g. because the owner deleted it, the client creates it again under the same name, with `If-None-Match: *`, with the new item, and with `next_url` pointing to the archive with the next lower number, if there is one.

## Editing and deleting items

**In a feed without archives**, the client finds the item by `id` in the feed file, changes it and sets `date_modified`, or removes it, and `PUT`s the feed file with `If-Match`. On `412`, it starts again. Items that were dropped from the feed file can't be edited or deleted any more.

**In a feed with archives**, only clients that support archives edit and delete items:

- **Editing.** The client looks for the item by `id` in the archives, starting with the one `archive_url` names and following `next_url`, and stops at the first archive that contains it. It changes the item there, sets `date_modified`, and `PUT`s the archive with `If-Match`. If the item is also in the feed file, it then changes it there in the same way. On `412`, it starts again.
- **Deleting.** The client removes the item from its archive in the same way, and then from the feed file. After removing it from the feed file, it fills the feed file up to 30 items again from the archives, so that it keeps holding the newest 30.

In both cases, after deleting an item, the client deletes the item's media files from the feed directory, unless another item still uses them.

**Archives are never rebalanced.** Only the feed file is filled up again after a delete. An archive that loses items keeps the rest, and no items move into it from other archives. In the worst case, an archive ends up empty; it keeps its place in the chain, and clients don't delete archive files. If the newest archive is emptied, the next post goes into it as usual.

## Reading

A reader subscribes to the feed file and polls it, as any JSON Feed. It contains the newest 30 items, so readers that don't know `_pups` simply see those.

In a feed with archives, a reader that wants the whole history follows `_pups.archive_url` to the newest archive, and from there `next_url` until it ends. Optionally, the feed file may also have a `next_url` that points to the newest archive, for readers that page through feeds but don't know `_pups`. If it has one, clients keep it equal to `archive_url`. Every item is in exactly one archive, but the newest 30 are also in the feed file; readers identify items by `id`, so they show each once. A reader that comes across an archive, e.g. through the `source` of an RFC-4 webmention, finds the feed to poll in the archive's `_pups.updates_url`.

Readers find the main feed at `https://<owner host>/feed.json` without any discovery, and all feeds of a domain in `host-meta.json`. A website or profile that wants to point to a feed links to its feed file.

## Rendering

This document doesn't say how a feed becomes a web page, and the server never produces HTML. Possible ways:

- A **static site generator**, running on the owner's server or elsewhere, that rebuilds the site whenever a feed file or an archive changes.
- A **static page** at `/index.html`, put there by the owner (apps can't upload HTML), that fetches `/feed.json` in the browser and renders it, following `_pups.archive_url` and `next_url` for older posts. The feed is served with `Access-Control-Allow-Origin: *`, so the page could even be hosted elsewhere. [webpup](https://github.com/bigwide-net/webpup) is such a page: it reads `host-meta.json` for the registered feeds and the [site properties](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#site-properties) (name, title, summary, avatar, banner, theme color, language), the SocialSpec files and the feeds, and shows them as a profile with a full reader, with archive pages addressed by their numbers (`feed.000003.json` is page 3).
- **No page at all.** The feeds are meant for readers, and the domain's home page shows something else.

Whatever is used, it is not a client of this document: it only reads the files. Items that have a page of their own may name it in `url`, if the client knows it.

## Using a feed for Feed Submissions (RFC-4)

The main feed, or a named feed, can be the author feed of [RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) posts and comments, as RFC-6 mentions. The owner's readers then see the comment in the feed.

1. The client gives the item `_source.inReplyTo` as usual, and the feed carries the owner's SocialSpec account in `_source.accounts`.
2. It posts the item as described in [Posting](#posting).
3. It sends the webmention with `source` = the URL of the archive it added the item to + `#<id>`, e.g. `https://tim.example.com/feed/feed.000002.json#c-Lq7Wm2xP9dRt4yZa`. The item never leaves that archive, so the `source` stays valid. In a feed without archives, `source` is the feed file's URL + `#<id>`; it is valid only as long as the item is among the newest 30.
4. To edit or delete the comment, it changes or removes the item as described in [Editing and deleting items](#editing-and-deleting-items) and resends the webmention with the same `source`.

The archives are on the owner's host, so RFC-4's `requireSocialSpec` check works for them as for the feed file.

## Example

`/feed.json` with three archives, and an RFC-4 comment:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Tim's Notes",
  "home_page_url": "https://tim.example.com/",
  "feed_url": "https://tim.example.com/feed.json",
  "description": "Short notes on feeds, the web and everything else.",
  "user_comment": "This feed is maintained by apps through a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. It contains the newest items. If it has _pups.archive_url, all items are in the archives, starting with the one there.",
  "icon": "https://tim.example.com/feed/images/icons/icon-k3x9.png",
  "favicon": "https://tim.example.com/feed/images/icons/favicon-k3x9.png",
  "authors": [{ "name": "Tim", "url": "https://tim.example.com/" }],
  "language": "en",
  "_pups": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
    "pupsControlled": true,
    "archive_url": "https://tim.example.com/feed/feed.000002.json"
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
      }
    },
    {
      "id": "p-8f2kQ9xLm4Tz7wRb",
      "title": "Archives without a CMS",
      "content_html": "<p>My feed now archives itself.</p><p><img src=\"https://tim.example.com/feed/images/2026/chain-w8d2.png\" alt=\"A chain of feeds\"></p>",
      "date_published": "2026-11-15T09:12:00Z"
    }
  ]
}
```

The feed file continues with 28 more items: the rest of `feed.000002.json`, then the newest items of `feed.000001.json`.

`/feed/feed.000002.json`, the newest archive, with the copied feed fields. It holds the two items above, and gets every new item until it holds 20:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Tim's Notes",
  "home_page_url": "https://tim.example.com/",
  "feed_url": "https://tim.example.com/feed/feed.000002.json",
  "description": "Short notes on feeds, the web and everything else.",
  "user_comment": "This is an archive of https://tim.example.com/feed.json, maintained by apps through a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. New items appear in the feed at _pups.updates_url, older ones are in the archive at next_url.",
  "next_url": "https://tim.example.com/feed/feed.000001.json",
  "icon": "https://tim.example.com/feed/images/icons/icon-k3x9.png",
  "favicon": "https://tim.example.com/feed/images/icons/favicon-k3x9.png",
  "authors": [{ "name": "Tim", "url": "https://tim.example.com/" }],
  "language": "en",
  "_pups": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
    "pupsControlled": true,
    "updates_url": "https://tim.example.com/feed.json"
  },
  "_source": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md",
    "accounts": { "socialspec": "@tim.example.com" }
  },
  "items": [
    { "id": "c-Lq7Wm2xP9dRt4yZa", "content_text": "Agreed, archives don't need a CMS.", "date_published": "2026-11-15T10:03:00Z", "_source": { "…": "…" } },
    { "id": "p-8f2kQ9xLm4Tz7wRb", "title": "Archives without a CMS", "content_html": "…", "date_published": "2026-11-15T09:12:00Z" }
  ]
}
```

`/feed/feed.000001.json` looks the same with 20 items and `next_url` → `feed.000000.json`; `/feed/feed.000000.json`, the oldest archive, has no `next_url`.

A named feed `photos` looks the same, with `"feed_url": "https://tim.example.com/feed/photos.json"`, its own title, and its archives and images in `/feed/photos/`.

## Open questions

- **WebSub.** A later version of PUPS should support WebSub: the server would ping a hub whenever a feed file changes, and `hubs` would become a managed field. Not yet.
- **Feed titles in `host-meta.json`.** A client that lists the owner's feeds has to fetch each one to show its title. Should the host-meta endpoint accept a `titles` object for links, as the JSON form of host-meta allows?
- **Names and URIs.** The link relation `https://bigwide.net/ns/feed` is a placeholder, like the URIs of RFC-6, to be published on bigwide.net.
