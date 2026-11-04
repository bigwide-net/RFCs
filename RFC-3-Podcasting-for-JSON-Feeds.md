# RFC: Podcast Namespace Extensions for JSON Feed

Nov 1, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-3-Podcasting-for-JSON-Feeds.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-3-Podcasting-for-JSON-Feeds.md)

## Summary

This proposal brings every element of the [Podcasting 2.0 namespace](https://github.com/Podcastindex-org/podcast-namespace/blob/main/docs/1.0.md) (`https://podcastindex.org/namespace/1.0`) to [JSON Feed](https://www.jsonfeed.org/version/1.1/): transcripts, chapters, people, seasons, alternate enclosures, value-for-value, live streams, podrolls and the rest. They all live in one extension object, `_podcast`, which mirrors the RSS tags one to one. A reader can therefore treat a JSON podcast exactly like an RSS podcast that uses the namespace.

It also adds the three iTunes fields that podcast apps need and JSON Feed lacks: `explicit`, `showType` and `episodeType`. Together with what JSON Feed already has (attachments with a duration, images, authors), a podcast no longer needs RSS.

**Why now.** JSON Feed was designed with podcasts in mind: `attachments` carry the audio file, its size and its duration. But a modern podcast feed is more than an enclosure. Apps show transcripts, chapters, hosts and guests, funding links and video versions, all of them defined by the podcast namespace, and none of them can be expressed in JSON Feed today. A publisher who wants them has to stay on RSS. Mirroring the namespace keeps a single model for both formats: publishers generate both feeds from the same data, readers use one code path, and every future namespace tag has an obvious JSON form.

## The `_podcast` object

Everything goes into one object named `_podcast`, following JSON Feed's [extension rules](https://www.jsonfeed.org/version/1.1/#extensions-a-name-extensions-a): the name starts with an underscore, and the object says which spec it implements in `about`.

```json
"_podcast": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md"
}
```

- It may appear on the **feed** (show-level fields), on an **item** (episode-level fields) and on a **live item** (see [Live items](#live-items)).
- `about` should be `https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md`, this document. **This is not its final location**: the URL will change once the spec has a permanent home. Readers must therefore **ignore the value of `about`**: they recognize the extension by the name `_podcast` alone, and read its fields whatever `about` says, or when it is missing.
- Every field is optional. A reader ignores fields it doesn't know and values of the wrong type, so the object can grow along with the namespace.

**Meaning comes from the namespace.** This document defines the JSON *shape* of each tag, nothing else. What a field means, which values it allows and how apps should use it is what the [namespace's tag documentation](https://github.com/Podcastindex-org/podcast-namespace/blob/main/docs/1.0.md) says about the tag it mirrors, including its override rules: episode-level `persons` replace the show's, episode-level `values` and `chat` override the show's.

## From tags to fields

Every field follows from its RSS tag by the same rules, so a converter needs no special cases beyond the table below:

1. **Name.** The field is named after the tag without the `podcast:` prefix: `<podcast:updateFrequency>` becomes `updateFrequency`.
2. **Repeating tags become arrays** with a plural name: `<podcast:transcript>` (multiple) becomes `transcripts: [ … ]`. `funding` stays `funding`. Tags that occur once become a single object.
3. **Attributes become fields** with the same name: `startTime`, `feedGuid`, `accountId`. The one name that is not a JSON identifier, `aspect-ratio`, becomes `aspectRatio`.
4. **The node value becomes a named field**, listed in the tables: `<podcast:person>Jane Doe</podcast:person>` becomes `{ "name": "Jane Doe" }`.
5. **Child elements become nested fields**: the `<podcast:source>` children of an alternate enclosure become its `sources` array.
6. **Types are JSON's.** Numbers are JSON numbers (`"split": 40`, not `"40"`), booleans are `true`/`false` (also for the namespace's `yes`/`no`), dates are [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) strings like JSON Feed's `date_published`. Everything else is a string.

Readers should be lenient where converters are likely to slip: accept numeric strings for numbers, `"true"`/`"false"`/`"yes"`/`"no"` for booleans, and a single object where an array is expected.

## Feed fields

On the feed's `_podcast` object (the RSS `<channel>`):

| Field | Type | RSS | Node value / notes |
| --- | --- | --- | --- |
| `guid` | string | `<podcast:guid>` | The UUIDv5 of the show. |
| `medium` | string | `<podcast:medium>` | `podcast`, `music`, `video`, `film`, `audiobook`, `newsletter`, `blog`, `publisher`, `course`, their list forms (`musicL`, …) or `mixed`. |
| `locked` | object | `<podcast:locked>` | `{ "value": true, "owner": "email@example.com" }`. `value` is the node value. |
| `blocks` | array | `<podcast:block>` | `{ "value": true, "id": "google" }`. `value` is the node value; `id` is optional. |
| `funding` | array | `<podcast:funding>` | `{ "url", "title" }`. `title` is the link text. |
| `persons` | array | `<podcast:person>` | `{ "name", "role", "group", "img", "href" }`. `name` is the node value. |
| `locations` | array | `<podcast:location>` | `{ "name", "rel", "geo", "osm", "country" }`. `name` is the node value. |
| `trailers` | array | `<podcast:trailer>` | `{ "title", "url", "pubdate", "length", "type", "season" }`. `title` is the node value; `pubdate` is RFC 3339. |
| `license` | object | `<podcast:license>` | `{ "identifier", "url" }`. `identifier` is the node value, e.g. `cc-by-4.0`. |
| `values` | array | `<podcast:value>` | See [Value](#value). |
| `images` | array | `<podcast:image>` | `{ "href", "alt", "aspectRatio", "width", "height", "type", "purpose" }`. |
| `socialInteracts` | array | `<podcast:socialInteract>` | `{ "protocol", "uri", "accountId", "accountUrl", "priority" }`. |
| `chat` | object | `<podcast:chat>` | `{ "server", "protocol", "accountId", "space" }`. |
| `txts` | array | `<podcast:txt>` | `{ "purpose", "value" }`. `value` is the node value. |
| `updateFrequency` | object | `<podcast:updateFrequency>` | `{ "title", "complete", "dtstart", "rrule" }`. `title` is the node value. |
| `podping` | object | `<podcast:podping>` | `{ "usesPodping": true }`. |
| `podroll` | array | `<podcast:podroll>` | The podroll's [remote items](#remote-items). |
| `publisher` | object | `<podcast:publisher>` | The publisher feed's [remote item](#remote-items). |
| `remoteItems` | array | `<podcast:remoteItem>` | For list feeds (`musicL`, `mixed`, …). See [Remote items](#remote-items). |
| `liveItems` | array | `<podcast:liveItem>` | See [Live items](#live-items). |
| `explicit` | boolean | `<itunes:explicit>` | See [iTunes fields](#itunes-fields). |
| `showType` | string | `<itunes:type>` | `episodic` or `serial`. |

```json
"_podcast": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md",
  "guid": "917393e3-1b1e-5cef-ace4-edaa54e1f810",
  "medium": "podcast",
  "locked": { "value": true, "owner": "podcast@example.com" },
  "funding": [{ "url": "https://example.com/donate", "title": "Support the show!" }],
  "persons": [
    { "name": "Terry Scott", "href": "https://example.com/terry", "img": "https://example.com/terry.jpg" },
    { "name": "June Whitfield" }
  ],
  "locations": [{ "name": "Austin", "rel": "creator", "geo": "geo:30.2711286,-97.7436995", "osm": "R113314", "country": "US" }],
  "updateFrequency": { "title": "Weekly", "rrule": "FREQ=WEEKLY" },
  "explicit": false,
  "showType": "episodic"
}
```

## Item fields

On an item's `_podcast` object (the RSS `<item>`):

| Field | Type | RSS | Node value / notes |
| --- | --- | --- | --- |
| `transcripts` | array | `<podcast:transcript>` | `{ "url", "type", "language", "rel" }`. |
| `chapters` | object | `<podcast:chapters>` | `{ "url", "type" }`. `type` is usually `application/json+chapters`. |
| `soundbites` | array | `<podcast:soundbite>` | `{ "startTime", "duration", "title" }`. Times in seconds; `title` is the node value. |
| `persons` | array | `<podcast:person>` | As on the feed. Replaces the show's list for this episode. |
| `locations` | array | `<podcast:location>` | As on the feed. |
| `season` | object | `<podcast:season>` | `{ "number": 3, "name": "Race for the Whitehouse 2020" }`. `number` is the node value. |
| `episode` | object | `<podcast:episode>` | `{ "number": 204, "display": "Ch.3" }`. `number` is the node value and may be decimal. |
| `license` | object | `<podcast:license>` | As on the feed. |
| `alternateEnclosures` | array | `<podcast:alternateEnclosure>` | See [Alternate enclosures](#alternate-enclosures). |
| `values` | array | `<podcast:value>` | See [Value](#value). Overrides the show's. |
| `funding` | array | `<podcast:funding>` | As on the feed. |
| `images` | array | `<podcast:image>` | As on the feed. |
| `contentLinks` | array | `<podcast:contentLink>` | `{ "href", "title" }`. `title` is the node value. |
| `socialInteracts` | array | `<podcast:socialInteract>` | As on the feed. |
| `chat` | object | `<podcast:chat>` | As on the feed. Overrides the show's. |
| `txts` | array | `<podcast:txt>` | As on the feed. |
| `explicit` | boolean | `<itunes:explicit>` | See [iTunes fields](#itunes-fields). |
| `episodeType` | string | `<itunes:episodeType>` | `full`, `trailer` or `bonus`. |

```json
{
  "id": "https://example.com/episodes/42",
  "url": "https://example.com/episodes/42",
  "title": "Why the Podcast Namespace Matters",
  "content_html": "<p>Our guest explains it all.</p>",
  "date_published": "2026-11-01T08:00:00Z",
  "attachments": [
    { "url": "https://example.com/ep42.mp3", "mime_type": "audio/mpeg", "size_in_bytes": 43200000, "duration_in_seconds": 3600 }
  ],
  "_podcast": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md",
    "season": { "number": 2 },
    "episode": { "number": 42 },
    "episodeType": "full",
    "transcripts": [
      { "url": "https://example.com/ep42.vtt", "type": "text/vtt", "rel": "captions" },
      { "url": "https://example.com/ep42.html", "type": "text/html" }
    ],
    "chapters": { "url": "https://example.com/ep42-chapters.json", "type": "application/json+chapters" },
    "soundbites": [{ "startTime": 1234.5, "duration": 42.25, "title": "The key argument" }],
    "persons": [
      { "name": "Terry Scott" },
      { "name": "Jane Doe", "role": "guest", "href": "https://example.com/jane" }
    ]
  }
}
```

## Alternate enclosures

`alternateEnclosures` mirrors `<podcast:alternateEnclosure>`: other versions of the episode's media, such as a video version, other bitrates or codecs, or other transports. Its `<podcast:source>` children become `sources` (at least one), its optional `<podcast:integrity>` becomes `integrity`:

```json
"alternateEnclosures": [
  {
    "type": "video/mp4",
    "length": 10562995,
    "bitrate": 681483.55,
    "height": 1080,
    "title": "Video",
    "sources": [
      { "uri": "https://example.com/file-1080.mp4" },
      { "uri": "https://example.com/file-1080.torrent", "contentType": "application/x-bittorrent" },
      { "uri": "ipfs://QmfQKJcp2xdByEt8mzWr1AJUhwvb9rdWPoacvdq2roDhgh" }
    ],
    "integrity": { "type": "sri", "value": "sha384-ExVqijgYHm15PqQqdXfW95x+Rs6C+d6E/ICxyQOeFevnxNLR/wtJNrNYTjIysUBo" }
  },
  {
    "type": "application/x-mpegURL",
    "length": 0,
    "height": 1080,
    "rel": "alternate",
    "sources": [{ "uri": "https://example.com/hls/master.m3u8" }]
  }
]
```

Fields of an alternate enclosure: `type`, `length`, `bitrate`, `height`, `lang`, `title`, `rel`, `codecs`, `default` (boolean), `sources` and `integrity`. Fields of a source: `uri`, `contentType`. Fields of `integrity`: `type` (`sri` or `pgp-signature`), `value`.

The episode's main media stays in JSON Feed's `attachments`, just as the RSS `<enclosure>` stays next to `<podcast:alternateEnclosure>`. Alternate enclosures are for readers that understand them; others play the attachment.

## Value

`values` mirrors `<podcast:value>` blocks. The `<podcast:valueRecipient>` children become `recipients`, the `<podcast:valueTimeSplit>` children become `timeSplits`:

```json
"values": [
  {
    "type": "lightning",
    "method": "keysend",
    "suggested": "0.00000005000",
    "recipients": [
      { "name": "Alice (Podcaster)", "type": "node", "address": "02d5c1bf8b940dc9cadca86d1b0a3c37fbe39cee4c7e839e33bef9174531d27f52", "split": 95 },
      { "name": "Hosting Provider", "type": "node", "address": "03ae9f91a0cb8ff43840e3c322c4c61f019d8c1c3cea15a25cfc425ac605e61a4a", "split": 5, "fee": true }
    ],
    "timeSplits": [
      {
        "startTime": 60,
        "duration": 237,
        "remotePercentage": 95,
        "remoteItem": { "feedGuid": "a94f5cc9-8c58-55fc-91fe-a324087a655b", "itemGuid": "https://podcastindex.org/podcast/4148683#1", "medium": "music" }
      },
      {
        "startTime": 367,
        "duration": 246,
        "recipients": [
          { "name": "Bobjim (Guest)", "type": "node", "address": "032f4ffbbafffbe51726ad3c164a3d0d37ec27bc67b29a159b0f49ae8ac21b8508", "split": 10 }
        ]
      }
    ]
  }
]
```

- A recipient has `type`, `address`, `split`, `name`, `customKey`, `customValue` and `fee` (boolean).
- A time split has `startTime`, `duration`, `remoteStartTime`, `remotePercentage`, and either `recipients` or a single `remoteItem`, like the RSS element.
- `suggested` stays a **string**: it is an amount of cryptocurrency with many decimal places, and a JSON number could lose precision on the way through a parser. It is the one exception to rule 6.

## Remote items

`<podcast:remoteItem>` points to another feed, or to an item in another feed. In JSON it is an object with the same attributes: `feedGuid` (required), `feedUrl`, `itemGuid`, `medium` and `title`. It appears in four places:

| Where | JSON | RSS |
| --- | --- | --- |
| Podroll ("shows you might like") | feed `podroll`: array of remote items | `<podcast:podroll>` with `<podcast:remoteItem>` children |
| Publisher feed of this show | feed `publisher`: one remote item | `<podcast:publisher>` with one `<podcast:remoteItem medium="publisher">` |
| List feeds (`musicL`, `podcastL`, `mixed`, …) | feed `remoteItems`: array of remote items | `<podcast:remoteItem>` in `<channel>` |
| Value time splits | `values[].timeSplits[].remoteItem` | `<podcast:remoteItem>` in `<podcast:valueTimeSplit>` |

`podroll` and `publisher` take the remote items directly because their RSS elements have no attributes of their own; they only wrap. In `publisher`, `medium` may be left out; it is always `publisher`.

```json
"_podcast": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md",
  "podroll": [
    { "feedGuid": "29cdca4a-32d8-56ba-b48b-09a011c5daa9", "feedUrl": "https://rss.buzzsprout.com/231452.rss", "title": "Buzzcast" },
    { "feedGuid": "917393e3-1b1e-5cef-ace4-edaa54e1f810", "feedUrl": "https://feeds.podcastindex.org/pc20.xml", "title": "Podcasting 2.0" }
  ],
  "publisher": { "feedGuid": "003af0a0-6a45-55cf-b765-68e3d349551a", "feedUrl": "https://example.org/publisher.json" }
}
```

A list feed has no regular items of its own. JSON Feed requires the `items` array, so it is present and empty: `"items": []`.

## Live items

`<podcast:liveItem>` is an item for a live stream. In JSON Feed, live items go into the feed's `_podcast.liveItems` array, **not** into `items`. Each one is a complete JSON Feed item, with every item field this document defines. The live item's own attributes go into its `_podcast.live` object:

| Field | Type | Meaning |
| --- | --- | --- |
| `live.status` | string | `pending`, `live` or `ended`. Required. |
| `live.start` | RFC 3339 date | When the stream is meant to start. Required. |
| `live.end` | RFC 3339 date | When the stream is meant to end. Recommended. |

```json
"_podcast": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md",
  "liveItems": [
    {
      "id": "https://example.com/live",
      "url": "https://example.com/podcast/live",
      "title": "Podcasting 2.0 Live Show",
      "content_text": "A look into the future of podcasting.",
      "attachments": [{ "url": "https://example.com/pc20/livestream?format=.mp3", "mime_type": "audio/mpeg" }],
      "_podcast": {
        "live": { "status": "live", "start": "2026-11-01T07:30:00-06:00", "end": "2026-11-01T09:30:00-06:00" },
        "alternateEnclosures": [
          { "type": "audio/mpeg", "default": true, "sources": [{ "uri": "https://example.com/pc20/livestream" }] }
        ],
        "contentLinks": [
          { "href": "https://example.com/html/livestream", "title": "Listen Live!" },
          { "href": "https://www.youtube.com/pc20/livestream", "title": "YouTube!" }
        ],
        "chat": { "server": "irc.zeronode.net", "protocol": "irc", "space": "#pc20" }
      }
    }
  ]
}
```

**Why not in `items`.** A reader that does not know `_podcast` would show a live item in `items` as an ordinary post: an episode with a stream URL that plays nothing yet (`pending`), or nothing anymore (`ended`). In RSS, `<podcast:liveItem>` is a separate element for the same reason, and readers that don't know it never see it. As in RSS, every live item needs a `contentLinks` entry as the fallback for apps that cannot play the stream.

## iTunes fields

Three iTunes elements have no JSON Feed counterpart, and podcast apps rely on them. They go into `_podcast` too:

| Field | Where | Type | RSS | Meaning |
| --- | --- | --- | --- | --- |
| `explicit` | feed, item | boolean | `<itunes:explicit>` | Whether the show or episode contains explicit content. |
| `showType` | feed | string | `<itunes:type>` | `episodic` (newest first, the default) or `serial` (to be heard in order, oldest first). |
| `episodeType` | item | string | `<itunes:episodeType>` | `full` (the default), `trailer` or `bonus`. |

The other iTunes elements already have a JSON Feed field, or a namespace field above:

| iTunes / RSS | JSON Feed |
| --- | --- |
| `<enclosure>` | the item's first audio or video `attachments` entry (`url`, `mime_type`, `size_in_bytes`) |
| `<itunes:duration>` | `duration_in_seconds` on that attachment |
| `<itunes:image>` (show) | `icon`, or a `_podcast.images` entry with `purpose: "artwork"` for full-size art |
| `<itunes:image>` (episode) | the item's `image` |
| `<itunes:author>` | `authors` |
| `<itunes:title>`, `<itunes:summary>` | `title`, `summary` |
| `<itunes:season>`, `<itunes:episode>` | `_podcast.season.number`, `_podcast.episode.number` |
| `<itunes:block>` | `_podcast.blocks` |
| `<itunes:complete>` | `_podcast.updateFrequency.complete` |

## Notes for publishers and readers

**Publishers**

- Write `_podcast` only when it has at least one field. Use absolute URLs; the namespace requires `https:` for every web URL (media `sources` may use other schemes, such as `ipfs:`).
- If you also publish RSS, write the same values there. The two feeds of a show should never disagree.
- Give the main audio as an attachment with `duration_in_seconds`, even when you also give alternate enclosures. It is what every JSON Feed reader can play.
- Limits from the namespace still apply: 128 characters for display strings such as funding and soundbite titles, person and location names; 32 for an episode's `display`; 4000 for a `txt` value.

**Readers**

- Feeds are untrusted. Drop URLs with schemes you cannot handle, numbers that are negative or not finite, and strings far beyond the namespace's limits.
- A reader can convert `_podcast` into its existing RSS data model by applying the rules in [From tags to fields](#from-tags-to-fields) backwards. That way, all of its podcast features work for both formats with one code path.
- Transcript, chapter and comment files often live on other hosts and send no CORS headers, so browser-based readers need a fallback (a proxy).

## Full example

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Example Podcast",
  "home_page_url": "https://example.com/",
  "feed_url": "https://example.com/podcast.json",
  "description": "A show about feeds.",
  "icon": "https://example.com/cover-512.jpg",
  "language": "en",
  "authors": [{ "name": "Terry Scott", "url": "https://example.com/terry" }],
  "_podcast": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md",
    "guid": "917393e3-1b1e-5cef-ace4-edaa54e1f810",
    "medium": "podcast",
    "explicit": false,
    "showType": "serial",
    "images": [{ "href": "https://example.com/cover-3000.jpg", "purpose": "artwork", "aspectRatio": "1/1", "width": 3000 }],
    "funding": [{ "url": "https://example.com/donate", "title": "Support the show!" }],
    "persons": [{ "name": "Terry Scott", "role": "host" }],
    "license": { "identifier": "cc-by-4.0" },
    "trailers": [{ "title": "Coming soon", "url": "https://example.com/trailer.mp3", "pubdate": "2026-10-01T08:00:00Z", "type": "audio/mpeg", "length": 1234567 }],
    "updateFrequency": { "title": "Every Monday", "rrule": "FREQ=WEEKLY;BYDAY=MO" },
    "podping": { "usesPodping": true }
  },
  "items": [
    {
      "id": "https://example.com/episodes/1",
      "url": "https://example.com/episodes/1",
      "title": "Episode 1: Why JSON",
      "content_html": "<p>We talk about JSON Feed and podcasts.</p>",
      "image": "https://example.com/ep1.jpg",
      "date_published": "2026-11-01T08:00:00Z",
      "attachments": [
        { "url": "https://example.com/ep1.mp3", "mime_type": "audio/mpeg", "size_in_bytes": 28800000, "duration_in_seconds": 2400 }
      ],
      "_podcast": {
        "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md",
        "season": { "number": 1, "name": "Beginnings" },
        "episode": { "number": 1 },
        "episodeType": "full",
        "transcripts": [{ "url": "https://example.com/ep1.vtt", "type": "text/vtt", "rel": "captions" }],
        "chapters": { "url": "https://example.com/ep1-chapters.json", "type": "application/json+chapters" },
        "persons": [
          { "name": "Terry Scott", "role": "host" },
          { "name": "June Whitfield", "role": "guest", "img": "https://example.com/june.jpg" }
        ],
        "alternateEnclosures": [
          { "type": "video/mp4", "height": 1080, "length": 512000000, "title": "Video", "sources": [{ "uri": "https://example.com/ep1-1080.mp4" }] }
        ]
      }
    }
  ]
}
```

## Open questions

- **iTunes fields in `_podcast`.** `explicit`, `showType` and `episodeType` come from the iTunes namespace, not the podcast namespace. They are here because a JSON podcast cannot do without them, and a second extension object for three fields seemed worse. Apple's categories are still missing; JSON Feed's free-form `tags` don't replace them. Should there be a `categories` field, or an `_itunes` object after all?
- **Plural names.** Rule 2 turns repeating tags into plural arrays (`persons`, `txts`), which reads well in JSON but means a converter needs the list of repeating tags. The alternative is to keep the tag names (`person: [ … ]`). Decide before anyone ships it.
- **Live items outside `items`.** Keeping them in `_podcast.liveItems` protects readers that don't know the extension, but readers that do now have two lists to merge. Is that the right trade-off?
- **Where `about` points.** It names this document at a temporary location. Should it instead name the namespace (`https://podcastindex.org/namespace/1.0`)? Either way readers ignore the value, so changing it breaks nothing.
