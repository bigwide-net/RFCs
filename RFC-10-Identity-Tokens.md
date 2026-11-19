# RFC: Identity Tokens - Acting as Your Domain on Other Servers

Nov 19, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-10-Identity-Tokens.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-10-Identity-Tokens.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-10-Identity-Tokens.md)

## Summary

This proposal lets an app prove to any server that it acts for the owner of a domain, without a redirect and without a login at that server. The app asks the user's [Personal Upload Server (RFC-6)](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md) for an **identity token**: a short-lived signed token that says "this request comes from `tim.example.com`, for `polls.example.com`, through this app". The app sends it with its request. The server checks the signature with a key that it finds through the user's domain, and knows who acts.

```http
POST /vote/https%3A%2F%2Fpolls.example.com%2Fp%2Flunch HTTP/1.1
Host: polls.example.com
Authorization: Bearer eyJ0eXAiOiJhdCtqd3QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjIwMjYtMTEifQ.…
Content-Type: application/json

{ "id": "https://polls.example.com/p/lunch", "values": { "vote": "pizza" } }
```

The protocol is built from existing specs:

- **Getting a token**: [OAuth 2.0 Token Exchange (RFC 8693)](https://www.rfc-editor.org/rfc/rfc8693) at the PUPS token endpoint. The app trades its PUPS access token for a token for another server.
- **The token**: a JWT in the [JWT Profile for OAuth 2.0 Access Tokens (RFC 9068)](https://www.rfc-editor.org/rfc/rfc9068), signed by the PUPS server, valid for one server and at most five minutes.
- **Sending it**: an ordinary bearer token ([RFC 6750](https://www.rfc-editor.org/rfc/rfc6750)).
- **Checking it**: the server finds the PUPS server through the domain in the token, as RFC-6 discovery does, and its public keys in a [JWK Set (RFC 7517)](https://www.rfc-editor.org/rfc/rfc7517).

It also adds the field `identity` to [Feed Actions (RFC-9)](https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md). A control with `"identity": true` performs its action as the user. The reader shows this next to the control, with the user's handle, e.g. *Sent as @tim.example.com*, and sends an identity token when the user clicks.

```json
{
  "type": "select",
  "label": "Your vote",
  "name": "vote",
  "style": "radio",
  "values": ["pizza:Pizza", "sushi:Sushi", "tacos:Tacos"],
  "action": "https://polls.example.com/vote/{id}",
  "identity": true
}
```

Some things it makes possible:

- **A poll in a public feed** with one vote per domain, instead of one feed URL per subscriber.
- **A discussion group with several moderators.** The group's server keeps a list of moderators' domains. Each moderator acts from their own reader, and removing a moderator means removing a line from the list, not rotating a secret key that everyone shares.
- **RSVPs, sign-ups and requests** that the server needs to attribute to a person: "@tim.example.com will attend".
- **Personal feeds.** A reader fetches a feed as the user, and the server shows them their own content at the same URL as everyone else: replies to their comments, the state of their submissions, the moderation queues of their groups. On a public [Feed Submissions (RFC-4)](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) server such as rssmention, that needs no account, just like commenting there.

**Why now.** RFC-9 left this open: PUPS proves who the user is only to the user's own server. Elsewhere, only capability URLs decide who may act, so anything personal needs a feed URL per subscriber with secret keys in it, and the publisher must know every subscriber in advance. "Sign in with your domain" from RFC-6 does prove identity to any app, but it needs a redirect through the user's PUPS server and an owner login for every server, and every server would have to be an OAuth client. That is fine for signing in to an app once. It is far too much for a click on a poll in a feed. The reader already holds a PUPS token. It only needs a way to turn it into a proof that one other server can check.

## Goals and non-goals

**Goals**

- **No redirect, no registration.** The user acts on a server they have never visited, with one click, and the server needs no account, client registration or user database.
- **The user always knows.** An app uses the user's identity only when the user does something that visibly says so, and only towards the server named there.
- **No keys to manage.** The user's PUPS server holds the only signing key. It creates it on installation. Neither the user nor any app ever sees or handles a key.
- **Standard tools.** Servers check tokens with any JWT library, and cache what they fetch. Apps need no cryptography.
- **The owner stays in control.** Each app needs a grant with step-up confirmation. The owner sees where each app acted for them, and can revoke it at any time.

**Non-goals**

- **Who the person is.** A token proves control of a domain, nothing more: not a real name, not that two domains belong to different people.
- **Authorization.** What a domain may do on a server (vote, moderate, nothing) is the server's decision.
- **Sessions.** A token is good for one request. Servers that want sessions build them themselves.
- **Anonymous or pseudonymous credentials.** The domain is the identity, and the server learns it.

## What is new and what is reused

**Used as they are.**

| Spec | Used for |
| --- | --- |
| [RFC 8693](https://www.rfc-editor.org/rfc/rfc8693) token exchange | The app gets an identity token for a server at the PUPS token endpoint. |
| [RFC 9068](https://www.rfc-editor.org/rfc/rfc9068) JWT access tokens, with [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519) and [RFC 7515](https://www.rfc-editor.org/rfc/rfc7515) | The format of the token and its claims. |
| [RFC 7517](https://www.rfc-editor.org/rfc/rfc7517) JWK Set, `jwks_uri` from [RFC 8414](https://www.rfc-editor.org/rfc/rfc8414) | Publishing the PUPS server's public keys. |
| [RFC 6750](https://www.rfc-editor.org/rfc/rfc6750) bearer tokens | Sending the token to the server. |
| RFC-6 discovery and `me` | Finding the PUPS server, and so its keys, from the domain in the token. |

**Used with stricter rules.**

| Spec | Rules of this RFC |
| --- | --- |
| RFC 8693 | Only the PUPS access token as subject token, one `audience`, which must be an `https` origin, and no other parameters. |
| RFC 9068 | `RS256` only. `aud` is a single origin, `sub` is the owner's `me`, and tokens live at most five minutes. |

**New in this RFC.**

- **The scope `identity-tokens`**, its consent screen and its log. See [The PUPS side](#the-pups-side).
- **`jwks_uri` in the configuration document** of RFC-6. The name comes from RFC 8414.
- **Finding the issuer from the subject.** An RFC 9068 server normally knows its issuers in advance. Here, any domain can be the subject, and the server finds the issuer that the domain names. See [Checking a token](#checking-a-token).
- **The `identity` field for Feed Actions**, and what readers show for it. See [Feed Actions](#feed-actions).
- **Subscribing to a feed as the user.** See [Personal feeds](#personal-feeds).

## Roles

- **Owner**: the person who controls the domain, e.g. `tim.example.com`, and runs a PUPS server for it, as in RFC-6.
- **PUPS server**: the owner's server. It issues identity tokens and publishes the keys to check them.
- **App**: the software that acts for the owner, typically a feed reader. It holds a PUPS grant with the scope `identity-tokens`.
- **Server**: any server that receives requests from apps and wants to know who sends them, e.g. a poll, a discussion group, an action endpoint of RFC-9 or a server of personal feeds. In OAuth terms it is the resource server, and the token's audience.

## Overview

1. **Grant, once.** The user connects the app to their domain, as in RFC-6, and grants the scope `identity-tokens` with step-up confirmation. The consent screen says that the app may act as `tim.example.com` on other sites.
2. **Disclosure.** Something in the app says that an action is performed as the user, and names the user's handle and the server, e.g. next to a poll in a feed: *Sent as @tim.example.com to polls.example.com*.
3. **Click.** The user acts.
4. **Exchange.** The app sends its PUPS access token to the PUPS token endpoint and asks for a token for `https://polls.example.com`. The PUPS server checks the grant, logs the request and returns a signed token, valid for 60 seconds.
5. **Request.** The app sends its request to `polls.example.com` with `Authorization: Bearer <token>`.
6. **Check.** The server checks the token's audience and lifetime, fetches `https://tim.example.com/.well-known/host-meta.json`, follows it to the PUPS configuration document and its keys (all of which it caches), and checks the signature. It now knows that `tim.example.com` acts, through the app named in the token.

## The token

An identity token is a JWT that follows RFC 9068, signed with `RS256`.

```json
{
  "typ": "at+jwt",
  "alg": "RS256",
  "kid": "2026-11"
}
```

```json
{
  "iss": "https://tim.example.com",
  "sub": "https://tim.example.com/",
  "aud": "https://polls.example.com",
  "client_id": "https://app.example/oauth/client.json",
  "iat": 1795084920,
  "exp": 1795084980,
  "jti": "Jm1d6Zr0qKcXb9pQ2vY7sA"
}
```

| Claim | Meaning |
| --- | --- |
| `iss` | The PUPS server's `issuer`, exactly as in its configuration document. |
| `sub` | The owner's identity: `me` from RFC-6, `https://<host>/`. The host is lowercase, in ASCII form, without port; the path is `/`. |
| `aud` | The server the token is for: one `https` origin, as a single string, e.g. `https://polls.example.com`, or `https://polls.example.com:8443` with a port other than 443. Never an array. |
| `client_id` | The app's `client_id` from RFC-6. The PUPS server verified it when the owner granted access. |
| `iat`, `exp` | Issued at, and expiry, in seconds since 1970 (UTC). `exp` is at most 300 seconds after `iat`; 60 seconds is recommended. |
| `jti` | A unique identifier, at least 128 bits from a secure random generator. |

The header's `typ` is `at+jwt`, as RFC 9068 requires, and `kid` names the key in the PUPS server's JWK Set. The token carries no `scope`: it grants nothing by itself, it only says who acts.

**The handle.** The user's handle is `@` followed by the host of `sub`, e.g. `@tim.example.com`, as in a [socialspec](https://socialspec.org) account. Apps show it to the user, and servers to everyone else.

## The PUPS side

This RFC extends RFC-6 with a scope, a grant type at the token endpoint and a key set.

### Configuration document

Two changes to the configuration document of RFC-6:

- `scopes_supported` contains `identity-tokens`.
- **`jwks_uri`**: the URL of the server's JWK Set, e.g. `https://tim.example.com/pups/jwks.json`. It uses `https`.

```json
{
  "issuer": "https://tim.example.com",
  "token_endpoint": "https://tim.example.com/oauth/token",
  "scopes_supported": ["identify", "write:/appdata/", "identity-tokens"],
  "jwks_uri": "https://tim.example.com/pups/jwks.json",
  "…": "…"
}
```

A server without `jwks_uri`, or without `identity-tokens` in `scopes_supported`, doesn't issue identity tokens. The RFC 8414 metadata lists the same `jwks_uri`, and `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported`.

**Nothing an app can write.** Whoever can change the configuration document or the JWK Set can sign in as the owner everywhere. Both must be at paths that no app can write through PUPS: not under `storage_base` or `public_base`, and not in a declared well-known path. RFC-6 already keeps apps out of `host-meta.json` and its PUPS link.

### Scope and consent

The scope **`identity-tokens`** lets an app get identity tokens for any server. It grants nothing to write.

- The consent screen shows it as a group of its own, after the four groups of RFC-6: "**act as tim.example.com on other sites**, e.g. vote in polls, moderate a group or read your personal feeds there". The details say that the app can prove to any site that it acts for the owner, whenever the owner uses it there, and that the apps page shows where it did.
- It needs **step-up confirmation**, like the well-known scopes, and the server never skips the consent screen for a request that contains it.
- A grant with this scope always comes with a refresh token, because the app needs a valid access token for every exchange.

### Token exchange

The app sends a token exchange request to the `token_endpoint`:

```http
POST /oauth/token HTTP/1.1
Host: tim.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&client_id=https%3A%2F%2Fapp.example%2Foauth%2Fclient.json
&subject_token=<access token>
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aaccess_token
&audience=https%3A%2F%2Fpolls.example.com
```

The server checks:

1. `subject_token` is a valid access token of this server, of a grant with the scope `identity-tokens`, issued to the `client_id` of the request. Otherwise: `invalid_grant`, or `invalid_scope` if the grant lacks the scope.
2. `audience` is exactly one `https` origin: scheme, lowercase host in ASCII form, and a port only if it isn't 443; no path, not even `/`, no query, user info or fragment. The host is not `localhost`, `*.local`, `*.localhost` or an IP address. Otherwise: `invalid_target`.
3. The request contains no `resource`, `scope`, `actor_token` or `requested_token_type` other than `urn:ietf:params:oauth:token-type:jwt`. Otherwise: `invalid_request`.
4. The grant is within its rate limit, e.g. 60 tokens per minute. Otherwise: `429`.

It answers:

```json
{
  "access_token": "eyJ0eXAiOiJhdCtqd3QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjIwMjYtMTEifQ.…",
  "issued_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "token_type": "Bearer",
  "expires_in": 60
}
```

The field is called `access_token` because RFC 8693 calls it that; the token is the identity token. The PUPS server stores no identity tokens, but it logs each one it issues (see [Log](#log)). The token endpoint is already CORS-enabled (RFC-6), so browser apps can call it.

This is a second exception to RFC-6's rule that access tokens are valid only at the upload API: the app also sends its access token to the token endpoint, as the subject of the exchange.

### Keys

- **One key pair, made by the server.** The server creates an RSA key pair of at least 2048 bits during setup. The private key never leaves the server: it is kept outside the storage and outside the web root (or as a secret of the hosting platform), never uploaded, never shown, and never needs anything from the owner.
- **The JWK Set** at `jwks_uri` contains the public keys, each with `kty` `RSA`, `use` `sig`, `alg` `RS256` and a `kid`. It is served as `application/json` with `Access-Control-Allow-Origin: *` and `Cache-Control: max-age` of at most 3600 seconds.

```json
{
  "keys": [
    { "kty": "RSA", "use": "sig", "alg": "RS256", "kid": "2026-11", "n": "0vx7agoebGcQSuu…", "e": "AQAB" }
  ]
}
```

- **Rotation.** To change the key, the server adds the new public key to the JWK Set, waits until every cache has seen it (the JWK Set's `max-age`, at most an hour), and then signs with the new key. It removes the old key five minutes after it signed the last token with it.
- **Compromise.** If the private key may have leaked, the server removes it from the JWK Set at once and creates a new one. Servers stop accepting the old key within their cache time, at most an hour. The apps page should offer the owner a button for this.

### Log

The PUPS server records every identity token it issues: the time, the `client_id` and the `aud`. The apps page shows, for each app, the sites it acted on, how often and when it last did. The log is private to the owner, and kept for at least 30 days.

When the owner revokes a grant, the app gets no more identity tokens. Tokens it already has expire within five minutes. A server may let the owner block single audiences, for one app or for all.

## Using identity tokens in an app

An app that holds the scope `identity-tokens`:

1. **Only when the user knows.** It gets an identity token only when the user does something that the app has presented as done with their identity, at the server named there. Never in the background, never when it fetches, prefetches or previews something, and never because content asked for it without the user's action. A feature that the user explicitly set up for one server counts as knowing, e.g. a feed they subscribed to as themselves (see [Personal feeds](#personal-feeds)).
2. **The audience from the URL.** `audience` is the origin of the URL the app is about to send the request to, taken from that URL and from nothing else, in particular not from a feed or a response.
3. **One token per request.** It gets a new token for every request, and sends it only in that request, as `Authorization: Bearer <token>`. Servers may reject a token they have seen before.
4. **Never the access token.** It sends its PUPS access token only to its PUPS server, as RFC-6 requires, and never to the server it acts on.
5. **Show the handle.** Wherever the app says that something is done as the user, it shows the handle, e.g. `@tim.example.com`. If the app is connected to several domains, the user chooses which one acts, and the app shows that one.

If a request with an identity token is redirected to another origin, the app doesn't send the token there.

## Checking a token

A server that accepts identity tokens checks each one in this order, and rejects it at the first failure with `401` and `WWW-Authenticate: Bearer error="invalid_token"`. The cheap checks come first, so that a bad token never makes the server fetch anything.

1. **Header.** The request carries `Authorization: Bearer <token>`. The JWT's header has `typ` `at+jwt` (or `application/at+jwt`), `alg` `RS256` and a `kid`. Any other `alg`, including `none`, is rejected.
2. **Audience.** `aud` is a single string and equals the server's own origin, which the server knows from its configuration, never from the request's `Host` header. A server that runs on several origins compares with the one the request was sent to.
3. **Lifetime.** `exp` is in the future and `iat` is not, each with at most 60 seconds of clock skew, and `exp` is at most 300 seconds after `iat`.
4. **Claims.** `iss`, `client_id` and `jti` are present, and `sub` is a canonical identity URL, `https://<host>/` as in [The token](#the-token).
5. **Issuer.** The server runs RFC-6 discovery for the host of `sub`: it fetches `https://<host>/.well-known/host-meta.json`, takes the first link with the PUPS relation, and fetches the configuration document. Its `issuer` must equal `iss` exactly. This step ties the token to the domain: a PUPS server can sign tokens for anyone, but only the issuer that the domain itself names counts for that domain.
6. **Signature.** The server fetches the JWK Set from the configuration document's `jwks_uri`, takes the key with the token's `kid`, which must be an RSA key of at least 2048 bits, and checks the signature.
7. **Replay.** The server may remember each `jti` until its `exp`, and reject a token it has seen before.

The token then proves that the owner of the host of `sub` acts, through the app named by `client_id`. The server takes the identity only from `sub`, never from the body of the request. What the domain may do is the server's own decision.

**Fetching.** The fetches in steps 5 and 6 follow the rules of RFC-6's [CIMD fetch](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md#cimd-fetch-ssrf): `https` only, a 5-second timeout and a 64 KB limit, at most 3 redirects to `https` URLs, and only to public addresses, checked after DNS resolution.

**Caching.** The server caches `host-meta.json`, the configuration document and the JWK Set according to their HTTP cache headers, for at most one hour, so that a changed key or a moved PUPS server takes effect within that time. If the JWK Set lacks a token's `kid`, the server fetches it again, at most once every five minutes per issuer. It caches failed discoveries for at least a minute, and limits how often it starts a discovery for new hosts, so that nobody can use it to flood a domain with requests.

Any JWT library that supports `RS256` and RFC 9068 does steps 1 to 4 and 6. Only step 5, finding the issuer from `sub`, needs code of its own.

**Errors.** A server that doesn't accept a domain, e.g. one that isn't a moderator, answers `403`. A missing or invalid token gets `401`.

## Feed Actions

This RFC adds one field to the buttons and selects of [Feed Actions (RFC-9)](https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md).

| Field | Type | Meaning |
| --- | --- | --- |
| `identity` | boolean | `true`: the action is performed as the user. The reader shows this with the control, including the user's handle, and sends an identity token with the request. Defaults to `false`. |

`identity` works like every other field: it can be set in a template and replaced in a declaration or a response. It applies to buttons and to selects with an `action`, the controls that send requests. Readers ignore it on other controls.

### What the reader shows

For a control with `"identity": true`, the reader shows, **before the user acts**:

- **The handle** of the domain that will act, e.g. `@tim.example.com`.
- **The host** that the action goes to, e.g. `polls.example.com`, unless it is the feed's own host. RFC-9 asks readers to show this anyway; for identity actions it is required.

The text is visible together with the control, or once for its group if several of the group's controls have `identity`. It is never only in a tooltip, the help text or a menu. How it looks is up to the reader:

```
What should we order on Friday?

  ( ) Pizza   ( ) Sushi   ( ) Tacos
  Sent as @tim.example.com to polls.example.com
```

**The reader's own words.** The reader doesn't know what an action does, and the publisher must not be able to describe it in the reader's voice. The notice therefore uses only generic verbs, such as *Sent as* or *Submitted as*, and never words from the feed or a response: not the control's `label`, not a display's `text`, not a `message`. The handle and the host are the only variable parts.

If the control has `confirm`, the reader adds the notice to the confirmation, separate from the publisher's question: "Block this user? They can't post or comment anymore." followed by *Sent as @tim.example.com to groups.example.com*. A `menu` button with `identity` shows the notice in its menu entry or in a confirmation before it sends anything.

### Performing the action

When the user triggers a control with `identity`, the reader, after the confirmation if there is one:

1. gets an identity token for the origin of the expanded action URL, as described in [Using identity tokens in an app](#using-identity-tokens-in-an-app);
2. sends the request as RFC-9 describes, with `Authorization: Bearer <token>`.

If it can't get a token, it says so, and doesn't send the request.

- **Not signed in.** A reader that has no grant with `identity-tokens` still shows the control, with a note such as *Sign in with your domain to use this*. Clicking it starts the sign-in of RFC-6, or the grant of the missing scope. Afterwards, the user clicks again: the reader never performs the action on its own after sign-in.
- **No domain.** A reader may explain that the action needs the user's own domain, and offer the item's link instead.
- **The user's own server.** If the action URL is under the `actions_base` of the user's own PUPS server, RFC-9's rules apply and the reader sends its access token instead. `identity` then only changes what the reader shows.

Readers still send no cookies. They send an identity token with an action only for controls with `identity`, and only when the user triggers them.

### On the server

The action endpoint checks the token as described in [Checking a token](#checking-a-token). A missing or invalid token gets `401`, with a `message`, e.g. "Sign in with your domain to vote. You can also vote on the website: https://polls.example.com/p/lunch". A domain that may not perform the action gets `403` with a `message`.

The response, and the next version of the feed, may name the user, e.g. a display "You voted for Pizza" or "Accepted by @tim.example.com". The server takes the name from the token's `sub`.

### Readers without support

RFC-9 requires readers that don't support this RFC to show **none** of an item's controls if one of them has `identity`, and to keep the item's `actions` fallback, as if they didn't support actions at all (see [RFC-9](https://github.com/bigwide-net/RFCs/blob/main/RFC-9-Feed-Actions.md#acting-as-the-user-on-other-servers)). Such a reader never offers an action that can only fail.

Publishers therefore add the ordinary `actions` fallback ([RFC-8](https://github.com/bigwide-net/RFCs/blob/main/RFC-8-Fallback-HTML.md)) to items with `identity` controls, with a link to a page where the user can do the same. A reader that supports this RFC removes it, as RFC-9 requires, also when the user isn't signed in.

### Examples

**A poll in a public feed.** Everyone gets the same feed. The server counts one vote per domain:

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
        "action": "https://polls.example.com/vote/{id}",
        "identity": true
      }
    ]
  }
}
```

Tim picks *Pizza*. His reader gets a token for `https://polls.example.com` from his PUPS server and sends the vote with it. The server checks the token, records the vote for `tim.example.com`, and answers with the results, as in RFC-9's poll example. If Tim votes again, the server answers with "You already voted for Pizza" and the results.

**Several moderators.** The Feed Readers group publishes its moderation queue at a capability URL that only its moderators know, as in RFC-9. The buttons have `identity`, so the server knows which moderator acts, and keeps the list of moderators' domains itself:

```json
"templates": {
  "accept": {
    "type": "button",
    "label": "Accept",
    "style": "major",
    "group": "accept",
    "action": "https://api.example.com/feed-readers/accept/{id}",
    "identity": true
  }
}
```

The reader shows *Sent as @tim.example.com to api.example.com* with the button. The server accepts the post, notes "Accepted by @tim.example.com", and answers `403` to domains that aren't moderators. If a moderator leaves, the group removes their domain from its list; the URL of the queue can stay the same, since its key no longer decides who may act.

## Personal feeds

A feed can be personal: the same URL shows each subscriber their own content. With identity tokens, a reader fetches such a feed as the user, and the server needs neither accounts nor a secret feed URL per subscriber.

This is most useful on public [Feed Submissions (RFC-4)](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) servers, such as an rssmention installation that hosts comments and discussion groups for many sites. Whoever comments there never registered, and doesn't need to: in RFC-4, their domain is their identity. Identity tokens let them read what belongs to that identity. Such a server can offer every author one personal feed, e.g. `https://comments.example.org/me.json`, with:

- replies to their comments and posts, across all sites and groups that the server hosts;
- the state of their submissions: pending, accepted or rejected, with the reason;
- the moderation queues of the groups they own or moderate, with the actions of RFC-9;
- the posts of private groups they belong to.

Public feeds can use it too, e.g. a poll that shows each subscriber their own vote after a refresh, where everyone else sees the plain poll.

### Subscribing as the user

A reader fetches a feed with identity tokens only if the user chose to subscribe to it as themselves:

- **The user decides.** When the user subscribes, the reader offers *Subscribe as @tim.example.com*. Afterwards it shows the handle with the feed, e.g. next to its title or in its settings, and lets the user change the subscription to an anonymous one at any time. Nothing in a feed or a response can turn a subscription into one with identity.
- **The 401 hint.** A feed that exists only in a personal version answers fetches without a token with `401` and `WWW-Authenticate: Bearer`. A reader that supports this RFC then offers to subscribe as the user, and subscribes only if the user agrees.
- **A new token for every fetch.** For each fetch of the feed, the reader gets a new identity token for the origin of the feed URL and sends it with that request. It sends it with nothing else: not with the items' links, images or attachments, and not with other feeds on the same origin, unless the user subscribed to them as themselves too. If the request is redirected to another origin, the token isn't sent there.
- **Actions stay separate.** Subscribing as the user doesn't make the feed's actions send identity tokens. Only controls with `identity` do that, as described in [Feed Actions](#feed-actions).

How a reader learns that a feed has an optional personal version, such as the poll above, is open (see [Open questions](#open-questions)). Until then, publishers say so in the feed's `description` or on their web page.

### On the server

- **Check every fetch.** The server checks the token as described in [Checking a token](#checking-a-token), with the feed's origin as audience.
- **No shared caching.** A personal feed is served with `Cache-Control: private` (or `no-store`) and `Vary: Authorization`, so that no cache between the server and the reader serves one user's feed to another.
- **Status codes.** `401` with `WWW-Authenticate: Bearer` for a feed that needs a token and got none, or got an invalid one. `403` for a domain that has no access to the feed, e.g. to a private group it doesn't belong to.
- **The same feed.** `feed_url` is the same for everyone. Item `id`s stay stable for each subscriber, so readers can keep track of what was read.
- **Polling.** Readers poll feeds, and each fetch costs the reader a token exchange. Servers should support conditional requests (`ETag`, `If-None-Match`), computed per subscriber, and readers should not poll personal feeds more often than other feeds.

## Security considerations

### Threat model

| Attacker | Wants to | Main defences |
| --- | --- | --- |
| A malicious server that receives a token | Use it on another server, or later | `aud` is one origin, lifetime at most five minutes, optional `jti` replay check |
| Someone who copies a token from a log or proxy | Act as the user | Same as above; the token is good for one request at one server |
| A malicious feed | Make the reader reveal the user's identity, or act as the user | Nothing without a click; the handle and the host are shown before the click; `aud` taken from the URL; only the user can subscribe as themselves |
| A cache between server and reader | Serve one user's personal feed to another | `Cache-Control: private` and `Vary: Authorization` on personal feeds |
| A malicious or compromised app with the scope | Act as the user anywhere | Step-up consent, rate limits, log on the apps page, revocation, `client_id` in every token |
| An app with only write scopes | Publish its own key or issuer | Configuration document, JWK Set and the PUPS link in `host-meta.json` are not writable through PUPS |
| Someone who wants to forge tokens | Sign in as someone else | Signature with the PUPS server's key; the issuer is found from the domain in `sub` |
| Someone who wants to misuse servers | Use servers to reach internal networks, or to flood a domain | Fetch rules, cheap checks first, caching of successes and failures, discovery rate limits |
| An attacker who gets the domain, or the PUPS server's key | Act as the owner | None in the protocol: domain control is identity. Key rotation limits a stolen key to the cache time |

### Tokens

- **Audience.** A token is valid at one origin. Servers compare `aud` with their configured origin, exactly. Several services on one origin share tokens; services that must not, use separate origins.
- **Lifetime.** At most five minutes, recommended 60 seconds. Revocation needs no list: when the grant is revoked, the app gets no new tokens.
- **Bearer.** Whoever holds a token can use it at its server until it expires. A server that keeps `jti`s makes it good for one request. Binding tokens to a request is an open question.
- **Algorithms.** `RS256` only. Servers never take the algorithm from anywhere but their own rules, never accept `none`, and never use a key from the token itself (`jwk`, `jku` or `x5u` headers are ignored).
- **No confusion with PUPS access tokens.** Identity tokens are never accepted by the PUPS upload API, and PUPS access tokens are never sent to other servers. PUPS access tokens are opaque, identity tokens are JWTs with another server as audience.

### The issuer

The check in step 5 is what makes a token mean "the owner of this domain acts". Any PUPS server can sign a token with any `sub`. Only the issuer that `https://<host>/.well-known/host-meta.json` points to is accepted for that host. Whoever controls `host-meta.json` and the PUPS link in it therefore controls the identity, as in RFC-6: the domain's owner, and the host or CMS that serves the file.

### Disclosure

Identity tokens tell a server who the user is. A feed that could make the reader send them at will could unmask every reader of the feed. Therefore readers send them with an action only when the user triggers a control that showed the handle and the host beforehand, and only to the origin of the action URL. With a feed fetch, they send them only for feeds that the user subscribed to as themselves, and only to the origin of the feed URL.

### One domain is not one person

Domains are cheap, and subdomains are free for whoever has one domain. A poll with one vote per domain is a poll with one vote per domain. Publishers who need more can combine identity tokens with capability URLs, or with their own list of domains that may vote.

## Privacy considerations

- **The server learns the handle and the app.** It learns the domain, which often is the user's name, and the `client_id` of the app the user uses. That is the purpose, and the reader shows it before the user acts.
- **The owner's server learns where the user acted.** The log stays on the owner's own PUPS server and is visible only to the owner.
- **Nothing else.** The token contains no other data about the user. Servers that want a name or an avatar read the user's public profile (SocialSpec, `host-meta.json`).
- **The fetches are visible.** A server that checks a token fetches `host-meta.json` and the PUPS documents from the user's domain, so the domain's logs show which servers checked tokens.
- **Personal feeds report reading.** Each fetch of a feed subscribed as the user tells its server that the user still follows it, and when their reader polls. Readers therefore show the handle with such subscriptions, and let the user make them anonymous.

## Relation to other specs

- **RFC-6 sign-in with your domain** proves the owner's identity to an app, interactively, with a redirect. Identity tokens let that app prove it further to any server, without one.
- **AT Protocol service auth** (Bluesky) is the closest model: the user's server signs short-lived JWTs for one other service, and the service finds the key through the user's identity. This RFC finds the server through `host-meta.json` instead of a DID document, and uses OAuth token exchange to get the token.
- **OpenID Connect ID tokens** are similar JWTs, but their audience is the client that signed the user in, and their issuer is configured in advance. Identity tokens are for a third server, which finds the issuer from the subject.
- **IndieAuth** proves a domain to an app with a redirect, like RFC-6.
- **HTTP Message Signatures** ([RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)), as used for actors in ActivityPub, sign each request with a key that is published for the actor. They bind the proof to the request, at the cost of canonicalizing requests on both sides. Here, the app needs no key and no cryptography.

## Conformance

**PUPS servers**

- The configuration document has `jwks_uri` and `identity-tokens` in `scopes_supported`; the JWK Set is served with CORS and a `max-age` of at most 3600; neither is writable through the upload API, `post_endpoint` or the host-meta endpoint.
- Consent for `identity-tokens` shows its own group and needs step-up confirmation; it is never skipped.
- Token exchange with a valid access token of a grant with the scope returns a token whose header and claims match [The token](#the-token), signed by a key in the JWK Set.
- Rejected: an access token without the scope (`invalid_scope`), a revoked or expired one (`invalid_grant`), a `client_id` other than the grant's, an audience with a path, `/`, query, user info, fragment, `http`, IP address, `localhost` or several values (`invalid_target`), a `resource` or `actor_token` (`invalid_request`).
- Issued tokens appear in the log on the apps page; after revocation, no more tokens are issued.

**Readers**

- A control with `identity` shows the handle, and the host if it differs from the feed's, before the user acts, also on `menu` buttons and in `confirm` dialogs. The notice is in the reader's own generic words and contains no text from the feed or a response.
- A token is requested only on a click, with the origin of the expanded action URL as `audience`, and sent only in that request.
- Without a grant, clicking starts sign-in, and the action is not sent afterwards without a new click.
- The `actions` fallback of an item with `identity` controls is removed.
- Feeds are fetched with a token only after the user subscribed as themselves; the handle is shown with the subscription; a `401` with `WWW-Authenticate: Bearer` leads to an offer, never to a token on its own; the token is sent with the feed's fetches only, not with its links, images, attachments or redirects to other origins.

**Servers**

- Tokens are rejected for: a wrong `aud`, an array `aud`, an expired token, a lifetime over 300 seconds, an `alg` other than `RS256`, a `sub` that isn't a canonical identity URL, an `iss` other than the one the domain's `host-meta.json` leads to, a `kid` not in the JWK Set, a bad signature.
- No fetch happens for a token that fails steps 1 to 4.
- Discovery fetches respect the fetch rules, including private addresses after DNS resolution.
- Personal feeds carry `Cache-Control: private` or `no-store` and `Vary: Authorization`.

## Open questions

- **Discovering personal feeds.** How does a reader learn that a public feed also has a personal version, so that it can offer to subscribe as the user? A property in the feed, a link attribute, a header in the feed's response?
- **Binding tokens to a request.** Should a token name the method and URL it is for, like AT Protocol's `lxm`, or be sender-constrained with [DPoP](https://www.rfc-editor.org/rfc/rfc9449)?
- **Optional identity.** Controls that the user may use either anonymously or as themselves, e.g. a poll that accepts both and counts them separately.
- **Owner policies.** Should owners be able to say "ask me every time" for an app, or allow it only certain audiences, and how would the app learn that?
- **RSS.** If Feed Actions are ported to RSS, `identity` goes with them.
- **Names.** The scope `identity-tokens` and the field `identity` are proposals.
