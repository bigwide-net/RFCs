# RFC: Comment HTML - An HTML Subset for Comments

Nov 4, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-5-Comment-HTML.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-5-Comment-HTML.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-5-Comment-HTML.md)

## Summary

This proposal defines **Comment HTML**: the HTML that a comment, a reply or a post in a discussion group may contain when one feed submits it to another. It is a small, strict subset of HTML. A base that every server accepts (paragraphs and line breaks) is extended by eight optional **features** — style, links, code, quotes, images, tables, lists and headlines — which each server switches on or off.

It has three audiences:

- **Clients** that write comments: produce Comment HTML, and only use the features the target allows.
- **Servers** that accept submissions, such as a [Feed Submissions (RFC-4)](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) server for a blog or a discussion group: check submissions against the subset and republish them in normalized form.
- **Readers**: Comment HTML is ordinary HTML. Any reader that shows HTML content shows Comment HTML.

[rssmention](https://github.com/bigwide-net/rssmention) implements it.

**Why now.** [rssmention](https://github.com/bigwide-net/rssmention) and [Big Wide](https://bigwide.net) open up a future of interoperable clients and forum software: a comment written in one app can be posted to a blog or a discussion group that runs on someone else's software. For that to work, client and server need rules that are more specific than "HTML is allowed", because no server can accept any HTML in a comment. Every forum and comment system has its own rules today, and none of them tells the client what they are.

When the client doesn't know what the server expects, the author finds out the hard way: the comment is rejected, or it shows up without the table, the link or the formatting they wrote. Defining the server's expectations, and announcing them to the client, avoids that frustration.

The needs of communities also differ. Some fight spam and allow no links at all. Some host complex discussions and want code, quotes and tables. Some want nothing but short, plain comments. Comment HTML gives them a common language to state their rules, so that any client can write comments for any server.

## Principles

1. **Reject, don't repair.** A submission that breaks a rule is rejected with a reason. The server never drops an element, closes a tag or guesses what was meant, so the author sees exactly what will be published. The one exception is attributes: unknown attributes are removed, because they never change what the reader sees.
2. **The client keeps the author within the rules.** Rejection is the server's last line of defense, not the way authors learn the rules. The client is responsible for preventing the user from breaking them: it must not offer features that the server doesn't support (no table button if tables are off), and it must enforce the limits (length, link text, URLs) before the comment is published.
3. **Small and closed.** Every element is listed here. There is no "everything else is kept as text": unknown elements are an error.
4. **Text first.** Comments are text. Markdown is the preferred input; HTML is the fallback, plain text the fallback after that.
5. **Servers decide.** Each feature is a switch. A group about programming turns on code; a group for photographers turns on images; a blog that fights spam turns off links.
6. **Plain HTML.** Every valid comment is ordinary HTML, so readers need nothing new.

## Elements

### Always allowed

| Element | Notes |
| --- | --- |
| `p` | Paragraphs. |
| `br` | Line breaks. |

Text without any element is also valid. A comment must contain some text or, if images are allowed, an image; an empty comment is rejected.

### Features

| Feature | Elements | Requirements |
| --- | --- | --- |
| `allowStyle` | `strong`, `em`, `s` | |
| `allowLinks` | `a` | `href` is required, see [URLs](#urls). Links must not be nested. |
| `allowCode` | `code`, `pre` | |
| `allowQuote` | `q`, `blockquote` | |
| `allowImages` | `img` | `src` is required, see [URLs](#urls). `alt` is recommended. |
| `allowTables` | `table`, `thead`, `tbody`, `tr`, `th`, `td` | Nesting is checked, see [Structure](#structure). |
| `allowLists` | `ul`, `ol`, `li` | Nesting is checked, see [Structure](#structure). |
| `allowHeadlines` | `h1`, `h2`, `h3`, `h4`, `h5`, `h6` | |

There is one more feature that adds no element:

| Feature | Meaning |
| --- | --- |
| `urlAutoLinking` | The server turns bare `http(s)` URLs in the text into links after checking the comment, except inside `a`, `code` and `pre`. It works even when `allowLinks` is off: the author can't choose the link text, but a URL in a comment is still clickable. |

Any element that is not in the base and not allowed by an enabled feature makes the submission invalid. This includes common elements such as `b`, `i`, `u`, `del`, `hr`, `div`, `span`, `figure`, `iframe` and `video`.

**Announcing the features.** A server tells clients which features are on. With RFC-4, they are the boolean rules of the same name in `<rm:policies>` / `_submissions.policies`, separately for posts and comments. Other systems may announce them differently, but should use the same names. A missing feature is off.

A client must not offer a feature that is off, and must not send its elements.

### Attributes

| Element | Attributes |
| --- | --- |
| `a` | `href` |
| `img` | `src`, `alt` |

All other attributes, on all elements, are removed: `class`, `style`, `id`, `title`, `target`, `rel`, `width`, `height`, `start`, `data-*`, event handlers. Unlike unknown elements, they don't invalidate the submission. Clients should not send them.

## Structure

Comment HTML is well-formed. A server rejects HTML that a browser would accept only through error recovery.

- **Every element is closed**, in the right order: `<p><em>a</em></p>`, not `<p><em>a</p></em>` and not `<p>a<p>b`. Closing tags that HTML lets you omit (`</p>`, `</li>`, `</td>`, `</tr>`) are required.
- **Void elements** are `br` and `img`. They may be written `<br>` or `<br/>`. No other element may be self-closing: `<p/>` is an error.
- **Markup is plain tags.** HTML comments (`<!-- … -->`) are allowed and removed. Doctypes, CDATA sections, processing instructions and a `<` that does not start a tag are errors; write `&lt;`.
- **Tables** contain only `thead`, `tbody` and `tr`; `thead` and `tbody` contain only `tr`; `tr` contains only `th` and `td`. `thead`, `tbody` and `tr` may not appear outside a table, `th` and `td` not outside a `tr`.
- **Lists** contain only `li`, and `li` appears only in `ul` and `ol`. Lists may be nested inside `li`.
- **No text in structure.** Text directly inside `table`, `thead`, `tbody`, `tr`, `ul` or `ol` is an error. Whitespace between elements is fine.
- **Links don't nest.** `a` inside `a` is an error.

Element and attribute names are case-insensitive; `<P>` is the same as `<p>`.

## Normalization

A server republishes the comment in normalized form, so that every reader and every downstream tool sees the same markup:

- Element and attribute names are lower-case.
- Only the attributes listed above remain. Attribute values are in double quotes, with `&`, `<`, `>` and `"` escaped.
- Void elements are written without a slash: `<br>`, `<img src="…" alt="…">`. Every other element has its closing tag.
- HTML comments are removed.
- Leading and trailing whitespace of the whole comment is removed. Text is otherwise kept as written, including its entities.
- With `urlAutoLinking`, bare URLs are wrapped in `<a href="…">…</a>`.

Normalization never changes the meaning of a valid comment. A client that wants to know exactly what will be published can apply the same steps itself. Servers that store the original Markdown (see below) republish it unchanged next to the normalized HTML.

## URLs

| Where | Rule |
| --- | --- |
| `a href`, `img src` | Absolute `http` or `https` URL, at most 512 characters. Relative URLs, `mailto:`, `data:` and every other scheme are rejected. |
| Link text | At most 120 characters of text inside an `a`. |

Short link texts keep comments readable and make it harder to disguise where a link goes. A long URL as link text is better left to `urlAutoLinking`, or shortened by the author.

## Markdown and plain text

A submission can carry its content in three forms. The server uses the first that is present:

1. **Markdown** (`<source:markdown>` in RSS, `_source.markdown` in JSON Feed, see [RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md)). It is rendered as [CommonMark](https://commonmark.org/) with tables and strikethrough, and the result must be valid Comment HTML. The HTML version of the item is then ignored.
2. **HTML** (`<content:encoded>` or `<description>` in RSS, `content_html` in JSON Feed).
3. **Plain text** (`content_text` in JSON Feed). Blank lines separate paragraphs (`<p>`), single newlines become `<br>`, and `&`, `<`, `>` and `"` are escaped.

Markdown rules:

- **Raw HTML is text.** `<b>` in Markdown is shown as `<b>`, not interpreted. Markdown can therefore only produce the elements its syntax maps to.
- **The features still apply.** `# Title` produces `h1` and needs `allowHeadlines`; `- item` needs `allowLists`; `![x](…)` needs `allowImages`; `` `x` `` needs `allowCode`; `> x` needs `allowQuote`; `*x*`, `**x**` and `~~x~~` need `allowStyle`; `[x](…)` needs `allowLinks`. A Markdown comment that uses a feature that is off is rejected.
- **Strikethrough** (`~~x~~`) produces `s`.
- **Not available:** thematic breaks (`---` produces `hr`, so the comment is rejected) and bare-URL autolinking by the Markdown renderer. Use `urlAutoLinking` for URLs.
- Attributes that renderers add (`class="language-js"` on code, `style="text-align: right"` on table cells, `start` on ordered lists) are removed like any other attribute.

**Length.** A server sets two limits (RFC-4's `minLength` and `maxLength`), measured differently:

- `maxLength` counts the characters of the form that was submitted: the Markdown if there is Markdown, the HTML otherwise, the text otherwise. Authors writing Markdown don't pay for the markup that rendering adds.
- `minLength` counts the **visible text** of the rendered comment: the text between the tags, with entities decoded and runs of whitespace collapsed to one space, trimmed. Block elements (`p`, `li`, `td`, headings, …) and `br` separate words like a space; inline elements (`strong`, `em`, `a`, `code`, `q`) don't. Markup never counts, so `<p><strong>Thanks!</strong></p>` has 7 characters. An image has no text: a comment that is only an image needs a `minLength` of 0.

rssmention uses a `minLength` of 10 by default, which keeps out "+1" and "Thanks!" comments that add nothing to a discussion.

## Titles

A title is plain text, never HTML. The server collapses runs of whitespace to one space, trims it, and rejects titles longer than 200 characters. Whether a title is required, allowed or forbidden is a server rule (RFC-4's `title`); comments usually have none.

## Notes for clients and servers

**Clients**

- Read the server's features before the author starts writing, and offer only those: no table button if tables are off.
- Prefer sending Markdown, and also write HTML for readers of the author feed. Generate both from the same text.
- Enforce the limits in the editor: length, link text, URLs, title. Show the problem before publishing; a rejected comment that is already in the author's feed is awkward to take back.
- Always close tags and quote attributes. Most HTML serializers do; HTML copied from a web page often doesn't.

**Servers**

- Check with a strict tokenizer, not a forgiving HTML parser: a parser repairs exactly the errors this subset rejects.
- Report the first problem with a clear reason that names the element or rule (`The element <hr> is not allowed`, `<li> must be inside <ul> or <ol>`). Authors fix one thing at a time.
- Apply the URL rules after auto-linking too: an auto-linked URL that is too long stays text.
- Store and republish the normalized HTML, and the Markdown if there was any.

## Examples

A valid comment on a server with `allowStyle`, `allowLinks` and `allowCode`:

```html
<p>Works for me with <code>curl -I</code>.</p>
<p>See <a href="https://www.rfc-editor.org/rfc/rfc5005">RFC 5005</a>, <em>section 2</em>.</p>
```

The same in Markdown:

```markdown
Works for me with `curl -I`.

See [RFC 5005](https://www.rfc-editor.org/rfc/rfc5005), *section 2*.
```

Submissions that are rejected, with the reason:

| Submission | Reason |
| --- | --- |
| `<p>Hello<p>World` | `p` not closed. |
| `<p><b>Important</b></p>` | `b` is not in the subset; use `strong`. |
| `<p>Read <a href="/about">this</a></p>` | Relative URL. |
| `<ul>Things<li>one</li></ul>` | Text directly inside `ul`. |
| `<p>A<br/>B</p><div/>` | `div` is not in the subset, and only void elements may be self-closing. |
| `<h2>Update</h2>` without `allowHeadlines` | Headline feature is off. |

What the server republishes for a valid comment that is not yet normalized, with `urlAutoLinking` on:

```html
<P CLASS="note">Docs: https://example.com/docs <!-- todo --><BR/>Thanks!</P>
```

```html
<p>Docs: <a href="https://example.com/docs">https://example.com/docs</a> <br>Thanks!</p>
```

## Open questions

- **More elements.** `hr`, `sup`/`sub`, `kbd`, `ol start` and `figure`/`figcaption` are harmless. Which are worth adding, and behind which feature?
- **Image size.** Without `width` and `height`, readers can't reserve space for images. Should `img` keep them, or should servers fetch the image and add them?
- **`mailto:` links.** In comments they invite address harvesting, but have legitimate uses.
- **Feature names.** The names come from rssmention's configuration. A version or a single list of enabled features (`"features": ["style", "links"]`) might be simpler for other implementations.
- **Repairing instead of rejecting.** Some clients can't control their HTML (a blog engine that writes `<p>` without closing it). Should servers be allowed to accept HTML that a standard HTML parser repairs unambiguously, as long as they republish the normalized result?
