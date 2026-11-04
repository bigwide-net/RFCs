# RFC: Feed HTML - An HTML Subset for Feed Content

Oct 31, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-2-HTML-Subset-for-Feed-Content.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-2-HTML-Subset-for-Feed-Content.md)

## Summary

This proposal defines **Feed HTML**: the subset of HTML that a post body in RSS (`<description>`, `<content:encoded>`), Atom (`<content type="html">`) or JSON Feed (`content_html`) can use and expect to look the same in every reader that supports Feed HTML. It lists the allowed elements, attributes and CSS properties, the URL rules, and how to embed third-party content such as YouTube videos or TikToks.

It has two audiences:

- **Blog authors and feed generators**: write within the subset, and the post renders as intended in every reader that supports Feed HTML. Anything outside it may be dropped.
- **Reader authors**: support at least the subset, so that authors can rely on it. A reader may show more, but should not show less.

**Why now.** Every reader sanitizes feed HTML, because feed content is untrusted and has to sit inside the reader's own page. But every reader does it differently and none documents it, so authors find out what works by testing in each reader, or not at all. The result is posts that lose their videos, tables or images in some readers, and readers that each rediscover the same rules. The subset here is what Downread, a reader that supports Big Wide, implements today, minus everything that only exists to repair broken feeds.

**Out of scope.** Readers also need workarounds for feeds that are broken: lazy-loading images without a `src`, embeds that depend on stripped scripts, CMS-specific markup. Those are not part of this subset. A feed that follows this document needs none of them.

## Principles

1. **The reader owns the page.** Content is shown in a column whose width, font, colors and theme (light or dark) the reader decides. Content styles text; it does not lay out the page.
2. **No active content.** No scripts, no event handlers, no forms. The reader may load a small set of known vendor scripts for embeds (see [Embeds](#embeds)), but never a script from the feed.
3. **No requests the user didn't agree to.** Images and media load normally. Third-party embeds may be held back until the user agrees to them, and CSS never loads anything.
4. **Degrade to text.** Whatever a reader drops, the text inside should still make sense. An embed always carries a link to what it embeds.

## Elements

Readers with Feed HTML support keep these elements:

| Group | Elements |
| --- | --- |
| Text structure | `p`, `br`, `hr` |
| Headings | `h1`, `h2`, `h3`, `h4`, `h5`, `h6` |
| Lists | `ul`, `ol`, `li`, `dl`, `dt`, `dd` |
| Inline formatting | `b`, `i`, `u`, `s`, `strike`, `strong`, `em`, `mark`, `small`, `del`, `ins`, `sub`, `sup` |
| Links and images | `a`, `img`, `figure`, `figcaption`, `picture`, `source` |
| Quotes and code | `blockquote`, `q`, `cite`, `code`, `pre`, `kbd`, `samp`, `var` |
| Tables | `table`, `thead`, `tbody`, `tfoot`, `tr`, `th`, `td`, `caption`, `colgroup`, `col` |
| Containers | `div`, `span`, `article`, `section`, `aside`, `details`, `summary` |
| Time | `time` |
| Media | `audio`, `video`, `track` (and `source` inside them) |
| Embeds | `iframe`, from the hosts in [Iframe embeds](#iframe-embeds) only |

Every other element is removed **but its content is kept**. `<abbr title="…">HTML</abbr>` becomes `HTML`; `<header>`, `<footer>`, `<nav>` and `<main>` become their children.

A few elements are removed **together with their content**, because their text makes no sense on its own:

| Element | Why | Use instead |
| --- | --- | --- |
| `script`, `style` | Active content; styles would leak into the reader | Inline `style` attributes within the [CSS subset](#css) |
| `svg`, `math` | Not supported inline; their `<title>` and `<text>` would end up as stray words | `<img src="diagram.svg" alt="…">`, or an image of the formula with the TeX source in `alt` |
| `textarea`, `option` | Form controls | Plain text, `<pre>` |

Form elements (`form`, `input`, `button`, `select`), `object`, `embed`, `canvas`, `frame` and `template` are not part of the subset either.

Headings: authors should start at `h2`. Readers show the post title themselves, usually at a level the body should not compete with, and many readers scale `h1` down.

## Attributes

### Global

These attributes are allowed on every element:

| Attribute | Notes |
| --- | --- |
| `title`, `lang`, `dir` | As in HTML. Use `dir` (not Unicode direction marks) for right-to-left text. |
| `style` | Only the properties in [CSS](#css) survive. |
| `id` | Kept, but readers may prefix it (Downread writes `id="uc-notes"` for `id="notes"`) so it can't clash with the reader's own ids. See [Links](#links) for what that means for `#fragment` links. |
| `class` | Removed, except the classes that identify an [embed](#script-embeds). Readers don't have the site's stylesheet, so a class has no effect anyway. |

`data-*` attributes are removed, except the ones listed for embeds. All event handler attributes (`onclick`, `onload`, …) are removed.

### Per element

| Element | Attributes |
| --- | --- |
| `a` | `href`, `target`, `rel` |
| `img` | `src`, `alt`, `width`, `height`, `loading`, `srcset`, `sizes` |
| `source` | `src`, `srcset`, `sizes`, `type`, `media` |
| `audio` | `src`, `controls`, `preload` |
| `video` | `src`, `controls`, `width`, `height`, `poster`, `preload` |
| `track` | `src`, `kind`, `srclang`, `label`, `default` |
| `iframe` | `src`, `width`, `height`, `frameborder`, `allowfullscreen`, `allow` |
| `blockquote`, `q` | `cite` |
| `ol` | `start`, `type`, `reversed` |
| `li` | `value` |
| `td`, `th` | `colspan`, `rowspan`, `headers`; `th` also `scope` |
| `col`, `colgroup` | `span` |
| `time` | `datetime` |
| `details` | `open` |

Notes:

- **`img` `width`/`height`** must be positive integers (pixels), as HTML requires. Readers drop other values. Readers scale images down to the column width and keep the aspect ratio, so the attributes act as the image's aspect ratio and its maximum size. Authors should always set them: without them, the layout shifts while images load.
- **`srcset`/`sizes`** work as in HTML. Readers choose from the candidates for their column width, which is narrower than most blog layouts. Don't use the `auto` keyword in `sizes`.
- **`a` `target`/`rel`** are overwritten: readers open links in a new tab with `rel="noopener noreferrer"`.
- **`iframe` `width`/`height`** give the aspect ratio, not a size; see [Iframe embeds](#iframe-embeds). Readers replace any `style` on an iframe.
- **Legacy table attributes.** Readers should also accept `align`, `valign`, `bgcolor`, `border`, `cellpadding`, `cellspacing` and `width` on table elements, because email-first newsletters still use them. A table `width` is limited to a percentage (0–100%) or at most 300 pixels; anything else is dropped. Authors should use CSS instead.

## CSS

Only inline `style` attributes are supported. There is no `<style>` element, no `<link rel="stylesheet">`, and no classes to style.

### Allowed properties

| Group | Properties |
| --- | --- |
| Text | `color`, `background-color`, `background`, `font-size`, `font-weight`, `font-style`, `font-variant`, `text-align`, `text-decoration`, `text-transform`, `text-indent`, `line-height`, `letter-spacing`, `word-spacing`, `white-space`, `vertical-align` |
| Box | `border`, `border-top`, `border-right`, `border-bottom`, `border-left`, `border-color`, `border-width`, `border-style`, `border-radius`, `margin` and `margin-*`, `padding` and `padding-*`, `box-sizing` |
| Size | `width`, `height`, `aspect-ratio`, `max-width`, `max-height` |
| Lists | `list-style`, `list-style-type`, `list-style-position` |
| Tables | `border-collapse`, `border-spacing`, `table-layout` |
| Display | `display` |

Everything else is removed, notably `position`, `float`, `z-index`, `transform`, `font-family`, `overflow`, `flex`/`grid` properties, `min-width`, `opacity`, `filter`, `animation` and `transition`.

### Value restrictions

Readers drop a declaration whose value breaks one of these rules; the rest of the `style` attribute stays.

| Rule | Applies to | Reason |
| --- | --- | --- |
| No `url(…)` and no backslash escapes | all properties | CSS must never load anything: it would be a tracking beacon and bypass the URL rules. Backslashes would allow writing `url(` in escaped form. Use `<img>` for images. |
| No percentages | `padding`, `padding-*` | Percentage padding is the "responsive box" hack that reserves space for content a script inserts later. Without the script it's a big empty gap. |
| `0`–`100%`, or at most `300px` (or a bare number up to 300) | `max-width`, `max-height` | Lets content shrink inside the column, never push it wider. |
| `none`, `inline` or `block` only | `display` | `flex`, `grid`, `table` and friends would take over the reader's layout. |

### What authors should expect

- **Colors.** Readers have light and dark themes. Text colored for a white page may be unreadable on a dark one. If you set `color`, also set `background-color` on the same element, and never rely on color alone to carry meaning.
- **Fonts.** The reader picks the font. `font-family` is not supported; use `<code>`, `<kbd>` or `<pre>` for monospace.
- **Widths.** The column is narrower than your blog's and changes with the window. Use percentages or `max-width` rather than fixed pixel widths. Wide tables and `<pre>` blocks scroll horizontally.
- **`display: none`** is allowed so that email-style preview text can be hidden, but hidden content is still in the feed. Don't use it for anything else.

## URLs

### Schemes

| Where | Allowed schemes |
| --- | --- |
| `a href` | `http`, `https`, `mailto` |
| `img src`, `img srcset` | `http`, `https`, `data` (images only: `data:image/…`, under 50,000 characters) |
| `source`, `audio`, `video`, `track`, `iframe` | `http`, `https` |
| `blockquote cite`, `q cite` | `http`, `https` |

Protocol-relative URLs (`//example.com/x.jpg`) are allowed and resolve to `https`. A link (`<a>`) or image (`<img>`) with a disallowed URL is replaced by a plain `<span>` with the same content. URLs pointing to `localhost`, `*.local`, `*.localhost` or private networks (`10.*`, `172.16–31.*`, `192.168.*`) are treated as disallowed, because they would make the reader's user agent probe its own network.

### Relative URLs

Readers resolve relative URLs against **the item's link** (`<link>` in RSS and Atom, `url` in JSON Feed), and against the feed's URL if the item has none. They don't use `xml:base` or `<base>`. Authors should use absolute URLs anyway: some readers, mail clients and aggregators don't resolve relative URLs at all.

### Links

Readers open every link in a new tab or the system browser.

**Fragment links.** A link like `<a href="#fn1">` resolves against the item link, so it opens the footnote on the original web page, not inside the reader. Readers may prefix ids (see [Global](#global)), so an in-post jump is not guaranteed. If footnotes matter, put them at the end of the post, where readers will see them anyway.

## Media

`<img>`, `<picture>`, `<audio>` and `<video>` work as in HTML, with the URL rules above.

- Always give images `alt`, and `width` and `height` (see [Per element](#per-element)).
- Use `<figure>` with `<figcaption>` for captions. Readers style `title` on an image as a tooltip at most.
- Give `<audio>` and `<video>` the `controls` attribute. Readers don't autoplay; `autoplay`, `loop` and `muted` are not part of the subset.
- A podcast's episode file belongs in the enclosure (`<enclosure>`, `rel="enclosure"`, `attachments`), not in the body. A body `<audio>` element is for additional clips.
- Readers that work offline may replace `<audio>`, `<video>` and `<iframe>` with a placeholder when the network is unavailable.

## Size and text

- **Size.** Readers may truncate a body that is longer than 128 KiB of sanitized HTML. Large posts should link to the full article.
- **Characters.** Readers may remove control characters, the zero-width space (U+200B), direction marks (U+200E, U+200F, U+061C), bidi embeddings, overrides and isolates (U+202A–U+202E, U+2066–U+2069), word joiners (U+2060–U+2065) and the BOM (U+FEFF). Use the `dir` attribute for direction instead of bidi marks, and don't rely on zero-width spaces for line breaking. Readers must **keep** the zero-width non-joiner and joiner (U+200C, U+200D): emoji sequences like 👨‍👩‍👧 and scripts like Persian and Devanagari need them.
- **Titles are text.** An item title is plain text. Readers strip tags from it and decode entities, so `<em>` in a title shows as nothing at all, and `&lt;em&gt;` shows as `<em>`.

## Embeds

Third-party content comes in two forms: an **iframe** from a known player host, or a **blockquote** that a vendor script turns into an embed. Both work the same way from the author's side: write the markup the vendor's "embed" button gives you, and keep the fallback link inside it.

Readers treat every embed as a request to a third party. They may hold it back behind a placeholder ("External content from YouTube — Show and allow") until the user agrees, per service. That is why the fallback link matters: it is what users see until then, and what readers without embed support show forever.

### Iframe embeds

Readers keep an `<iframe>` only if its `src` is on one of these hosts (including subdomains):

| Service | Hosts | Example `src` |
| --- | --- | --- |
| YouTube | `youtube.com`, `youtube-nocookie.com` | `https://www.youtube-nocookie.com/embed/VIDEO_ID` |
| Vimeo | `vimeo.com` | `https://player.vimeo.com/video/123456789` |
| X (Twitter) | `twitter.com`, `x.com` | `https://platform.twitter.com/embed/Tweet.html?id=…` |
| JW Player | `cdn.jwplayer.com` | `https://cdn.jwplayer.com/players/MEDIA_ID-PLAYER_ID.html` |
| Simplecast | `player.simplecast.com` | `https://player.simplecast.com/EPISODE_ID` |
| Captivate | `player.captivate.fm` | `https://player.captivate.fm/episode/EPISODE_ID` |

An iframe from any other host is replaced by an empty `<span>`. Put a link next to it, or use a blockquote embed where the service has one.

**Sizing.** Readers make video iframes full column width and use `width`/`height` as the aspect ratio only:

- `width="560" height="315"` renders at 100% width with `aspect-ratio: 560 / 315`.
- Percentage or missing dimensions fall back to 16:9.
- Podcast players (Simplecast, Captivate) have a fixed height instead: full width, and the `height` attribute (or `height` in `style`) in pixels, kept between 50 and 800. Without one, 200.

So set `width` and `height` to the video's real proportions (`width="315" height="560"` for a vertical video), and don't wrap the iframe in a percentage-padding box.

**YouTube.** Readers may rewrite `youtube.com/embed/…` to `youtube-nocookie.com/embed/…`, YouTube's privacy-enhanced mode. Authors can use that host directly. Readers should send the iframe a referrer (Downread uses `referrerpolicy="origin"`); YouTube refuses to play without one.

**Recommended markup:**

```html
<figure>
  <iframe src="https://www.youtube-nocookie.com/embed/CpQnZgZCIo8"
          width="560" height="315" title="How feeds work"
          allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
          allowfullscreen></iframe>
  <figcaption><a href="https://www.youtube.com/watch?v=CpQnZgZCIo8">How feeds work</a> on YouTube</figcaption>
</figure>
```

### Script embeds

X, TikTok, Instagram, Threads, Reddit and Bluesky embed posts with a `<blockquote>` that holds the post's text and a link, plus a `<script>` that replaces the blockquote with the real embed. Readers remove the `<script>`, as they remove every script. Instead they recognize the blockquote by its class or attribute and load the vendor's script themselves, from a fixed URL, once the user allows the service.

So: keep the vendor's blockquote as it is. The `<script>` may stay too; readers ignore it. Without consent, offline, or in a reader without embed support, the blockquote shows as an ordinary quote with a link.

| Service | Blockquote marker | Kept attributes | Script the reader loads |
| --- | --- | --- | --- |
| X (Twitter) | `class="twitter-tweet"` | — | `https://platform.twitter.com/widgets.js` |
| TikTok | `class="tiktok-embed"` | `cite` | `https://www.tiktok.com/embed.js` |
| Instagram | `class="instagram-media"` | `data-instgrm-permalink`, `data-instgrm-version`, `data-instgrm-captioned` | `https://www.instagram.com/embed.js` |
| Threads | `class="text-post-media"` | `data-text-post-permalink`, `data-text-post-version` | `https://www.threads.com/embed.js` |
| Reddit | `class="reddit-embed-bq"` | `data-embed-height`, `data-embed-theme`, `data-embed-depth`, `data-embed-showmedia`, `data-embed-showtitle`, `data-embed-showedits`, `data-embed-created`, `data-embed-showusername`, `data-embed-locale` | `https://embed.reddit.com/widgets.js` |
| Bluesky | `data-bluesky-uri` attribute (class `bluesky-embed`) | `data-bluesky-uri`, `data-bluesky-cid`, `data-bluesky-embed-color-mode` | `https://embed.bsky.app/static/embed.js` |

Per service:

**X.** The script reads the post from the **last link** in the blockquote, so that link must point to the status:

```html
<blockquote class="twitter-tweet">
  <p lang="en" dir="ltr">Feeds are back.</p>
  — Example (@example) <a href="https://twitter.com/example/status/1850000000000000000">October 29, 2026</a>
</blockquote>
```

**TikTok.** The video id comes from `cite`, which must be the video's URL (`…/video/<id>`). TikTok's markup also sets `data-video-id`; readers derive it from `cite`, so `cite` is the one that has to be right:

```html
<blockquote class="tiktok-embed" cite="https://www.tiktok.com/@example/video/7300000000000000000">
  <a href="https://www.tiktok.com/@example/video/7300000000000000000">Watch on TikTok</a>
</blockquote>
```

**Instagram.** Keep `class="instagram-media"` and `data-instgrm-permalink`:

```html
<blockquote class="instagram-media" data-instgrm-permalink="https://www.instagram.com/p/ABC123/" data-instgrm-version="14">
  <a href="https://www.instagram.com/p/ABC123/">View this post on Instagram</a>
</blockquote>
```

**Threads.** Same pattern as Instagram, with Threads' class and attributes:

```html
<blockquote class="text-post-media" data-text-post-permalink="https://www.threads.com/@example/post/ABC123" data-text-post-version="0">
  <a href="https://www.threads.com/@example/post/ABC123">View on Threads</a>
</blockquote>
```

**Reddit.** Use the current `reddit-embed-bq` markup; the link inside points to the post or comment:

```html
<blockquote class="reddit-embed-bq" data-embed-height="316">
  <a href="https://www.reddit.com/r/rss/comments/abc123/feeds_are_back/">Feeds are back</a>
  by <a href="https://www.reddit.com/user/example/">u/example</a> in <a href="https://www.reddit.com/r/rss/">rss</a>
</blockquote>
```

**Bluesky.** Bluesky's script looks for the `data-bluesky-uri` attribute, not the class. The URI must use the author's **DID**, not their handle (`at://did:plc:…`, not `at://example.com`); the embed service rejects handles:

```html
<blockquote class="bluesky-embed" data-bluesky-uri="at://did:plc:abc123/app.bsky.feed.post/3k2abc" data-bluesky-cid="bafyrei…">
  <p lang="en">Feeds are back.</p>
  — Example (<a href="https://bsky.app/profile/did:plc:abc123">@example.com</a>)
  <a href="https://bsky.app/profile/did:plc:abc123/post/3k2abc">October 29, 2026</a>
</blockquote>
```

### Other services

Spotify, SoundCloud, Facebook, Twitch, Mastodon and other services are not in the subset yet. Their iframes and scripts are removed. Link to the content instead, with a descriptive label rather than a bare URL:

```html
<p><a href="https://open.spotify.com/episode/…">Listen on Spotify</a></p>
```

### Notes for reader authors

- **Gate before attaching.** Strip or hold back the iframe `src`, and the blockquote marker class or attribute, before the content enters the document. Otherwise the request goes out before the user had a say.
- **One script, many posts.** Load each vendor script once per page, and ask it to rescan when new posts appear (`twttr.widgets.load()`, `instgrm.Embeds.process()`, `bluesky.scan()`, TikTok's `tiktokEmbed.lib.render(nodes)`). Reddit's script has no rescan API; inject it again.
- **Instagram and Threads share a global** (`window.instgrm`). Allowing one service must not hydrate the other's blockquotes. Remove the marker class while a blockquote is gated.
- **Never load a script URL from the feed.** Detect embeds by their marker and load the script from your own list.

## Notes for publishers and readers

**Publishers**

- Generate the feed body from your content, not from your rendered page: without the site's header, scripts, share buttons and "related posts".
- Use absolute URLs, and give every image `alt`, `width` and `height`.
- Don't depend on classes, external CSS, JavaScript, `font-family` or layout properties. If a post only works with them, link to the web version.
- For every embed, keep a fallback link inside it.
- Test in a dark theme.

**Readers**

- Sanitize before rendering, with the allowlist in this document. Readers with a server can also sanitize when content enters their system, but should then use the same allowlist on both sides, so a post doesn't render differently depending on where it was cleaned.
- Feed content is untrusted: never let it run scripts, load CSS resources, or reach the user's local network.
- Supporting more than this subset is fine. Supporting less makes authors write for the lowest common denominator, which is what this RFC tries to end.

## Test feeds

Downread publishes static test feeds at [test-rss.downread.com](https://test-rss.downread.com/). Reader authors can subscribe to them to check their support for Feed HTML; feed authors can compare them with their own output.

| Feed | What it tests |
| --- | --- |
| [basic.rss.xml](https://test-rss.downread.com/basic.rss.xml), [basic.atom.xml](https://test-rss.downread.com/basic.atom.xml), [basic.json](https://test-rss.downread.com/basic.json) | Elements: formatting, headings, lists, tables, code, figures with captions, Unicode and emoji, a long read. |
| [styles.rss.xml](https://test-rss.downread.com/styles.rss.xml) | CSS: inline styles that must survive, and styles that readers must remove or neutralize. |
| [embeds.rss.xml](https://test-rss.downread.com/embeds.rss.xml), [embeds.json](https://test-rss.downread.com/embeds.json) | Embeds: YouTube, Vimeo, X, Instagram, TikTok, Threads, Reddit and Bluesky, one per post and all in one post, with the vendor's original markup. |
| [enclosures.rss.xml](https://test-rss.downread.com/enclosures.rss.xml), [enclosures.json](https://test-rss.downread.com/enclosures.json) | Images outside the body (enclosures, Media RSS, JSON Feed attachments), including one that duplicates a body image. |

The same host has feeds for the social extensions, such as [social.rss.xml](https://test-rss.downread.com/social.rss.xml); [the index](https://test-rss.downread.com/) lists them all. Some test posts deliberately contain markup outside the subset, to check that readers remove it cleanly.

## Full example

```html
<p>Last week I switched my blog to <strong>full-text feeds</strong>. Here is what changed.</p>

<h2>Numbers</h2>
<table>
  <thead><tr><th scope="col">Month</th><th scope="col">Subscribers</th></tr></thead>
  <tbody>
    <tr><td>September</td><td style="text-align: right">1,204</td></tr>
    <tr><td>October</td><td style="text-align: right">1,877</td></tr>
  </tbody>
</table>

<figure>
  <img src="https://example.com/img/chart.png" alt="Subscribers by month, rising from 1,204 to 1,877"
       width="1200" height="600"
       srcset="https://example.com/img/chart-600.png 600w, https://example.com/img/chart.png 1200w"
       sizes="(max-width: 700px) 100vw, 700px">
  <figcaption>Subscribers by month</figcaption>
</figure>

<blockquote cite="https://example.org/feeds-are-back">
  <p>Feeds never left. We just stopped looking at them.</p>
</blockquote>

<p style="background-color: #fff3c4; color: #3d3000; padding: 0.5em 1em; border-radius: 4px">
  <strong>Note:</strong> numbers exclude crawlers.
</p>

<figure>
  <iframe src="https://www.youtube-nocookie.com/embed/CpQnZgZCIo8" width="560" height="315"
          title="How feeds work" allowfullscreen></iframe>
  <figcaption><a href="https://www.youtube.com/watch?v=CpQnZgZCIo8">How feeds work</a></figcaption>
</figure>

<details>
  <summary>How I counted</summary>
  <p>Unique IPs per user agent and day, see <a href="https://example.com/posts/counting">this post</a>.</p>
</details>
```

## Open questions

- **A name and a version.** Should the subset be versioned ("Feed HTML 1") so that feeds can declare what they target, and readers what they support?
- **Inline SVG and MathML.** Both are common on technical blogs and can be sanitized. Is it worth adding them, or is `<img src="….svg">` enough?
- **More elements.** `abbr`, `dfn`, `ruby`/`rt`/`rp`, `wbr` and `bdi` are harmless and useful for text. They are missing only because nobody asked yet.
- **Footnotes.** In-post `#fragment` links don't work reliably once readers prefix ids. Should the subset define how readers rewrite them (prefix `href="#x"` the same way as `id="x"`)?
- **More embed services.** Which other embeds are used in feeds in the wild (Spotify, SoundCloud, Mastodon, Facebook, Twitch, CodePen, GitHub Gist…)? Any service that shows up in real feeds should be added. Each needs a host or marker, a consent name, and a fallback.
- **Dark mode.** Should readers adjust or drop author colors in dark themes, or should the subset drop `color` and `background-color` altogether?
