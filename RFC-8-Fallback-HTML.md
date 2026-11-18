# RFC: Fallback HTML - Backward Compatibility for Feed Extensions

Nov 18, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-8-Fallback-HTML.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-8-Fallback-HTML.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-8-Fallback-HTML.md)

## Summary

This proposal lets a post body contain HTML that only readers **without** support for a particular feed extension show. The publisher wraps that HTML in an element with the class `without-extension-<name>`, where `<name>` is the extension's name. A reader that supports the extension removes the element; every other reader shows it like any other HTML, because that is what readers already do with classes they don't know.

```html
<div class="without-extension-multiimage">
  <img src="https://example.com/images/img2.png" alt="The harbor at dawn" width="1200" height="800">
  <img src="https://example.com/images/img3.png" alt="Fishing boats leaving" width="1200" height="800">
</div>
```

A feed can thereby use a new extension, and readers can show it in the best possible way, without leaving behind the readers that don't support it yet. The fallback is the same feed, for everyone: there is no negotiation, no second feed and no `User-Agent` sniffing.

It has three audiences:

- **Publishers and feed generators**: when an item uses an extension that many readers don't support, add fallback HTML with the same content to the body, in the extension's class.
- **Reader authors**: for every extension you support, remove its fallback HTML before showing the post. Readers that support no extension need to do nothing.
- **Authors of extension specs**: give the extension a name, and require readers that support it to remove its fallback.

**Why now.** The Big Wide RFCs add things to feeds that most readers don't show: posts with several photos, card layouts for photo and video feeds, comments and discussions. Every existing feed extension faces the same choice. Either the publisher puts the content only into the extension, and readers that don't know it lose it. Or the publisher also puts it into the body, and readers that know the extension show it twice: once as a gallery, once more as images below the text. Most publishers therefore don't use new extensions at all, so readers have no reason to support them. Fallback HTML breaks that circle. A publisher can adopt an extension on day one, and a reader that supports it shows a clean post, while every other reader still shows the whole content.

The idea isn't new. HTML e-mails use conditional comments (`<!--[if mso]>`) to give Outlook different markup than every other mail client, and web pages use `<noscript>` for browsers without JavaScript. Fallback HTML is the same idea for feed extensions, built from nothing but a class.

## Principles

1. **Old readers need nothing.** Fallback HTML is ordinary HTML. A reader that has never heard of this RFC shows it, and that is exactly the right behavior, because it doesn't support the extension either.
2. **Readers decide by what they support.** The feed is the same for everyone. Each reader removes the fallbacks of the extensions it supports, and nothing else.
3. **A fallback only repeats.** Fallback HTML contains what the extension already carries, in a form that every reader can show. A reader that removes it loses nothing.
4. **Only removal.** The mechanism can hide content from supporting readers, never show content only to them. That belongs in the extension itself.

## The class

An element whose `class` attribute contains the token `without-extension-<name>` is **fallback HTML** for the extension `<name>`.

- **Any element.** The class may be on any element of [Feed HTML (RFC-2)](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md). Use a `div` around block content, a `span` inside a paragraph, or put the class directly on a `p`, `figure`, `img` or `a`.
- **The whole element.** The element is fallback HTML together with everything inside it.
- **Other classes.** The token may appear next to other classes: `class="gallery without-extension-multiimage"`. Readers split the attribute at whitespace, as HTML does, and look at each token.
- **Exact match.** The token is matched exactly and case-sensitively, as HTML class selectors are in standards mode. `Without-Extension-MultiImage` is not a fallback.
- **Several extensions.** An element with several fallback classes is removed if the reader supports **any** of them. `class="without-extension-a without-extension-b"` is shown only by readers that support neither. Nesting works the same way.

There is no `with-extension-<name>` class. A reader that doesn't know an extension doesn't know this RFC's meaning for it either, and would show the content anyway. Content that only supporting readers should show belongs in the extension's own fields.

## Extension names

An extension name consists of lower-case ASCII letters, digits and hyphens. It starts and ends with a letter or digit, and is at most 64 characters long: `multiimage`, `podcast-transcript`. Readers ignore classes whose name breaks these rules.

**Who names an extension.** The spec that defines the extension also defines its name and says what counts as supporting it, usually in a short section called "Fallback HTML". The name must stay the same for the lifetime of the extension. An incompatible new version of an extension gets a new name, such as `multiimage-2`, so that readers that only know the old version keep showing the fallback.

**Existing namespaces.** For an element of an existing RSS namespace whose spec doesn't define a name, the name is the namespace's customary prefix and the element's local name, lower-cased and joined with a hyphen: `<podcast:transcript>` is `podcast-transcript`, `<podcast:chapters>` is `podcast-chapters`, `<source:markdown>` is `source-markdown`. The same name applies to the element's JSON Feed counterpart (`_podcast.transcripts` from [RFC-3](https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md) is also `podcast-transcript`). A reader supports such an extension when it shows the element's content to the user: the transcript, the chapters.

## What "supports" means

A reader supports an extension when it shows, wherever it shows the post body, the content that the extension's fallback stands in for. Two rules make that predictable:

- **Support, not data, decides.** A reader removes the fallback for every extension it supports, whether or not the item actually uses that extension. Checking the item is the publisher's job: a fallback for an extension the item doesn't use is the publisher's mistake.
- **Per view.** A reader that shows a post in different ways decides for each of them. If its timeline shows only the first image of a post but its full view shows a gallery, the full view removes the `multiimage` fallback and the timeline keeps it, if it shows the body at all.

A reader that supports an extension only partially, such as a gallery that shows at most four images, should treat it as not supported, unless the extension's spec defines what partial support means.

## Example: several images

RSS allows only one enclosure per item, and most readers show only the first, from `<enclosure>`, `<media:content>` or JSON Feed's `attachments`. We call support for more than one image **`multiimage`**. (`multiimage` stands for a future extension and is used here only as an example. What exactly it supports is up to its spec.)

A photo blog posts three photos. It lists all three as media and adds the second and third to the body, as fallback:

```xml
<item>
  <title>Morning at the harbor</title>
  <link>https://example.com/posts/harbor</link>
  <guid>https://example.com/posts/harbor</guid>
  <media:content url="https://example.com/images/img1.png" type="image/png" medium="image" width="1200" height="800"/>
  <media:content url="https://example.com/images/img2.png" type="image/png" medium="image" width="1200" height="800"/>
  <media:content url="https://example.com/images/img3.png" type="image/png" medium="image" width="1200" height="800"/>
  <content:encoded><![CDATA[
    <p>Up at five for this light. Worth it.</p>
    <div class="without-extension-multiimage">
      <img src="https://example.com/images/img2.png" alt="Fishing boats leaving the harbor" width="1200" height="800">
      <img src="https://example.com/images/img3.png" alt="Gulls on the pier" width="1200" height="800">
    </div>
  ]]></content:encoded>
</item>
```

- A reader that shows only the first image shows it, then the text, then the other two images from the body. Nothing is lost.
- A reader that supports `multiimage` removes the `div` and shows the three images as a gallery, with the text below. Nothing is shown twice.

## Example: a transcript link

Most podcast apps don't show [`<podcast:transcript>`](https://github.com/Podcastindex-org/podcast-namespace/blob/main/docs/1.0.md#transcript). A publisher who wants every listener to find the transcript can link it in the show notes, for those apps only:

```html
<p>Episode 42: we talk about feeds that write.</p>
<p class="without-extension-podcast-transcript">
  <a href="https://example.com/episodes/42/transcript.html">Read the transcript</a>
</p>
```

An app that shows the transcript itself removes the link.

## Processing in the reader

A reader that supports at least one extension with a fallback processes the body like this:

1. Parse the HTML (`<content:encoded>`, `<description>`, Atom `<content type="html">`, JSON Feed `content_html`).
2. **Remove** every element that carries a `without-extension-<name>` class for an extension the reader supports, together with its content.
3. Sanitize the rest as usual, e.g. with the Feed HTML allowlist (RFC-2). RFC-2 keeps `without-extension-*` classes, so the fallbacks of extensions the reader doesn't support survive sanitization. Without a matching style rule, their class has no effect, and their content is shown as ordinary HTML.

Everything derived from the body uses the result of step 2: excerpts, previews, notifications, read-time estimates, full-text search and text-to-speech.

**Removing, not hiding.** Readers should remove fallback HTML from the document, not hide it. A web-based reader *can* hide it with CSS instead, with one rule per supported extension in its own stylesheet:

```css
.without-extension-multiimage { display: none !important; }
```

But browsers still load images inside hidden elements, so the user downloads every image twice. Hidden text also still ends up in excerpts and search, and is read by some screen readers. Use CSS hiding only where removing is impractical, and make sure the class survives sanitization (see below).

**Sanitizing on a server.** Some readers sanitize posts on a server and show them in several client apps, which may support different extensions. Such a server should **keep** `without-extension-*` classes through sanitization, as RFC-2 allows, and let each client remove the fallbacks of the extensions it supports. A server that can't do that must remove only the fallbacks of extensions that **every** one of its clients supports.

## Notes for publishers and readers

**Publishers**

- Add fallback HTML only for extensions the item actually uses, and only with content that the extension carries. The post must make sense both with and without the fallback.
- Never put the main content, the only copy of anything, or anything readers must see into fallback HTML.
- Keep fallback HTML within Feed HTML (RFC-2), and simple. It is for the readers with the least support.
- Put the fallback where readers without the extension expect the content: images after the text, a transcript link at the end of the show notes.
- Don't hide fallback HTML yourself, with `style="display: none"` or the `hidden` attribute. It would then be hidden in the readers that need it.
- If an RSS item has both `<description>` and `<content:encoded>` with HTML, add the fallback to both. Plain text (`content_text`, JSON Feed's `summary`, titles) has no fallback mechanism.
- Fallback HTML is not hidden from anyone who reads the feed itself. Don't use it to hide anything.

**Readers**

- For every extension you support, remove its fallback before showing the post or deriving anything from it.
- Don't remove the fallback of an extension you don't support, even if you know the name.
- If you support an extension in one view only, remove its fallback in that view only.

**Extension specs**

- Define a name and say what counts as supporting the extension.
- Require readers that support the extension to remove its fallback HTML. A reader that supports the extension but ignores its fallback shows the content twice, which is what this RFC exists to avoid.
- Describe the recommended fallback, so that publishers produce the same one.

**Converters and aggregators**

- Feed converters (RSS to JSON Feed and back), planets and other republishers keep the classes as they are.
- A converter that drops an extension's data, because the target format can't express it, must also turn its fallback HTML into ordinary HTML by removing the class. Otherwise a reader that supports the extension removes the fallback while the extension's data is gone, and the content is lost.

## Fallback HTML and comments

[Comment HTML (RFC-5)](https://github.com/bigwide-net/RFCs/blob/main/RFC-5-Comment-HTML.md) removes every `class` attribute from submitted comments and group posts, so a fallback in a comment would become ordinary HTML, and readers that support the extension would show its content twice. Clients must not use fallback HTML in submissions to an RFC-4 server until RFC-5 allows it.

## Why a class

| Alternative | Why not |
| --- | --- |
| A new element, such as `<fallback>` | Not valid HTML. Sanitizers treat unknown elements differently: some keep their content, others drop it, so old readers might lose the fallback. |
| A `data-*` attribute | Works in principle, since readers remove unknown attributes and keep the content. But a class is the common way to mark content for a reader (RFC-2's embeds work the same way), and a web reader can hide it with a plain selector. |
| Conditional comments, as in HTML e-mail | The wrong way around: readers remove comments, so content inside them would reach no one. |
| `<noscript>`, `hidden`, `display: none` | Hide the content from every reader that understands them, not only from the ones that support the extension. |
| Negotiation (a header, a parameter, a second feed) | Feeds are static files, served to everyone and cached. One feed for every reader is the whole point. |

## Full example

The same post as in [Example: several images](#example-several-images), as JSON Feed. The images are attachments, and the body carries the fallback:

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Example Photos",
  "home_page_url": "https://example.com/",
  "feed_url": "https://example.com/feed.json",
  "items": [
    {
      "id": "https://example.com/posts/harbor",
      "url": "https://example.com/posts/harbor",
      "title": "Morning at the harbor",
      "date_published": "2026-11-18T07:30:00Z",
      "image": "https://example.com/images/img1.png",
      "content_html": "<p>Up at five for this light. Worth it.</p><div class=\"without-extension-multiimage\"><img src=\"https://example.com/images/img2.png\" alt=\"Fishing boats leaving the harbor\" width=\"1200\" height=\"800\"><img src=\"https://example.com/images/img3.png\" alt=\"Gulls on the pier\" width=\"1200\" height=\"800\"></div>",
      "attachments": [
        { "url": "https://example.com/images/img1.png", "mime_type": "image/png" },
        { "url": "https://example.com/images/img2.png", "mime_type": "image/png" },
        { "url": "https://example.com/images/img3.png", "mime_type": "image/png" }
      ]
    }
  ]
}
```

What readers show:

| Reader | Shows |
| --- | --- |
| Knows neither this RFC nor `multiimage` | `img1.png` as the item image, the text, then `img2.png` and `img3.png` from the body. |
| Supports `multiimage` | A gallery of all three attachments and the text. The fallback `div` is removed. |
| Knows this RFC, but not `multiimage` | The same as the first reader: it removes only the fallbacks of extensions it supports. |

## Open questions

- **The prefix.** `without-extension-` is long but says what it does. Would a shorter prefix (`fallback-`, `no-ext-`) be better, or is a longer, unambiguous one less likely to clash with classes that sites already use?
- **A list of names.** Should the names live in one place, such as the README of this repository, so that publishers and readers can see which ones exist and which spec defines each?
- **Names for namespace elements.** Is the `<prefix>-<element>` rule for existing namespaces good enough, or should only names that a spec defines explicitly be valid? Not every element has an obvious "supported" behavior.
- **Partial support.** Should names be able to express levels of support (`multiimage-4` for "up to four images"), or is "treat partial support as no support" enough?
- **Comments.** Should Comment HTML (RFC-5) keep `without-extension-*` classes, so that comments and group posts can use extensions too?
