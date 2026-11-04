# RFC: Comment HTML - An HTML Subset for Comments

Nov 4, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-5-Comment-HTML.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-5-Comment-HTML.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-5-Comment-HTML.md)

## Summary

This proposal defines **Comment HTML**: the HTML that a comment, a reply or a post in a discussion group may contain when one feed submits it to another. It is a small, strict subset of [Feed HTML (RFC-2)](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md). A base that every server accepts (paragraphs and line breaks) is extended by eight optional **features** — style, links, code, quotes, images, tables, lists and headlines — which each group switches on or off.

It has three audiences:

- **Clients** that write comments: produce Comment HTML, and only use the features the target allows.
- **Servers** that accept submissions, such as a [Feed Submissions (RFC-4)](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) group server: check submissions against the subset and republish them in normalized form.
- **Readers**: anything that renders Feed HTML renders Comment HTML.

[rssmention](https://github.com/bigwide-net/rssmention) implements it.

**Why now.** Feed HTML describes what a blog post may contain, and a reader *sanitizes* towards it: what doesn't fit is dropped, and the post is shown anyway. That is right for a post that the author has already published on their own site. It is wrong for a comment that a server republishes under its own name, next to other people's comments. Moderators need to know what they are approving, other readers need comments that look alike, and the author needs to know that what they wrote is what gets shown. Comments also have different needs: nobody embeds a video in a reply, but a code snippet or a quote is common. So a comment server *rejects* what doesn't fit, and tells the author why, instead of quietly changing it.

## Principles

1. **Reject, don't repair.** A submission that breaks a rule is rejected with a reason. The server never drops an element, closes a tag or guesses what was meant, so the author sees exactly what will be published. The one exception is attributes: unknown attributes are removed, because they never change what the reader sees.
2. **Small and closed.** Every element is listed here. There is no "everything else is kept as text": unknown elements are an error.
3. **Text first.** Comments are text. Markdown is the preferred input; HTML is the fallback, plain text the fallback after that.
4. **Groups decide.** Each feature is a switch. A group about programming turns on code; a group for photographers turns on images.
5. **A subset of Feed HTML.** Every valid comment is valid Feed HTML, so readers need nothing new.

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
| `allowStyle` | `strong`, `em` | |
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

Any element that is not in the base and not allowed by an enabled feature makes the submission invalid. This includes the elements that Feed HTML allows but Comment HTML doesn't, such as `b`, `i`, `s`, `del`, `hr`, `div`, `span`, `figure`, `iframe` and `video`.

**Announcing the features.** A server tells clients which features are on. With RFC-4, they are the boolean rules of the same name in `<rm:policies>` / `_submissions.policies`, separately for posts and comments. Other systems may announce them differently, but should use the same names. A missing feature is off.

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

1. **Markdown** (`<source:markdown>` in RSS, `_source.markdown` in JSON Feed, see [RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md)). It is rendered as [CommonMark](https://commonmark.org/) with tables, and the result must be valid Comment HTML. The HTML version of the item is then ignored.
2. **HTML** (`<content:encoded>` or `<description>` in RSS, `content_html` in JSON Feed).
3. **Plain text** (`content_text` in JSON Feed). Blank lines separate paragraphs (`<p>`), single newlines become `<br>`, and `&`, `<`, `>` and `"` are escaped.

Markdown rules:

- **Raw HTML is text.** `<b>` in Markdown is shown as `<b>`, not interpreted. Markdown can therefore only produce the elements its syntax maps to.
- **The features still apply.** `# Title` produces `h1` and needs `allowHeadlines`; `- item` needs `allowLists`; `![x](…)` needs `allowImages`; `` `x` `` needs `allowCode`; `> x` needs `allowQuote`; `*x*` and `**x**` need `allowStyle`; `[x](…)` needs `allowLinks`. A Markdown comment that uses a feature the group has off is rejected.
- **Not available:** thematic breaks (`---` produces `hr`, so the comment is rejected), strikethrough (`~~x~~` stays as literal text), and bare-URL autolinking by the Markdown renderer. Use `urlAutoLinking` for URLs.
- Attributes that renderers add (`class="language-js"` on code, `style="text-align: right"` on table cells, `start` on ordered lists) are removed like any other attribute.

**Length.** A group sets two limits (RFC-4's `minLength` and `maxLength`), measured differently:

- `maxLength` counts the characters of the form that was submitted: the Markdown if there is Markdown, the HTML otherwise, the text otherwise. Authors writing Markdown don't pay for the markup that rendering adds.
- `minLength` counts the **visible text** of the rendered comment: the text between the tags, with entities decoded and runs of whitespace collapsed to one space, trimmed. Block elements (`p`, `li`, `td`, headings, …) and `br` separate words like a space; inline elements (`strong`, `em`, `a`, `code`, `q`) don't. Markup never counts, so `<p><strong>Thanks!</strong></p>` has 7 characters. An image has no text: a comment that is only an image needs a `minLength` of 0.

rssmention uses a `minLength` of 10 by default, which keeps out "+1" and "Thanks!" comments that add nothing to a discussion.

## Titles

A title is plain text, never HTML. The server collapses runs of whitespace to one space, trims it, and rejects titles longer than 200 characters. Whether a title is required, allowed or forbidden is a group rule (RFC-4's `title`); comments usually have none.

## Comment HTML and Feed HTML

| | Feed HTML (RFC-2) | Comment HTML |
| --- | --- | --- |
| Applies to | Posts in a feed the author publishes | Content that a server republishes for someone else |
| Unknown elements | Removed, content kept | Submission rejected |
| Broken markup | Repaired by the HTML parser | Submission rejected |
| Elements | About 70 | 25, most of them behind a feature |
| Attributes | Many, plus a CSS subset | `a@href`, `img@src`, `img@alt` only |
| URLs | `http(s)`, `mailto`, small `data:` images, relative | Absolute `http(s)` only |
| Embeds, media, CSS | Yes | No |

Every valid comment is valid Feed HTML. A server that republishes comments in a feed therefore needs no extra rules in the reader.

## Notes for clients and servers

**Clients**

- Read the group's features before the author starts writing, and offer only those: no table button in a group without tables.
- Prefer sending Markdown, and also write HTML for readers of the author feed. Generate both from the same text.
- Validate locally with the same rules, and show the author the problem before publishing. A rejected comment that is already in the author's feed is awkward to take back.
- Always close tags and quote attributes. Most HTML serializers do; HTML copied from a web page often doesn't.

**Servers**

- Check with a strict tokenizer, not a forgiving HTML parser: a parser repairs exactly the errors this subset rejects.
- Report the first problem with a clear reason that names the element or rule (`The element <hr> is not allowed`, `<li> must be inside <ul> or <ol>`). Authors fix one thing at a time.
- Apply the URL rules after auto-linking too: an auto-linked URL that is too long stays text.
- Store and republish the normalized HTML, and the Markdown if there was any.

## Examples

A valid comment in a group with `allowStyle`, `allowLinks` and `allowCode`:

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
| `<h2>Update</h2>` in a group without `allowHeadlines` | Headline feature is off. |

What the server republishes for a valid comment that is not yet normalized, with `urlAutoLinking` on:

```html
<P CLASS="note">Docs: https://example.com/docs <!-- todo --><BR/>Thanks!</P>
```

```html
<p>Docs: <a href="https://example.com/docs">https://example.com/docs</a> <br>Thanks!</p>
```

## Open questions

- **Strikethrough.** `s` or `del` is common in comments ("~~wrong~~ right"). Should it be part of `allowStyle`?
- **More of Feed HTML.** `hr`, `sup`/`sub`, `kbd`, `ol start` and `figure`/`figcaption` are harmless. Which are worth adding, and behind which feature?
- **Image size.** Without `width` and `height`, readers can't reserve space for images. Should `img` keep them, or should servers fetch the image and add them?
- **`mailto:` links.** Feed HTML allows them. In comments they invite address harvesting, but have legitimate uses.
- **Feature names.** The names come from rssmention's configuration. A version or a single list of enabled features (`"features": ["style", "links"]`) might be simpler for other implementations.
- **Repairing instead of rejecting.** Some clients can't control their HTML (a blog engine that writes `<p>` without closing it). Should servers be allowed to accept HTML that a standard HTML parser repairs unambiguously, as long as they republish the normalized result?
