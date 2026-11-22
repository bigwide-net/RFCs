# RFC: Feed Actions - Buttons, Choices and Inputs for Feed Items

Nov 18, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-9-Feed-Actions.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-9-Feed-Actions.md)

## Summary

This proposal lets a JSON feed offer **actions**: things the user can do with an item, or with the feed itself, right in the reader. The feed describes a short list of **controls**: buttons, selects, text inputs and short texts. The reader draws them with its own widgets. When the user clicks a button, the reader sends a `POST` request to a URL that the feed names, with the values the user entered, and the response tells the reader what to show next.

Everything lives in one extension object, `_actions`:

- On the **feed**, `templates` defines controls once, by name, so that items can reuse them. The feed's own `actions` lists the controls for the feed itself.
- On an **item**, `actions` lists the controls for that item, in order. Each one is a template, possibly with some fields changed, or a complete control of its own.
- The **response** to an action is a small JSON document. It can show a message, replace the item's controls, and replace the item's content, for example with the result of a poll.

Some things it makes possible:

- **A moderation queue.** A feed lists new submissions to an [RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) blog or discussion group. Every item has the buttons *Accept* and *Reject*, a field for the reason of a rejection, *Block user* in a menu, and a drop-down to file it into a category. The moderator works through the queue in their reader.
- **A poll.** An item asks a question, and the options are radio buttons. After the vote, the item shows the results.
- **Simple workflows**: confirm an appointment, rename something, mark a ticket as done with a comment, RSVP to an event, all from a feed.

Who may act is decided by capability URLs in private feeds, or, on the user's own server, by their [Personal Upload Server (RFC-6)](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md). Readers that don't support actions show a link to the publisher's web page instead, using [Fallback HTML (RFC-8)](https://github.com/bigwide-net/RFCs/blob/main/RFC-8-Fallback-HTML.md).

For now, actions are defined for JSON Feed only. The same model can be ported to RSS as a namespace if there is demand (see [Open questions](#open-questions)).

**Why now.** Feeds are good at telling people that something happened, and bad at letting them do anything about it. Whoever runs a discussion group with RFC-4 has to leave their reader to moderate it, a poll in a feed is a link to a web form, and a notification about an appointment is a link to a page with one button on it. The reader already shows the item; it only lacks a way to show the button. Feed HTML ([RFC-2](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md)) rightly forbids forms and scripts in content, so the buttons can't come from the post body. They have to be data that the reader turns into its own controls, under its own rules.

## Principles

1. **Data, not markup.** The feed describes what the user can do. The reader decides how it looks, and draws the controls itself. No forms or scripts in the content.
2. **Nothing happens without a click.** The reader sends a request only when the user triggers an action, never while they type, and never when it shows, prefetches or refreshes the feed.
3. **The feed is the truth.** The response to an action changes what the reader shows right away, but the next fetch of the feed replaces it. Publishers update the feed to make a change last.
4. **Plain HTTP.** An action is one `POST` with a small JSON body, answered with a small JSON body. Any web framework can implement it in a few lines.
5. **Old readers lose nothing.** Every item with actions also links to a page where the user can do the same, shown only by readers without action support.

## The `_actions` object

The extension follows JSON Feed's [extension rules](https://www.jsonfeed.org/version/1.1/#extensions-a-name-extensions-a), like `_source` ([RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md)):

```json
"_actions": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md"
}
```

- It may appear on the **feed**, with `templates` and `actions`, and on an **item**, with `actions` only.
- `about` should be `https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md`, this document. **This is not its final location.** Readers must **ignore the value of `about`** and recognize the extension by the name `_actions` alone.
- A reader ignores fields it doesn't know, so the object can grow later.

## Controls

A control is a JSON object with a `type`:

| Type | What it is | Sends a request |
| --- | --- | --- |
| `button` | A button, or an entry in a menu. | When clicked. |
| `select` | A choice from a fixed list, as a drop-down or radio buttons. | When the user picks a value, if it has an `action`. Otherwise it only provides a value to buttons. |
| `input` | A single-line field for text, a number, a date or a date and time. | Never. It provides a value to buttons. |
| `display` | A short text: an explanation, a heading, or a state such as "Accepted by Tim". | Never. |

A reader ignores a control whose `type` it doesn't know, or that lacks a required field. It ignores unknown values of `state`, `style` and `inputType` and uses the defaults instead, so that new values can be added without breaking older readers.

### Common fields

These fields apply to every type:

| Field | Type | Meaning |
| --- | --- | --- |
| `type` | string | **Required.** `button`, `select`, `input` or `display`. |
| `state` | string | `show` (the default), `disabled` (shown, but can't be used) or `hidden` (not shown). |
| `group` | string | Controls with the same `group` belong together. See [Groups](#groups). |
| `helpText` | string | Plain text, at most 1000 characters, with extra information. Readers show it on request, typically behind a **?** button next to the control. |

All texts in controls are plain text. Readers never interpret them as HTML or Markdown.

### Buttons

| Field | Type | Meaning |
| --- | --- | --- |
| `label` | string | **Required.** The button's text, at most 100 characters. |
| `action` | URL template | **Required.** Where the reader sends the `POST`, see [URL template](#url-template). |
| `name` | string | Sent as `action` in the request body, so that one URL can serve several buttons. See [Names](#names). |
| `style` | string | `major`, `minor` (the default) or `menu`. |
| `confirm` | string | Plain text, at most 200 characters. If present, the reader asks the user to confirm with this question before it sends the request: "Block this user? They can't post or comment anymore." |

| Style | Typical presentation |
| --- | --- |
| `major` | A prominent button, for the action the user most likely wants. |
| `minor` | An ordinary, less prominent button. |
| `menu` | An entry in a menu of further actions (the "⋯" menu), for rare or destructive actions. |

### Selects

| Field | Type | Meaning |
| --- | --- | --- |
| `label` | string | **Required.** At most 100 characters. |
| `values` | array of strings | **Required.** The choices, in order, at most 50. Each is written `value:label`, see [Values](#values). |
| `value` | string | The value that is currently selected. Without it, nothing is selected. |
| `name` | string | Makes the selected value available to buttons and to the select's own request. See [Names](#names). |
| `action` | URL template | If present, the reader sends a request as soon as the user picks a value. Picking the value that is already selected sends nothing. |
| `style` | string | `dropdown` (the default) or `radio`, for radio buttons with all choices visible. Good for polls and short lists. |
| `confirm` | string | As for buttons. Only used with `action`. |
| `errorMessage` | string | Plain text, at most 200 characters. An error shown next to the select, typically in red. |

A select needs a `name`, an `action`, or both. A reader ignores a select with neither.

### Inputs

| Field | Type | Meaning |
| --- | --- | --- |
| `label` | string | **Required.** At most 100 characters. |
| `name` | string | **Required.** See [Names](#names). |
| `inputType` | string | `string` (the default), `number`, `date` or `datetime`. |
| `value` | string or number | The current value: what the field shows when it appears. Without it, the field is empty. |
| `maxLength` | integer > 0 | For `string`: the maximum number of characters. Readers may limit every input to 10,000 characters. |
| `errorMessage` | string | Plain text, at most 200 characters. An error shown next to the input, typically in red. |

| `inputType` | The user enters | Sent as |
| --- | --- | --- |
| `string` | One line of text | A JSON string |
| `number` | A decimal number | A JSON number |
| `date` | A date, with the reader's date picker | A string with an [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) `full-date`: `2026-11-19` |
| `datetime` | A date and a time, in the user's time zone | A string with an RFC 3339 `date-time` including the offset: `2026-11-19T14:30:00+01:00` |

`value` uses the same format. Readers check what the user enters against `inputType` and `maxLength` before they send anything, and show their own error if it doesn't fit.

### Display

| Field | Type | Meaning |
| --- | --- | --- |
| `text` | string | **Required.** The text to show, at most 1000 characters. |
| `header` | boolean | `true` shows the text as a heading, typically bold. Defaults to `false`. |
| `url` | string | An absolute `http` or `https` URL. The reader shows the text as a link to it. |

A display shows information next to the other controls: an explanation ("Rejected posts are deleted after 30 days"), a heading for a group, or the current state of the item ("Accepted by Tim, 10:42"). It has a `helpText` like every other control, but no `label`, `name` or `action`.

With a `url`, the display is a link, e.g. to the post an item is about: "Mention from @bob.example at bob.example/feed.json#p-81". Opening it sends nothing to the feed's server; the reader opens it as it opens links in content, by the rules of [RFC-2](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md#schemes), and shows the text without a link if the URL isn't allowed there. A reader that knows the URL as a feed item (a feed URL + `#id`) may show that item itself. Readers that don't know `url` show the text, which therefore should make sense without the link.

### Values

Each entry of a select's `values` is a string with the value, a colon and the label: `"tech:Tech and Gadgets"`. The value is what the reader sends; the label is what the user sees.

- The string is split at the **first** colon, so labels may contain colons: `"t1:Talk: How feeds work"`.
- An entry without a colon is both value and label: `"Yes"`.
- Values must be unique within a select, and may not be empty.

### Error messages

`errorMessage` lets a feed or a response point at the field that is wrong ("This name is taken"), rather than only showing a general message. Readers show it until the user changes the value of that select or input, and may hide it then.

## Templates

`templates` is an object on the feed's `_actions`. Each key is the **name** of a template, each value a control. Template names consist of lower-case ASCII letters, digits, `-` and `_`, and are at most 64 characters long.

```json
"_actions": {
  "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md",
  "templates": {
    "accept": {
      "type": "button",
      "label": "Accept",
      "style": "major",
      "action": "https://api.example.com/api/accept/{id}"
    },
    "category": {
      "type": "select",
      "label": "Category",
      "name": "category",
      "values": ["tech:Tech and Gadgets", "news:News", "misc:Other"],
      "action": "https://api.example.com/api/category/{id}"
    }
  }
}
```

A template may lack fields that are required for its type, such as the `text` of a display, if every control that uses it provides them.

## Declaring controls

A template does nothing by itself. The feed and each item list their controls in an `actions` array:

```json
"_actions": {
  "actions": [
    { "template": "accept" },
    { "template": "category", "value": "news" },
    { "type": "display", "text": "Submitted twice before." }
  ]
}
```

Each entry is a control:

- **With `template`**: the template of that name, with the entry's other fields replacing the template's. Every field except `type` can be replaced, including `action`, `label`, `values`, `value` and `errorMessage`. An entry whose template doesn't exist is ignored.
- **Without `template`**: a complete control, which must have a `type` and every required field.

Where they go:

- **Feed actions** concern the feed as a whole, such as "Pause submissions" for a moderation queue. They go into the feed's `_actions.actions`, next to `templates`. Readers show them near the feed's title or in the feed's menu.
- **Item actions** go into the item's `_actions.actions`. Readers show them with the item, usually below its content. Items have no `templates` of their own: an item only uses the feed's.

Replacing fields per item is what makes most uses work: a poll item sets `value` to the user's vote, a moderation item sets `state` to `disabled` once it is handled, a display shows each item's own state, and a feed that uses one capability URL per item replaces `action`.

### Order

Readers show the controls in the order of the array. The order is a recommendation, not a promise:

- Readers may collect `menu` buttons into one common menu, together with the reader's own entries for the item, such as "Share" or "Mark as read".
- On a small screen, a reader may wrap controls onto several lines, or put buttons into a menu.

Readers keep the order of the choices within a select, and never leave out a control that is shown.

### Groups

Controls with the same `group` belong together, like the fields of a small form: a reader shows them next to each other, for example on one line, at the position of the group's first control. The group also decides which values a button sends, see [What is sent](#what-is-sent).

A group is a string of at most 64 characters, compared exactly. It is never shown to the user; give the group a display with `header` if it needs a title. Controls without a `group` form one group together.

```json
"actions": [
  { "template": "accept", "group": "accept" },
  { "type": "input", "name": "reason", "label": "Reason for rejection", "maxLength": 200, "group": "reject" },
  { "template": "reject", "group": "reject" }
]
```

Here *Reject* sends the reason, and *Accept* sends nothing, because the reason is not in its group. A reader may show the two groups on two lines: *Accept* on the first, the reason and *Reject* on the second.

### Hidden controls

`hidden` is different from leaving a control out: a hidden input or select still provides its value, like a hidden field in an HTML form. A disabled one doesn't, as in HTML. Feeds can use hidden inputs to pass a fixed value to a button without putting it into the URL.

## Performing an action

When the user clicks a button, or picks a value in a select with an `action`, the reader, after asking for confirmation if the control has `confirm`, sends a `POST` request to the control's URL. Only controls that are shown and enabled can be triggered.

### Names

`name` gives an input or a select a name under which its value is sent. A button's `name` identifies the button itself. Names consist of lower-case ASCII letters, digits and `_`, are at most 64 characters long, and must not be `id`, which is reserved for the item's id.

Names must be unique within a group. If two controls in a group share a name, readers use the first and ignore the name of the others. The same name may appear in different groups: two groups can each have an input named `comment`.

### What is sent

A triggered button or select sends the values of every named input and select **in its own group** that is not disabled, including its own value if it is a select. Controls without a `group` count as one group, so in an item without any groups, every button sends every value. The name of the group is never sent.

An empty input, and a select with nothing selected, send `null` in the body and an empty string in the URL.

### URL template

The `action` field is a [URI Template (RFC 6570)](https://www.rfc-editor.org/rfc/rfc6570). Its variables are:

- `{id}`: the item's `id`.
- `{name}`, for every named input and select whose value is sent: the value as text, as entered (numbers as decimals, dates as in the table above).

All variables use simple string expansion, so every character outside the unreserved set is percent-encoded, and values can safely go into a path or a query parameter. Variables without a value expand to an empty string.

```
Template:  https://api.example.com/api/changeName?name={newname}&id={id}
Values:    newname = "Tim & Co", id = "https://example.com/posts/42"
Request:   https://api.example.com/api/changeName?name=Tim%20%26%20Co&id=https%3A%2F%2Fexample.com%2Fposts%2F42
```

Feed actions have no item, so their URL must not contain `{id}`. Readers ignore a feed control whose URL does.

The expanded URL must be an absolute `https` URL. Readers ignore controls whose URL template doesn't start with `https://`, and refuse to send a request whose URL points to `localhost`, `*.local`, `*.localhost` or a private network, as [RFC-2](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md#schemes) does for links.

### Request

The body is a JSON object:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | The item's `id`. Absent for feed actions. |
| `action` | string | The `name` of the button or select that was triggered. Absent if it has none. |
| `values` | object | The values that are sent, by name, see [What is sent](#what-is-sent). Absent if there are none. |

```http
POST /api/reject/https%3A%2F%2Fexample.com%2Fposts%2F42 HTTP/1.1
Host: api.example.com
Content-Type: application/json
Accept: application/json

{
  "id": "https://example.com/posts/42",
  "action": "reject",
  "values": { "reason": "Off-topic for this group." }
}
```

The values are in the body even if the URL template uses them too, so that a server can choose either.

### Response

The server answers with `Content-Type: application/json` and an object whose fields are all optional:

| Field | Type | Meaning |
| --- | --- | --- |
| `message` | string | Plain text, at most 500 characters, shown to the user, e.g. as a short notification. |
| `actions` | array | The new controls of the same item, or of the feed for a feed action. They replace the old ones completely, in the same form as an `actions` declaration. |
| `html` | string | New content for the item, in [Feed HTML (RFC-2)](https://github.com/bigwide-net/RFCs/blob/main/RFC-2-HTML-Subset-for-Feed-Content.md). |
| `markdown` | string | New content for the item, in Markdown. Preferred over `html`. |

```json
{
  "message": "Post accepted",
  "actions": [
    { "type": "display", "text": "Accepted by you, 10:42" },
    { "template": "undo" },
    { "template": "category" }
  ]
}
```

**`actions`.** Without `actions`, the controls stay as they are, including what the user entered. With it, the reader replaces the item's controls by the new list, and inputs and selects show the `value` the new list gives them. To mark a field as wrong, the server sends the controls again, with an `errorMessage` on that field and the user's input as its `value`. An empty array removes all controls.

**`html` and `markdown`.** Either one replaces the content of the item in the reader: the body it shows, whether it came from `content_html`, `content_text` or a Markdown version. Title, images and attachments stay. This gives direct feedback in the item itself: the results of a poll after the vote, a confirmation with the details of the appointment, the post with a note that it was accepted.

- A response contains at most one of them. Markdown is preferred: it is simpler to write and safer to render. If a response contains both, readers use `markdown`.
- Readers treat the new content like the item's original content: they sanitize HTML as Feed HTML, remove Fallback HTML ([RFC-8](https://github.com/bigwide-net/RFCs/blob/main/RFC-8-Fallback-HTML.md)) for the extensions they support, and resolve relative URLs against the item's `url`.
- The new content is limited to 128 KiB, as RFC-2 limits post bodies.
- For a feed action, `html` and `markdown` are ignored: the feed has no content to replace.

An empty body, or `{}`, means success with nothing to change.

**Status.** A `2xx` status means the action succeeded. Any other status means it failed: the reader shows the `message` as an error, or a generic error if there is none, and puts a select that triggered the request back to its previous value. It still applies `actions`, `html` and `markdown`, so that a server can say "Someone else already accepted this post", or mark a field as wrong, and update the item at the same time. A redirect is followed only to another `https` URL; a response that is not JSON is treated as an error without message.

**Until the next fetch.** Changes from responses, to the controls and to the content, last until the reader fetches the feed again; then the feed replaces them. A publisher who wants the change to stay, which is nearly always, updates the feed before answering, or at least before the next fetch. A reader that caches items keeps the changes in its cache until then.

### While the request runs

- The reader **should** show a loading indicator on the triggered control, or on the item, until the response arrives.
- It disables the controls of the group meanwhile, so that the request can't be sent twice, and shows the new value of a select right away.
- It never repeats a request on its own. If the network fails, it says so, keeps what the user entered, and lets them try again.
- Servers should make actions idempotent where possible: accepting an accepted post accepts it, voting twice for the same option counts once.

## Authorization

Actions add no login of their own. Two mechanisms decide who may act:

- **Capability URLs**, for any server: whoever can read the feed can act on it.
- **PUPS**, for the user's own server: the reader proves that the user is the owner of the domain, with the token it already has from the user's [Personal Upload Server (RFC-6)](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md).

Readers send no cookies with action requests, and no credentials other than a PUPS token under the rules below, or an identity token from [RFC-10](https://github.com/bigwide-net/RFCs/blob/main/RFC-10-Identity-Tokens.md) under its rules. A feed must never be able to make the reader act on the user's behalf on a site the user happens to be logged into.

### Capability URLs

A feed whose actions only certain people may perform is private, and its action URLs contain a secret that only the feed's subscribers know: `https://api.example.com/api/accept/{id}?key=7f3c…`. Whoever can read the feed can act on it. A feed for many users, such as a poll that allows one vote per person, gives every subscriber their own feed URL, with their own keys in the action URLs.

Capability URLs need nothing from the reader, and work for every server.

### PUPS on the user's own server

A user who runs PUPS on their domain can act on feeds that their own server publishes, such as moderating the comments on their own blog, without any keys in URLs. The reader signs in with the user's domain, as RFC-6 describes, and sends its access token with the action requests.

**What PUPS adds.** This RFC extends RFC-6 with two things:

- **`actions_base`**, a new field in the PUPS configuration document: the base URL under which the server publishes feeds with actions and receives their requests, e.g. `https://tim.example.com/pups/actions/`. It ends with `/`, uses `https`, and must not overlap with any path that apps can write through PUPS (`storage_base`, `public_base` and the well-known paths). Without it, the server offers no actions.
- **The scope `actions`**: permission to perform actions on the owner's server. The consent screen shows it as "Act on your behalf on your own server, e.g. moderate comments". Like `identify`, it grants nothing to write.

**When the reader sends the token.** The reader sends `Authorization: Bearer <access token>` only if all of these are true:

1. It holds a token with the scope `actions` from the user's own PUPS server.
2. The feed's URL starts with that server's `actions_base`.
3. The action's expanded URL starts with the same `actions_base`.

It also sends the token when it fetches a feed whose URL starts with `actions_base`, so that such feeds can be private without a key in their URL. It sends the token nowhere else. This is an exception to RFC-6's rule that access tokens are only sent to the upload API, and the reason for rule 2: only the owner's server can publish a feed under `actions_base`, so no other feed, and no app that writes files through PUPS, can make the reader use the token.

**On the server.** The action handler accepts a request if the token is valid and has the scope `actions`. A PUPS installation has exactly one owner, so a valid token means that the owner acts. The handler is usually part of the PUPS server, or a service on the same domain that checks tokens with the PUPS server, e.g. by [token introspection (RFC 7662)](https://www.rfc-editor.org/rfc/rfc7662). How it does that is up to the installation.

**Without a token.** A reader that is not signed in, or whose token lacks the scope, sends the request without it. The server answers `401` with a `message` such as "Sign in with your domain to moderate". A reader that supports PUPS then offers the user to sign in, or to grant the missing scope.

**Example.** Tim runs an RFC-4 server for comments on his blog at `tim.example.com`, as part of his PUPS installation, with `actions_base` `https://tim.example.com/pups/actions/`. It publishes the moderation queue at `https://tim.example.com/pups/actions/moderation.json`, with action URLs like `https://tim.example.com/pups/actions/accept/{id}`. Tim's reader, signed in to his domain with the scope `actions`, fetches the queue and accepts comments with his token. Anyone else who finds the URL gets a `401`.

**Only the user's own server, for now.** The token proves to the user's own server that the owner acts, and to nobody else. Proving who the user is to other servers, for example to vote once in a poll on someone else's blog, is the subject of [Identity Tokens (RFC-10)](https://github.com/bigwide-net/RFCs/blob/main/RFC-10-Identity-Tokens.md), see below.

### Acting as the user on other servers

[Identity Tokens (RFC-10)](https://github.com/bigwide-net/RFCs/blob/main/RFC-10-Identity-Tokens.md) adds the field `identity` to buttons and selects. A control with `"identity": true` performs its action as the user: the reader shows the user's handle with the control, e.g. *Sent as @tim.example.com*, and sends an identity token from the user's PUPS server with the request. RFC-10 defines the field, what readers show, and how servers check the token.

**Readers that don't support RFC-10.** Readers ignore unknown fields, but not this one. A reader that ignored it would offer an action that can only fail, because the server needs a token that the reader can't send. Therefore a reader that doesn't support RFC-10:

- shows **none** of an item's controls if any of them, after applying templates, has `identity` set to `true`, whatever its `state`. It keeps the item's `actions` fallback, as if it didn't support actions at all (see [Fallback HTML](#fallback-html));
- shows none of the feed's actions if any of them has `identity`;
- removes all controls of an item when a response gives it such a control, and shows the item with its fallback again.

`identity` is the only field with this rule. A later field that changes what an action does, so that a reader must not offer the action without understanding it, gets the same rule by being added to this section.

## CORS

Browser-based readers send action requests from their own origin, and the JSON body triggers a [CORS preflight](https://fetch.spec.whatwg.org/#cors-preflight-fetch). Action servers must answer it:

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Authorization, Content-Type
```

and send `Access-Control-Allow-Origin: *` with the response itself, so that the reader can read it. The wildcard is safe, because PUPS tokens are bearer tokens that the reader sends explicitly, never cookies. A server under `actions_base` sends the same headers for its feeds, so that browser-based readers can fetch them with the token.

## Fallback HTML

Readers without action support don't know `_actions`, and would show the item without any way to act on it. Publishers therefore add a link to a web page where the user can do the same, as fallback HTML ([RFC-8](https://github.com/bigwide-net/RFCs/blob/main/RFC-8-Fallback-HTML.md)) with the extension name **`actions`**:

```html
<p class="without-extension-actions">
  <a href="https://example.com/moderation/42">Accept or reject this post</a>
</p>
```

- A reader **supports `actions`** when it shows item controls of all four types and performs actions as this RFC describes, including replacing the content from a response. A reader that supports only some types must keep the fallback.
- A reader that supports `actions` removes the fallback, even for an item whose controls are all hidden or disabled. The only exception is an item whose controls it doesn't show because one of them has `identity` and the reader doesn't support RFC-10 (see [Acting as the user on other servers](#acting-as-the-user-on-other-servers)): it keeps the fallback.
- Feed actions have no fallback: JSON Feed has no HTML for the feed itself. Publishers mention the page for feed actions in the feed's `description` or on the `home_page_url`, if they matter.
- A link in the fallback must not perform the action itself (e.g. `GET /accept/42`): link checkers, previews and mail scanners follow links. It should lead to a page where the user confirms.

## Notes for publishers and readers

**Publishers**

- Always add fallback HTML to items with actions, unless every subscriber is known to use a reader that supports them.
- Keep labels short. Readers may truncate them, and on a phone three buttons share one line.
- Put destructive actions (*Block user*, *Delete*) into the `menu` style, and give them `confirm`.
- Use groups whenever an item has more than one button, so that each button sends only the values that belong to it.
- Check every value on the server. Readers check `inputType` and `maxLength`, but anyone can send a request by hand. Report problems with `errorMessage` on the field.
- Update the feed when an action changes something, so the next fetch shows the same state as the response. The response is for the moment, the feed for later.
- Don't put anything secret into the feed that a subscriber shouldn't see. Capability keys in action URLs are fine for a private feed; they are no protection in a public one.
- Answer quickly. The user is waiting for the button to come back.

**Readers**

- Send requests only when the user triggers an action, and only for controls that are shown and enabled.
- Show a loading indicator while a request runs.
- Never send cookies, and send a PUPS token only to the user's own server, as described in [PUPS on the user's own server](#pups-on-the-users-own-server). Identity tokens follow [RFC-10](https://github.com/bigwide-net/RFCs/blob/main/RFC-10-Identity-Tokens.md).
- Treat labels, texts, values and messages as plain text. Only `html` and `markdown` in a response are content, and they go through the same sanitizing as the item's own.
- Show which host an action talks to where the user can see it, such as in the confirmation dialog or a tooltip, at least when it differs from the feed's host.
- Keep changes from responses until the next fetch, then let the feed win.

## Full example

A moderation queue for a discussion group. The feed itself has a *Pause submissions* action. Every pending item shows its state, *Accept*, *Reject* with a reason, *Block user* in the menu, and a category. The second item is already accepted and filed as *News*.

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Moderation: Feed Readers group",
  "home_page_url": "https://groups.example.com/feed-readers/moderation",
  "feed_url": "https://groups.example.com/feed-readers/moderation.json?key=7f3c9a",
  "_actions": {
    "about": "https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md",
    "templates": {
      "status": {
        "type": "display",
        "header": true
      },
      "accept": {
        "type": "button",
        "label": "Accept",
        "style": "major",
        "group": "accept",
        "action": "https://api.example.com/feed-readers/accept/{id}?key=7f3c9a"
      },
      "reason": {
        "type": "input",
        "name": "reason",
        "label": "Reason for rejection",
        "maxLength": 200,
        "group": "reject",
        "helpText": "Sent to the author with the rejection. Leave empty to reject without a reason."
      },
      "reject": {
        "type": "button",
        "label": "Reject",
        "group": "reject",
        "action": "https://api.example.com/feed-readers/reject/{id}?key=7f3c9a"
      },
      "undo": {
        "type": "button",
        "label": "Undo",
        "action": "https://api.example.com/feed-readers/undo/{id}?key=7f3c9a"
      },
      "block": {
        "type": "button",
        "label": "Block user",
        "style": "menu",
        "confirm": "Block this user? They can't post or comment in this group anymore.",
        "action": "https://api.example.com/feed-readers/block/{id}?key=7f3c9a"
      },
      "category": {
        "type": "select",
        "label": "Category",
        "name": "category",
        "values": ["tech:Tech and Gadgets", "news:News", "misc:Other"],
        "action": "https://api.example.com/feed-readers/category/{id}?key=7f3c9a"
      },
      "pause": {
        "type": "button",
        "label": "Pause submissions",
        "style": "menu",
        "action": "https://api.example.com/feed-readers/pause?key=7f3c9a"
      }
    },
    "actions": [
      { "template": "pause" }
    ]
  },
  "items": [
    {
      "id": "https://alice.example.org/feed.json#c17",
      "url": "https://alice.example.org/posts/c17",
      "title": "Which reader shows comment counts?",
      "authors": [{ "name": "Alice", "url": "https://alice.example.org/" }],
      "date_published": "2026-11-18T09:12:00Z",
      "content_html": "<p>I'm looking for a reader that shows the total comment count, not just direct replies.</p><p class=\"without-extension-actions\"><a href=\"https://groups.example.com/feed-readers/moderation/c17\">Moderate this post</a></p>",
      "_actions": {
        "actions": [
          { "template": "status", "text": "Waiting for review" },
          { "template": "accept" },
          { "template": "reason" },
          { "template": "reject" },
          { "template": "block" },
          { "template": "category" }
        ]
      }
    },
    {
      "id": "https://bob.example.net/feed.json#88",
      "url": "https://bob.example.net/posts/88",
      "title": "Downread 2.0 is out",
      "authors": [{ "name": "Bob", "url": "https://bob.example.net/" }],
      "date_published": "2026-11-18T08:40:00Z",
      "content_html": "<p>With galleries and feed actions.</p><p class=\"without-extension-actions\"><a href=\"https://groups.example.com/feed-readers/moderation/88\">Moderate this post</a></p>",
      "_actions": {
        "actions": [
          { "template": "status", "text": "Accepted by Tim, 08:51" },
          { "template": "undo" },
          { "template": "block" },
          { "template": "category", "value": "news" }
        ]
      }
    }
  ]
}
```

The moderator types a reason into Alice's post and clicks *Reject*. The button is in the group `reject`, so the reader sends the reason, but not the category:

```http
POST /feed-readers/reject/https%3A%2F%2Falice.example.org%2Ffeed.json%23c17?key=7f3c9a HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "id": "https://alice.example.org/feed.json#c17",
  "values": { "reason": "Please ask this in the Help group." }
}
```

and gets:

```json
{
  "message": "Post rejected",
  "actions": [
    { "template": "status", "text": "Rejected by you, 10:42" },
    { "template": "undo" },
    { "template": "block" }
  ]
}
```

### A poll

A poll is one item with a `radio` select that votes as soon as the user picks an option. The feed is personal (one URL per subscriber), so the action URL can identify the voter:

```json
{
  "id": "https://polls.example.com/p/lunch",
  "url": "https://polls.example.com/p/lunch",
  "title": "What should we order on Friday?",
  "content_html": "<p>Vote until Thursday noon.</p><p class=\"without-extension-actions\"><a href=\"https://polls.example.com/p/lunch\">Vote on the website</a></p>",
  "_actions": {
    "actions": [
      {
        "type": "select",
        "label": "Your vote",
        "name": "vote",
        "style": "radio",
        "values": ["pizza:Pizza", "sushi:Sushi", "tacos:Tacos"],
        "action": "https://polls.example.com/vote/{id}?voter=k29d1e"
      }
    ]
  }
}
```

The user picks *Pizza*. The reader sends `{"id": "https://polls.example.com/p/lunch", "values": {"vote": "pizza"}}`, and the server answers with the results as the new content, and a vote that can't be changed anymore:

```json
{
  "message": "Thanks for voting!",
  "markdown": "Vote until Thursday noon.\n\n**Results after 28 votes**\n\n| Option | Votes |\n| --- | --- |\n| Pizza | 12 |\n| Tacos | 9 |\n| Sushi | 7 |",
  "actions": [
    { "type": "display", "text": "You voted for Pizza.", "helpText": "Each subscriber has one vote. The poll closes on Thursday at noon." }
  ]
}
```

The next version of the feed carries the same content and controls for this subscriber, so the result is still there after a refresh.

### Renaming

An input and a button in the same group, with the value in the URL:

```json
"actions": [
  { "type": "input", "name": "newname", "label": "New name", "value": "Feed Readers", "maxLength": 60, "group": "rename" },
  { "type": "button", "label": "Rename", "group": "rename",
    "action": "https://api.example.com/api/changeName?name={newname}&id={id}" }
]
```

If the name is taken, the server answers with status `409` and the same controls, with an error on the field:

```json
{
  "message": "The group was not renamed.",
  "actions": [
    { "type": "input", "name": "newname", "label": "New name", "value": "RSS", "maxLength": 60, "group": "rename",
      "errorMessage": "There is already a group with this name." },
    { "type": "button", "label": "Rename", "group": "rename",
      "action": "https://api.example.com/api/changeName?name={newname}&id={id}" }
  ]
}
```

## Open questions

- **RSS.** Should actions be ported to RSS as a namespace, with `<actions:template>` on the channel and `<actions:control>` on items? The model maps directly; it was left out to keep the first version small.
- **Default item actions.** A moderation queue repeats the same `actions` on every item. Should the feed be able to declare default controls for all items, which items then change?
- **More inputs.** Multi-line text, multiple choice, checkboxes, `required`, and `min`/`max` for numbers and dates are obvious next steps. Each needs a rule for the request body and for readers that don't know it.
- **Partial updates.** A response replaces all controls of an item. Should controls have an `id`, so that a response can change a single one?
- **Values format.** `value:label` strings are compact. Objects (`{"value": "tech", "label": "Tech"}`) would be easier to extend, e.g. with a description per choice.
