# RFC: PUPS Feeds - Publishing a Blog Feed with a Personal Upload Server

Nov 15, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-7-Feed-Publishing-with-PUPS.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-7-Feed-Publishing-with-PUPS.md)

## Summary

This proposal describes how apps maintain a blog feed, or any similar stream of posts, on the owner's domain through a [Personal Upload Server (PUPS, RFC-6)](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md). It defines no new protocol, and only one small extension, `_pups`. It is a list of conventions for clients: how to set up a feed, which fields they may change, how to post, edit and delete items, how to move older items into archive feeds, and how to register feeds in `host-meta.json`, so that several apps can share a feed without breaking it.

In short:

- The owner's **main feed** is `/feed.json`, a [JSON Feed 1.1](https://www.jsonfeed.org/version/1.1/) document. Its media and archives live in `/feed/`. Further **named feeds** live in `/feeds/`: the feed `photos` is `/feeds/photos.json`, with its files in `/feeds/photos/`.
- **JSON Feed only.** No RSS, no Atom. An owner who needs RSS needs a real CMS.
- **Clients only write feeds marked as theirs.** Every file they create carries `"_pups": { "pupsControlled": true }`. A feed without it is never overwritten.
- A client **posts** by adding an item to the top of the feed, with a read-modify-write cycle protected by ETags.
- A feed should keep **20 to 30 items**. When it grows beyond its limit, the client moves the oldest 10 into an **archive feed**, `feed.000000.json`, `feed.000001.json` and so on, linked with `next_url`.
- An item that needs a **permanent URL**, e.g. as the source of an RFC-4 webmention, is written into the archive it will end up in right away.
- Every feed is **registered in `host-meta.json`**, so apps and readers can find all of the owner's feeds.
- How a feed becomes a web page is **out of scope**.

**Why now.** PUPS gives apps a place to write the owner's feeds, and says that several apps may write the same feed. It doesn't say what they should write. Without shared conventions, every app would set up the feed differently, overwrite fields set by another app, and let the file grow forever or cut it off where it likes. A feed that one app archives into `feed-page-2.json` and another into `/feed/old/1.json` is no longer one feed. The rules here are few, but every client that writes a feed through PUPS must follow them.

## Goals and non-goals

**Goals**

- Any app that holds a feed's scopes can set up the feed, post to it, and edit or delete posts, without knowing which other apps write it.
- An owner can have several feeds, e.g. a blog, a photo stream and a podcast, and apps can find them.
- A feed that the owner maintains by other means is never touched.
- Feeds stay small enough to fetch often, and older posts stay available in archives.
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
3. **Post.** The client loads the feed, adds the new item at the top and writes it back with `If-Match`.
4. **Archive.** If that makes the feed longer than its limit, the client first moves the oldest 10 items into the newest archive, or into a new one.
5. **Edit and delete.** The client finds the item in the feed or along the archive chain, changes it and writes that file back.

## Feeds and their files

Every feed consists of a **feed file**, the JSON Feed that readers subscribe to, and a **feed directory**, which holds its archives, icons and media.

| | Main feed | Named feed `photos` |
| --- | --- | --- |
| Feed file | `/feed.json` | `/feeds/photos.json` |
| Feed directory | `/feed/` | `/feeds/photos/` |
| Archives | `/feed/feed.000000.json`, … | `/feeds/photos/feed.000000.json`, … |
| Icons | `/feed/images/icons/` | `/feeds/photos/images/icons/` |
| `feed_url` | `https://<owner host>/feed.json` | `https://<owner host>/feeds/photos.json` |
| Scopes | `write:/feed.json write:/feed/` | `write:/feeds/photos.json write:/feeds/photos/` |

`<owner host>` is the host of `me` from the token response. All rules in this document apply to every feed in the same way. Where the text says `/feed.json` and `/feed/`, read `/feeds/<name>.json` and `/feeds/<name>/` for a named feed.

### The main feed

`/feed.json` is the owner's main feed, e.g. their blog. Readers find it on the domain without any discovery. Most owners need nothing else.

### Named feeds

An owner who wants more than one feed, e.g. one for photos and one for a podcast, gives each further feed a **name**.

- A name consists of 1 to 40 lowercase ASCII letters, digits and `-`, and doesn't start or end with `-`: `photos`, `podcast`, `notes-de`.
- The owner chooses the name when the feed is created. The client may suggest one derived from the title. The name is part of every URL of the feed and its archives, so it never changes.
- `/feeds/` itself holds only feed files, `/feeds/<name>.json`, and feed directories, `/feeds/<name>/`. Clients store nothing else there.
- A named feed's `home_page_url` is the domain's root too. Its title, not its name, is what readers show.

## Server requirements

The client needs a PUPS server whose configuration document declares the [well-known paths](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#the-list-of-well-known-paths) of the feeds it writes:

```json
"well_known_paths": [
  { "path": "/feed.json", "url": "https://tim.example.com/feed.json", "rels": ["https://bigwide.net/ns/feed"] },
  { "path": "/feed/", "url": "https://tim.example.com/feed/" },
  { "path": "/feeds/", "url": "https://tim.example.com/feeds/", "rels": ["https://bigwide.net/ns/feed"] }
]
```

- For the main feed, `/feed.json` and `/feed/`. For named feeds, `/feeds/`. Named feeds have no well-known paths of their own; they are files and directories inside `/feeds/`.
- The client requests both scopes of a feed, its file and its directory, and needs both: without the directory it can neither upload media nor archive. If the owner grants only one of them, the client treats the grant as denied and tells the owner why.
- A client that works with one named feed asks for that feed's scopes, `write:/feeds/photos.json write:/feeds/photos/`. Only a client that manages all of the owner's feeds, e.g. a blogging app with several blogs, should ask for `write:/feeds/`. A client with per-feed scopes that wants to create another feed runs a new authorization with the new feed's scopes.
- If the server doesn't declare the paths, e.g. because the domain's website already serves a feed there, the client can't publish that feed. It tells the owner, and may offer to keep a feed in its own directory instead.
- All feed scopes need step-up confirmation (RFC-6).

The client reads files from `storage_base` and writes them there, as RFC-6 requires for ETags. It uses the **public URLs** (based on the `url` of the well-known entries) in everything it writes into a feed: `feed_url`, `next_url`, icons and media.

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
| `next_url` | The newest archive, see [Archives](#archives) | managed |
| `icon`, `favicon` | Images in the feed directory's `images/icons/`, see [Icons](#icons) | owner |
| `authors`, `language`, `expired` | As in JSON Feed | owner |
| `hubs` | **Never set.** PUPS servers don't support WebSub yet. | — |
| `items` | The newest items, newest first, see [Posting](#posting) | managed |
| `_pups` | The PUPS marker, see [The `_pups` object](#the-_pups-object) | fixed |
| other extensions | E.g. `_podcast` ([RFC-3](https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md)), `_source` ([RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md)) | owner |

- **The home page is the domain.** A feed belongs to the domain, not to an app, so `home_page_url` is always the domain's root, never the client's website. What the domain shows there is up to the owner (see [Rendering](#rendering)).
- **Unknown fields are kept.** A client changes only the fields it means to change and writes everything else back unchanged, including fields and extensions it doesn't know, in the feed and in every item. [RFC-6](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#well-known-paths) requires this for all shared well-known files, and it is what allows several clients to share a feed.
- **No `hubs`.** A `hubs` entry without a server that notifies the hub would make readers wait for updates that never come. Clients don't add it, and remove it when the owner asks.

### The `_pups` object

Every file that a client creates under this document, feed files and archives, carries a `_pups` object, following JSON Feed's [extension rules](https://www.jsonfeed.org/version/1.1/#extensions-a-name-extensions-a):

```json
"_pups": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
  "pupsControlled": true
}
```

- `pupsControlled: true` says that the file is maintained by PUPS clients according to this document.
- **A client never overwrites or deletes a feed file or an archive unless it carries `"pupsControlled": true`.** The only files a client writes without the marker are the ones it creates, with `If-None-Match: *`, and those get the marker.
- Clients keep `_pups` unchanged when they write the file.
- `about` should be this document. **This is not its final location**: the URL will change once the spec has a permanent home. Clients therefore ignore the value of `about` and look only at `pupsControlled`.

The marker protects feeds that the owner maintains by hand or with another tool: a client that finds such a feed leaves it alone (see [Setting up a feed](#setting-up-a-feed)). The owner hands a feed over to PUPS clients by adding the marker, and takes it back by removing it.

Items may carry a `_pups` object too, with a single field, `archive`. See [Permanent URLs](#permanent-urls).

### `user_comment`

`user_comment` tells people who open the file how it is maintained. Clients set it to:

> This feed is maintained by apps through a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. Older items are in the archive at next_url.

The text is for humans. Clients recognize a PUPS feed by `_pups`, never by this text.

### Icons

`icon` (512 × 512 pixels) and `favicon` (64 × 64) are optional, chosen by the owner, and stored in `images/icons/` in the feed directory, e.g. `https://tim.example.com/feed/images/icons/icon-k3x9.png`.

- The client lets the owner pick or upload the image. It may offer to crop or scale it, but doesn't create an icon on its own, e.g. from its own logo.
- Every new image gets a **new file name**, so that readers and caches that keep the old URL never show a stale image under a new name. After the feed has been written with the new URL, the client deletes the old file.
- Only icons go into `images/icons/`. Clients don't store anything else there.

## Setting up a feed

After authorization, the client `GET`s the feed file from `storage_base`.

**`404`: no feed yet.** The client offers to create one. It asks the owner for:

- the **name**, for a named feed;
- the **title** (required) and the **description**;
- optionally the icon and favicon, the authors (name, URL, avatar), and the language.

It may offer advanced options, but should hide them by default: for a podcast, `_podcast.blocks` (`podcast:block`, to keep the feed out of podcast directories), `explicit`, `medium`, `locked` and the show's `guid`; for RFC-4, the owner's SocialSpec account in `_source.accounts`. The client says clearly that the feed is public.

It then writes the feed with the fixed fields from the table above, including `_pups`, an empty `items` array or the first post, and `If-None-Match: *`. If that fails with `412`, someone else has created a feed under that name in the meantime; the client loads it and continues as below, or, for a named feed, asks the owner for another name. Finally, it [registers the feed](#registering-feeds-in-host-metajson) in `host-meta.json`.

**`200`: the feed exists.** The client checks that the file is a JSON Feed (a `version` starting with `https://jsonfeed.org/version/1`) and that it carries `"pupsControlled": true`. If either is missing, **the client never writes the file**. It tells the owner that the feed is maintained by other means, and that they can hand it over to PUPS clients as described below. A JSON Feed 1.0 file with the marker is upgraded to 1.1 when it is next written (`author` becomes `authors`).

**Handing over an existing feed.** An owner who wants PUPS clients to take over a feed that was made by hand or by another tool adds the `_pups` object to it, e.g. with a text editor. A client may show the owner the exact lines to add, but never adds them itself. From then on, the feed is treated like any other:

- The first client that writes it sets the fixed fields (`home_page_url`, `feed_url`, `user_comment`) to their values, and registers the feed if it isn't registered yet.
- A file with more items than the limit is brought down to size gradually, 10 items per post, as described in [Posting](#posting).
- An existing `next_url` is kept and becomes the end of the archive chain. The archives it points to are not PUPS archives and are never written, unless the owner marks them too.

## Registering feeds in host-meta.json

PUPS has no directory listing, so there is no other way to learn which feeds a domain has. Every feed should therefore be registered in the domain's `host-meta.json`, with the link relation `https://bigwide.net/ns/feed`, through PUPS's [host-meta endpoint](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#editing-host-metajson):

```json
{ "action": "add", "rel": "https://bigwide.net/ns/feed", "href": "https://tim.example.com/feeds/photos.json", "type": "application/feed+json" }
```

- **When.** The client that creates a feed registers it right afterwards. This includes the main feed, although readers also find it at `/feed.json` without discovery.
- **No duplicates.** Before it registers a feed, the client reads `host-meta.json` and skips the request if a link with this relation and `href` exists already, e.g. because the owner added it by hand.
- **Only feed files.** `href` is always a feed file, `/feed.json` or `/feeds/<name>.json`, never an archive or another file in a feed directory.
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
      "rel": "https://bigwide.net/ns/feed", "type": "application/feed+json", "href": "https://tim.example.com/feeds/photos.json",
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

- **`id`** is unique across the feed and all its archives, and **never changes**, also not when the item moves into an archive. Clients should use a random string of at least 16 characters from `A–Z`, `a–z`, `0–9`, `-` and `_`, e.g. `p-8f2kQ9xLm4Tz7wRb`, so that the id can be used in a URL fragment without encoding (RFC-4 references items as feed URL + `#id`).
- **`date_published`** is always set. `date_modified` is set whenever the item is edited.
- **Content.** JSON Feed requires `content_html` or `content_text`. For `content_html`, clients should consider [Feed HTML (RFC-2)](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md): HTML within that subset looks the same in every reader that supports it, anything beyond it may be dropped. Markdown is an alternative: the client writes the text as Markdown in `_source.markdown` ([RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md#item-markdown)), and the same text as `content_html` or `content_text` for readers that don't know the extension.
- **`url`** is the item's web page. The client sets it only if it knows that the page exists, e.g. because the owner entered it; it never invents one. `external_url` can be used for link posts.
- **Media** (images, podcast episodes, other attachments) go into the feed directory, under the PUPS file-type allowlist, and are referenced by their absolute public URL. Clients organize the directory as they like, except for the names reserved below; `images/` and `media/` are recommended. Like icons, media files get new names when they change, and are never overwritten.
- **Order.** `items` is sorted newest first: a new item is inserted at index 0. Clients never reorder items, also not by `date_published`. Position decides what is archived.
- **Who may change an item.** Any client may edit or delete any item, whichever client created it, but only when the owner asks it to. It keeps all fields of the item that it doesn't change. Moving an item into an archive doesn't change it, so every client archives every item.

**Reserved names in a feed directory**: the archive files `feed.<NNNNNN>.json`, and the directory `images/icons/`. Clients store nothing else under these names.

## Posting

Posting is a read-modify-write of the feed file, with ETags so that concurrent writes by another client, or by the owner, are never lost ([RFC-6 Concurrency](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#concurrency)):

1. `GET` the feed file from `storage_base` and remember its `ETag`. If it doesn't carry `"pupsControlled": true`, stop.
2. Insert the new item at the top of `items`.
3. If `items` now has **30 items or fewer**, `PUT` the file with `If-Match: "<etag>"`. Done.
4. Otherwise, take the **oldest 10 items** (the last 10 in `items`) out of the feed and move them into an archive, as described in [Archives](#archives). This may change `next_url`.
5. `PUT` the feed file with `If-Match: "<etag>"`.

If any write fails with `412`, the client starts again at step 1. It uploads the item's media files before step 1, so that the item never references a file that doesn't exist yet.

**Target size.** The recommended target size is 20 to 30 items: the feed has at most 30 items, and archiving brings it back to about 20. It is not fixed. A client may use other limits, e.g. 50 and 40 for a feed with short posts, or let the owner configure them; this document uses 30 items and batches of 10 throughout. Clients that use different limits on the same feed don't break anything: each applies its own when it posts. A feed that is much larger than the limit, e.g. after a hand-over, shrinks by one batch per post.

**Size in bytes (optional).** Items can be large, e.g. long posts with full HTML. A client may also move a batch when the feed file would be larger than a size limit it chooses, e.g. 256 KB, even though it has fewer items than its item limit. It still moves 10 items at a time, and never leaves fewer than 10 in the feed.

## Archives

Archives hold the items that no longer fit into the feed file. They are JSON Feeds too, stored in the feed directory and linked into a chain with `next_url`, from the newest to the oldest:

```
/feed.json  →  /feed/feed.000002.json  →  /feed/feed.000001.json  →  /feed/feed.000000.json
```

- **Names.** Archives are called `feed.<NNNNNN>.json`, with a six-digit number starting at `000000`. Each new archive gets the next number, so the newest archive has the highest one. The name is the same for every feed; the feed directory tells them apart: `/feeds/photos/feed.000000.json`.
- **Links.** `next_url` of the feed file points to the newest archive. `next_url` of each archive points to the next older one. The oldest archive has no `next_url`, unless the chain continues into older archives of a feed that was handed over.
- **Size.** An archive holds at most 30 items, like the feed file. Items are moved in batches of 10, so an archive normally fills up in three steps.
- **Feed fields.** A new archive gets a copy of the top-level fields of the feed file, including `_pups`, with these exceptions: `user_comment` says that it is an archive, `next_url` points to the next older archive, `items` holds the moved items, and `_podcast.liveItems` is left out. `feed_url` stays the URL of the feed file, so a reader that subscribes to an archive by mistake ends up with the live feed.
- **Marker.** Like the feed file, an archive is written only if it carries `"pupsControlled": true`.

The archive's `user_comment`:

> This is an archive of https://tim.example.com/feed.json, maintained by apps through a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. Newer items are in the feed at feed_url, older ones in the archive at next_url.

### Moving items into an archive

Step 4 of [Posting](#posting) moves a batch of items, the oldest 10, out of the feed file:

1. **Find the newest archive.** If `next_url` of the feed file names an archive file `feed.<N>.json` in the feed directory, `GET` it from `storage_base` and remember its `ETag`.
2. **Add to it if it has room.** If the archive carries the marker and would have 30 items or fewer with the batch added, insert the batch at the top of its `items`, in the same order, and `PUT` it with `If-Match`. `next_url` of the feed file stays unchanged.
3. **Otherwise create a new archive**, number `N + 1`, or `000000` if the feed file has no `next_url`, or one that doesn't name an archive file. Its `next_url` is the current `next_url` of the feed file, i.e. the archive before it, if there is one. `PUT` it with `If-None-Match: *`, and set `next_url` of the feed file to its public URL.

**Pre-archived items.** An item in the batch that carries `_pups.archive` goes into that archive instead (see [Permanent URLs](#permanent-urls)). Normally that is the archive chosen above. If not, the client writes the item into its own archive as well, replacing the copy there, and doesn't count it towards the size of the other one.

Then continue with step 5 of Posting, writing the feed file. The order matters: **archives are always written before the feed file**. If the client is interrupted between the writes, the items are in both files for a while, which is harmless, but they are never in neither.

Example: `/feed.json` has 30 items and `next_url` → `feed.000001.json`, which holds 30 items. A new post makes 31. The client takes the oldest 10, finds no room in `feed.000001.json`, creates `feed.000002.json` with these 10 items and `next_url` → `feed.000001.json`, then writes `/feed.json` with 21 items and `next_url` → `feed.000002.json`.

### Interrupted and concurrent writes

Because a post may write several files, the client has to deal with writes that only half happened:

- **Duplicates.** When the client adds items to an archive, an item whose `id` is already there replaces the old copy instead of being added a second time, and doesn't count towards the size limit. This cleans up after an interrupted move and after a retry, and it is what makes [pre-archived](#permanent-urls) items join their archive without taking up room twice.
- **`412` on the archive.** Another client has changed it. The client starts the whole post again at step 1.
- **`412` when creating an archive.** The file exists already: created ahead of time for a permanent URL, left behind by an interrupted client, or created by another client at the same moment. The client loads it. If it carries the marker, its `next_url` is the expected one and the batch fits, the client adds the batch as in step 2. Otherwise it reloads the feed file: if that has changed, it starts the whole post again at step 1; if not, it tries the next number.
- **`412` on the feed file** after the archive was written. The client starts again at step 1. The batch may now be in the archive and still in the feed file; the duplicate rule takes care of it.
- **A missing archive.** If the archive that `next_url` names returns `404`, e.g. because the owner deleted it, the client creates it again under the same name, with `If-None-Match: *`. Its `next_url` points to the archive with the next lower number, if there is one.

Readers may see an item in the feed file and in an archive at the same time. JSON Feed readers identify items by `id`, so they show it once.

## Permanent URLs

An item in the feed file is addressed as, e.g., `https://<owner host>/feed.json#<id>`, but only until it is archived. Some uses need a URL that never changes, most importantly the `source` of an [RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) webmention, which the client sends again whenever it edits or deletes the item.

Archiving is deterministic, so the client can tell which archive an item will end up in. It **writes the item into that archive right away**, and uses the archive's URL as the item's permanent URL:

1. Compute the item's archive, as described below.
2. Set `_pups.archive` on the item to the archive's public URL: `"_pups": { "archive": "https://tim.example.com/feed/feed.000002.json" }`. Once set, it never changes.
3. Write the item into that archive. If the archive exists, add the item at the top of its `items` and `PUT` it with `If-Match`. If not, create it with `If-None-Match: *`, as a [new archive](#archives) with the item as its only entry, and `next_url` pointing to the archive with the next lower number, if there is one, even if that one doesn't exist yet either.
4. Write the item into the feed file, as in [Posting](#posting).
5. Use the archive's URL + `#<id>` as the permanent URL, e.g. as the webmention's `source`.

An archive created ahead of time is not yet linked from the feed file; readers following the chain don't see it until regular archiving reaches it and links it (see the `412` rule in [Interrupted and concurrent writes](#interrupted-and-concurrent-writes)). The RFC-4 server doesn't follow the chain; it fetches the archive directly.

**Computing the archive.** With limits of 30 items and batches of 10 (a client with other limits uses its own numbers):

- `p` is the item's position in the feed file, counted from the oldest item, starting at 1. For a new post, it is the number of items including the new one.
- The item will be moved in batch `j = floor((p − 1) / 10)`, counting from 0 for the next batch to be moved.
- `N` is the number of the newest archive, the one `next_url` names, and `k` the number of items in it that are not also in the feed file. Without an archive, `N = −1` and `k = 30`.
- `r = floor((30 − k) / 10)` batches still fit into archive `N`.
- If `j < r`, the item ends up in archive `N`. Otherwise it ends up in archive `N + 1 + floor((j − r) / 3)`.

Example: `/feed.json` has 24 items and `next_url` → `feed.000001.json`, which holds 10. A new comment makes 25, so `p = 25` and `j = 2`. Archive 1 has room for `r = 2` more batches, batches 0 and 1. Batch 2 starts archive 2. The client writes the comment into `/feed/feed.000002.json`, then into `/feed.json`, and sends the webmention with `source` = `https://tim.example.com/feed/feed.000002.json#c-Lq7Wm2xP9dRt4yZa`.

The computation assumes that nothing else changes until the item is archived. Deleted items, other clients' limits, archiving by size or the owner's own edits can make it wrong. That doesn't break the permanent URL: an item with `_pups.archive` always goes into that archive when it is moved, so the URL stays valid. The archive may then end up a little larger than 30 items, or slightly out of order with its neighbours.

A client that needs a permanent URL for an item that is already archived uses that archive. For an item that is still in the feed file, it follows the steps above with the item's current position.

## Editing and deleting items

**Editing.** The client looks for the item by `id`: first in the feed file, then along `next_url` from archive to archive, and stops at the first file that contains it. It changes the item, sets `date_modified` and `PUT`s the file with `If-Match`. On `412`, it searches again from the feed file, because the item may have been archived in the meantime. If the item is still in the feed file and carries `_pups.archive`, the client changes the copy in that archive first, then the one in the feed file.

**Deleting.** The client removes the item in the same way, from every copy: the one in the feed file, the one in its `_pups.archive`, and, because an interrupted move can leave a copy behind, the one in the newest archive. Then it deletes the item's media files from the feed directory, unless another item still uses them.

Archives are never rebalanced. An archive that shrinks after deletions keeps its size, and an empty archive keeps its place in the chain; clients don't delete archive files.

## Reading

Nothing in this document is new for readers. A feed file is an ordinary JSON Feed with `next_url` pagination, as JSON Feed 1.1 describes it, and readers that don't follow `next_url` simply see the newest items. A reader that wants the whole history follows `next_url` until it ends. Readers that don't know `_pups` ignore it, as JSON Feed requires.

Readers find the main feed at `https://<owner host>/feed.json` without any discovery, and all feeds of a domain in `host-meta.json`. A website or profile that wants to point to a feed links to its feed file.

## Rendering

This document doesn't say how a feed becomes a web page, and the server never produces HTML. Possible ways:

- A **static site generator**, running on the owner's server or elsewhere, that rebuilds the site whenever a feed file or an archive changes.
- A **static page** at `/index.html`, uploaded by the owner, that fetches `/feed.json` in the browser and renders it, following `next_url` for older posts. The feed is served with `Access-Control-Allow-Origin: *`, so the page could even be hosted elsewhere.
- **No page at all.** The feeds are meant for readers, and the domain's home page shows something else.

Whatever is used, it is not a client of this document: it only reads the files. Items that have a page of their own may name it in `url`, if the client knows it.

## Using a feed for Feed Submissions (RFC-4)

The main feed, or a named feed, can be the author feed of [RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) posts and comments, as RFC-6 mentions. The owner's readers then see the comment in the feed.

1. The client gives the item `_source.inReplyTo` as usual, and the feed carries the owner's SocialSpec account in `_source.accounts`.
2. It writes the item with a [permanent URL](#permanent-urls): first into the archive it will end up in, then into the feed file.
3. It sends the webmention with `source` = the archive's URL + `#<id>`, e.g. `https://tim.example.com/feed/feed.000002.json#c-Lq7Wm2xP9dRt4yZa`.
4. To edit or delete the comment, it changes or removes the item as described in [Editing and deleting items](#editing-and-deleting-items) and resends the webmention with the same `source`.

The archive is on the owner's host, so RFC-4's `requireSocialSpec` check works for it as for the feed file.

## Example

`/feed.json` with two archives, and a comment that already has its permanent URL in the not yet linked archive `feed.000002.json`:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Tim's Notes",
  "home_page_url": "https://tim.example.com/",
  "feed_url": "https://tim.example.com/feed.json",
  "description": "Short notes on feeds, the web and everything else.",
  "user_comment": "This feed is maintained by apps through a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. Older items are in the archive at next_url.",
  "next_url": "https://tim.example.com/feed/feed.000001.json",
  "icon": "https://tim.example.com/feed/images/icons/icon-k3x9.png",
  "favicon": "https://tim.example.com/feed/images/icons/favicon-k3x9.png",
  "authors": [{ "name": "Tim", "url": "https://tim.example.com/" }],
  "language": "en",
  "_pups": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
    "pupsControlled": true
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
      "_pups": { "archive": "https://tim.example.com/feed/feed.000002.json" }
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

`/feed/feed.000001.json`, the newest linked archive, with the copied feed fields:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Tim's Notes",
  "home_page_url": "https://tim.example.com/",
  "feed_url": "https://tim.example.com/feed.json",
  "description": "Short notes on feeds, the web and everything else.",
  "user_comment": "This is an archive of https://tim.example.com/feed.json, maintained by apps through a Personal Upload Server (PUPS), following https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md. Newer items are in the feed at feed_url, older ones in the archive at next_url.",
  "next_url": "https://tim.example.com/feed/feed.000000.json",
  "icon": "https://tim.example.com/feed/images/icons/icon-k3x9.png",
  "favicon": "https://tim.example.com/feed/images/icons/favicon-k3x9.png",
  "authors": [{ "name": "Tim", "url": "https://tim.example.com/" }],
  "language": "en",
  "_pups": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md",
    "pupsControlled": true
  },
  "items": [
    { "id": "p-T5mWc2hJd9Ue4oPz", "content_text": "…", "date_published": "2026-10-02T07:30:00Z" }
  ]
}
```

`/feed/feed.000000.json`, the oldest archive, looks the same, without `next_url`. `/feed/feed.000002.json` looks the same too, with `next_url` → `feed.000001.json` and the comment as its only item, until regular archiving fills it and links it from `/feed.json`.

A named feed `photos` looks the same, with `"feed_url": "https://tim.example.com/feeds/photos.json"`, its own title, and its archives and images in `/feeds/photos/`.

## Open questions

- **WebSub.** A later version of PUPS should support WebSub: the server would ping a hub whenever a feed file changes, and `hubs` would become a managed field. Not yet.
- **Feed titles in `host-meta.json`.** A client that lists the owner's feeds has to fetch each one to show its title. Should the host-meta endpoint accept a `titles` object for links, as the JSON form of host-meta allows?
- **Names and URIs.** The link relation `https://bigwide.net/ns/feed` is a placeholder, like the URIs of RFC-6, to be published on bigwide.net.
