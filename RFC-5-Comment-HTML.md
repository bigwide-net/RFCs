# RFC: Comment HTML and Markdown - Formatting Rules for Comments

Nov 4, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-5-Comment-HTML.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-5-Comment-HTML.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-5-Comment-HTML.md)

## Summary

This proposal defines the formatting that a comment, a reply or a post in a discussion group may use when one feed submits it to another. It comes in two forms that share the same rules:

- **Comment Markdown**: [CommonMark](https://commonmark.org/) with tables and strikethrough, and without raw HTML. It is the preferred form.
- **Comment HTML**: a small, strict subset of HTML. It is what Comment Markdown renders to, and the form that readers show.

A base that every server accepts (paragraphs and line breaks) is extended by eight optional **features** — style, links, code, quotes, images, tables, lists and headlines — which each server switches on or off. A feature covers both forms: when a server turns off tables, it accepts neither `<table>` nor a Markdown pipe table.

It has three audiences:

- **Clients** that write comments: produce Comment Markdown or Comment HTML, and only use the features the target allows.
- **Servers** that accept submissions, such as a [Feed Submissions (RFC-4)](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) server for a blog or a discussion group: check submissions against the rules and republish them in normalized form.
- **Readers**: Comment HTML is ordinary HTML, and Comment Markdown is ordinary CommonMark. Any reader that shows HTML or Markdown content shows them.

[rssmention](https://github.com/bigwide-net/rssmention) implements it.

**Why now.** [rssmention](https://github.com/bigwide-net/rssmention) opens up a future of interoperable clients and forum software: a comment written in one app can be posted to a blog or a discussion group that runs on someone else's software. For that to work, client and server need rules that are more specific than "HTML is allowed" or "Markdown is allowed", because no server can accept any HTML in a comment, and most Markdown can produce headlines, images and tables that a server may not want. Every forum and comment system has its own rules today, and none of them tells the client what they are.

When the client doesn't know what the server expects, the author finds out the hard way: the comment is rejected, or it shows up without the table, the link or the formatting they wrote. Defining the server's expectations, and announcing them to the client, avoids that frustration.

The needs of communities also differ. Some fight spam and allow no links at all. Some host complex discussions and want code, quotes and tables. Some want nothing but short, plain comments. Comment HTML and Markdown give them a common language to state their rules, so that any client can write comments for any server.

## Principles

1. **Reject, don't repair.** A submission that breaks a rule is rejected with a reason. The server never drops an element, closes a tag or guesses what was meant, so the author sees exactly what will be published. The one exception is attributes: unknown attributes are removed, because they never change what the reader sees.
2. **The client keeps the author within the rules.** Rejection is the server's last line of defense, not the way authors learn the rules. The client is responsible for preventing the user from breaking them: it must not offer features that the server doesn't support (no table button if tables are off), and it must enforce the limits (length, link text, URLs) before the comment is published.
3. **Small and closed.** Every element and every piece of Markdown syntax is listed here. There is no "everything else is kept as text": unknown elements are an error.
4. **Text first.** Comments are text. Markdown is the preferred input; HTML is the fallback, plain text the fallback after that.
5. **One set of rules.** Markdown is checked by what it renders to. Every rule for Comment HTML applies to Comment Markdown as well, so the two forms never disagree about what is allowed.
6. **Servers decide.** Each feature is a switch. A group about programming turns on code; a group for photographers turns on images; a blog that fights spam turns off links.
7. **Plain HTML and Markdown.** Every valid comment is ordinary HTML and, if written in Markdown, ordinary CommonMark, so readers need nothing new.

## Forms of content

A submission can carry its content in three forms. The server uses the first that is present:

1. **Markdown** (`<source:markdown>` in RSS, `_source.markdown` in JSON Feed, see [RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md)). The server renders it to HTML, see [Markdown](#markdown), and the result must be valid Comment HTML. The HTML version of the item is then ignored.
2. **HTML** (`<content:encoded>` or `<description>` in RSS, `content_html` in JSON Feed).
3. **Plain text** (`content_text` in JSON Feed). Blank lines separate paragraphs (`<p>`), single newlines become `<br>`, and `&`, `<`, `>` and `"` are escaped.

## Elements and syntax

### Always allowed

| HTML | Markdown | Notes |
| --- | --- | --- |
| `p` | Text, with a blank line between paragraphs | Paragraphs. |
| `br` | Two spaces or a `\` at the end of a line | Line breaks. |

Text without any element is also valid. A comment must contain some text or, if images are allowed, an image; an empty comment is rejected.

### Features

| Feature | HTML | Markdown | Requirements |
| --- | --- | --- | --- |
| `allowStyle` | `strong`, `em`, `s` | `**strong**` or `__strong__`, `*em*` or `_em_`, `~~s~~` | |
| `allowLinks` | `a` | `[text](https://…)`, `[text][ref]` with a `[ref]: https://…` line, `<https://…>` | `href` is required, see [URLs](#urls). Links must not be nested. |
| `allowCode` | `code`, `pre` | `` `code` ``; code blocks fenced with ` ``` ` or `~~~`, or indented by four spaces | |
| `allowQuote` | `q`, `blockquote` | `> quote` for `blockquote`; `q` has no Markdown syntax | |
| `allowImages` | `img` | `![alt](https://…)`, `![alt][ref]` | `src` is required, see [URLs](#urls). `alt` is recommended. |
| `allowTables` | `table`, `thead`, `tbody`, `tr`, `th`, `td` | Pipe tables: `\| a \| b \|`, then a delimiter row `\| --- \| --- \|`, then the rows | Nesting is checked, see [Structure](#structure). |
| `allowLists` | `ul`, `ol`, `li` | `- item`, `* item` or `+ item` for `ul`; `1. item` or `1) item` for `ol` | Nesting is checked, see [Structure](#structure). |
| `allowHeadlines` | `h1` … `h6` | `# h1` … `###### h6`; a line underlined with `===` (`h1`) or `---` (`h2`) | |

A Markdown comment that uses the syntax of a feature that is off is rejected, just like the HTML element would be. Note that some of this syntax is easy to write by accident: a line starting with `- ` is a list, a line starting with `# ` is a headline, and text in `*` is emphasized. A client that offers a Markdown editor should point that out while the author writes (see [Notes for clients](#notes-for-clients-and-servers)), and the author can escape the character with `\`: `\# 1`, `\- not a list`, `\*not emphasized\*`.

There is one more feature that adds no element:

| Feature | Meaning |
| --- | --- |
| `urlAutoLinking` | The server turns bare `http(s)` URLs in the text into links after checking the comment, except inside `a`, `code` and `pre`. This applies to HTML and Markdown alike: the Markdown renderer itself doesn't link bare URLs. It works even when `allowLinks` is off: the author can't choose the link text, but a URL in a comment is still clickable. |

### Not allowed

Any element that is not in the base and not allowed by an enabled feature makes the submission invalid. This includes common elements such as `b`, `i`, `u`, `del`, `hr`, `div`, `span`, `figure`, `iframe` and `video`.

In Markdown, everything that is not listed above is unavailable:

- **Thematic breaks** (`---`, `***` or `___` on a line of their own) produce `hr`, so the comment is rejected. Note that `---` directly under a line of text is a headline instead.
- **Raw HTML is text.** `<b>x</b>` in Markdown is shown as `<b>x</b>`, not interpreted, and so are HTML comments. Markdown can therefore only produce the elements its syntax maps to.
- **Extensions** beyond tables and strikethrough are not part of Comment Markdown: no task lists, footnotes, definition lists, math, emoji shortcodes, bare-URL autolinking (use `urlAutoLinking`) or typographic replacements. Their syntax stays plain text.

**Announcing the features.** A server tells clients which features are on. With RFC-4, they are the boolean rules of the same name in `<rm:policies>` / `_submissions.policies`, separately for posts and comments. Other systems may announce them differently, but should use the same names. A missing feature is off.

A client must not offer a feature that is off, and must not send its elements or its Markdown syntax.

### Attributes

| Element | Attributes | Markdown |
| --- | --- | --- |
| `a` | `href` | The URL in `[text](url)` |
| `img` | `src`, `alt` | The URL and the alt text in `![alt](url)` |

All other attributes, on all elements, are removed: `class`, `style`, `id`, `title`, `target`, `rel`, `width`, `height`, `start`, `data-*`, event handlers. Unlike unknown elements, they don't invalidate the submission. Clients should not send them.

This includes the attributes that Markdown renderers add: `class="language-js"` on fenced code, `style="text-align: right"` on cells of aligned table columns, `start` on ordered lists that don't start at 1, and the link title in `[text](url "title")`. Their Markdown syntax is allowed, but it has no effect on the HTML: an ordered list that starts with `3.` is published as a list starting at 1, and table columns are not aligned.

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

A Markdown renderer always produces well-formed HTML, so these rules only ever reject HTML submissions. Markdown needs no closing tags: a list ends at the next blank line followed by unindented text, a quote at the first line without `>`.

## Markdown

Comment Markdown is CommonMark, plus two extensions as defined by [GitHub Flavored Markdown](https://github.github.com/gfm/): **tables** and **strikethrough**.

- The server renders it with raw HTML disabled and without any other extension, then checks the result as Comment HTML. The features, URL rules and length rules apply to the rendered result.
- Entities and backslash escapes work as in CommonMark: `&copy;` is ©, `\*` is a literal `*`.
- Strikethrough is written with two tildes (`~~x~~`) and produces `s`.
- A reader that knows Markdown may render the republished Markdown itself (RFC-1 recommends it). Because of the rules above, it gets the same elements as the server's HTML, minus the removed attributes.

## Normalization

A server republishes the comment in normalized form, so that every reader and every downstream tool sees the same markup:

- Element and attribute names are lower-case.
- Only the attributes listed above remain. Attribute values are in double quotes, with `&`, `<`, `>` and `"` escaped.
- Void elements are written without a slash: `<br>`, `<img src="…" alt="…">`. Every other element has its closing tag.
- HTML comments are removed.
- Leading and trailing whitespace of the whole comment is removed. Text is otherwise kept as written, including its entities.
- With `urlAutoLinking`, bare URLs are wrapped in `<a href="…">…</a>`.

Normalization never changes the meaning of a valid comment. A client that wants to know exactly what will be published can apply the same steps itself. If the comment was written in Markdown, the server republishes the Markdown unchanged next to the normalized HTML.

## URLs

| Where | HTML | Markdown | Rule |
| --- | --- | --- | --- |
| Link and image URLs | `a href`, `img src` | `(url)` in links and images, `[ref]: url` definitions, `<url>` | Absolute `http` or `https` URL, at most 512 characters. Relative URLs, `mailto:`, `data:` and every other scheme are rejected. |
| Link text | The text inside an `a` | The text in `[…]` | At most 120 characters. |

Short link texts keep comments readable and make it harder to disguise where a link goes. A long URL as link text is better left to `urlAutoLinking`, or shortened by the author. Note that `<https://…>` in Markdown uses the URL as link text, so the limit applies to it as well.

## Length

A server sets two limits (RFC-4's `minLength` and `maxLength`), measured differently:

- `maxLength` counts the characters of the form that was submitted: the Markdown if there is Markdown, the HTML otherwise, the text otherwise. Authors writing Markdown don't pay for the markup that rendering adds.
- `minLength` counts the **visible text** of the rendered comment: the text between the tags, with entities decoded and runs of whitespace collapsed to one space, trimmed. Block elements (`p`, `li`, `td`, headings, …) and `br` separate words like a space; inline elements (`strong`, `em`, `s`, `a`, `code`, `q`) don't. Markup never counts, so `<p><strong>Thanks!</strong></p>` and `**Thanks!**` both have 7 characters. An image has no text: a comment that is only an image needs a `minLength` of 0.

rssmention uses a `minLength` of 10 by default, which keeps out "+1" and "Thanks!" comments that add nothing to a discussion.

## Titles

A title is plain text, never HTML or Markdown. The server collapses runs of whitespace to one space, trims it, and rejects titles longer than 200 characters. Whether a title is required, allowed or forbidden is a server rule (RFC-4's `title`); comments usually have none.

## Notes for clients and servers

**Clients**

- Read the server's features before the author starts writing, and offer only those: no table button if tables are off.
- Prefer sending Markdown, and also write HTML for readers of the author feed. Generate the HTML from the Markdown with the same rules, so the two never differ.
- In a Markdown editor, check the text against the features as the author types, and highlight syntax that a disabled feature would reject: a line starting with `#` or `- `, a `---` line, a `[link](…)` when links are off. Offer to escape it.
- Enforce the limits in the editor: length, link text, URLs, title. Show the problem before publishing; a rejected comment that is already in the author's feed is awkward to take back.
- When sending HTML, always close tags and quote attributes. Most HTML serializers do; HTML copied from a web page often doesn't.

**Servers**

- Render Markdown with a CommonMark renderer, raw HTML disabled and only the table and strikethrough extensions enabled. Then check its output like submitted HTML.
- Check HTML with a strict tokenizer, not a forgiving HTML parser: a parser repairs exactly the errors this subset rejects.
- Report the first problem with a clear reason that names the element or rule (`The element <hr> is not allowed`, `<li> must be inside <ul> or <ol>`). Authors fix one thing at a time. For Markdown, naming the element is usually enough: `<h1> is not allowed` tells the author that a line starts with `#`.
- Apply the URL rules after auto-linking too: an auto-linked URL that is too long stays text.
- Store and republish the normalized HTML, and the Markdown if there was any.

## Examples

A valid comment on a server with `allowStyle`, `allowLinks` and `allowCode`, in Markdown:

```markdown
Works for me with `curl -I`.

See [RFC 5005](https://www.rfc-editor.org/rfc/rfc5005), *section 2*.
```

The same in HTML, which is also what the server publishes for the Markdown:

```html
<p>Works for me with <code>curl -I</code>.</p>
<p>See <a href="https://www.rfc-editor.org/rfc/rfc5005">RFC 5005</a>, <em>section 2</em>.</p>
```

Submissions that are rejected, with the reason:

| HTML | Markdown | Reason |
| --- | --- | --- |
| `<p>Hello<p>World` | — | `p` not closed. Markdown can't produce this error. |
| `<p><b>Important</b></p>` | — | `b` is not in the subset; use `strong`. In Markdown, `<b>` is just text. |
| `<p>Read <a href="/about">this</a></p>` | `Read [this](/about)` | Relative URL. |
| `<ul>Things<li>one</li></ul>` | — | Text directly inside `ul`. |
| `<p>A</p><hr><p>B</p>` | `A` / `---` on its own line after a blank line / `B` | `hr` is not in the subset. |
| `<h2>Update</h2>` without `allowHeadlines` | `## Update` | Headline feature is off. |
| `<ol><li>Step</li></ol>` without `allowLists` | `1. Step` | List feature is off. |

What the server republishes for a valid HTML comment that is not yet normalized, with `urlAutoLinking` on:

```html
<P CLASS="note">Docs: https://example.com/docs <!-- todo --><BR/>Thanks!</P>
```

```html
<p>Docs: <a href="https://example.com/docs">https://example.com/docs</a> <br>Thanks!</p>
```

## Open questions

- **More elements.** `hr`, `sup`/`sub`, `kbd`, `ol start` and `figure`/`figcaption` are harmless. Which are worth adding, and behind which feature? `hr` and `ol start` would also make the Markdown syntax that produces them work as expected.
- **More Markdown.** Task lists and footnotes are common in GitHub Flavored Markdown. Are they worth the extra elements and attributes they need?
- **Image size.** Without `width` and `height`, readers can't reserve space for images. Should `img` keep them, or should servers fetch the image and add them?
- **`mailto:` links.** In comments they invite address harvesting, but have legitimate uses.
- **Feature names.** The names come from rssmention's configuration. A version or a single list of enabled features (`"features": ["style", "links"]`) might be simpler for other implementations.
- **Repairing instead of rejecting.** Some clients can't control their HTML (a blog engine that writes `<p>` without closing it). Should servers be allowed to accept HTML that a standard HTML parser repairs unambiguously, as long as they republish the normalized result?
