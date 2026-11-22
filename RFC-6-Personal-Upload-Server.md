# RFC: Personal Upload Server - Letting Apps Write Files to Your Own Domain

Nov 13, 2026 · @Tim Jansen

Latest version on GitHub: [RFC-6-Personal-Upload-Server.md](https://github.com/bigwide-net/RFCs/blob/main/RFC-6-Personal-Upload-Server.md) · [Change history](https://github.com/bigwide-net/RFCs/commits/main/RFC-6-Personal-Upload-Server.md)

## Summary

This proposal defines the **Personal Upload Server (PUPS)**: a small server, run by one person on their own domain, that lets apps upload files to that domain. An app only needs to know the domain. It discovers the server, asks the owner for permission with standard OAuth, and gets a token that can create, replace and delete files under the paths the owner approved. Everything an app uploads is public, served from the owner's domain, except in two small [private areas](#private-areas) that only the owner's apps can read: a `private/` folder in each app's own directory, and `/pups/private/` for private files that extensions define, such as an inbox.

It is the write side of feed-based personal publishing:

- A client of [Feed Submissions (RFC-4)](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) writes the author feed with the user's comments (RSS or JSON Feed) to the user's domain, then sends the webmention.
- A profile editor writes the [SocialSpec](https://socialspec.org) files (`/.well-known/following.opml`, `/.well-known/about.md`, `/.well-known/contact.vcf`).
- A feed-based social app publishes the user's posts as a feed on the user's domain.

The protocol is built from existing specs, with as little new as possible:

- **Discovery**: one link in `/.well-known/host-meta.json` ([RFC 6415](https://www.rfc-editor.org/rfc/rfc6415), JSON form) points to a JSON configuration document.
- **Authorization**: [OAuth 2.1](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/) authorization code flow with PKCE (S256), public clients only.
- **Client identity**: [OAuth Client ID Metadata Documents](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/) (CIMD): the `client_id` is a URL on the app's own domain, so apps need no registration.
- **Uploads**: plain HTTP `PUT` and `DELETE` with bearer tokens ([RFC 6750](https://www.rfc-editor.org/rfc/rfc6750)) and ETag preconditions ([RFC 9110](https://www.rfc-editor.org/rfc/rfc9110#name-conditional-requests)), plus a `POST` fallback for hosts that block those methods.
- **Sign-in with your domain**: because only the domain's owner can approve, every authorization also proves control of the domain. The token response names the owner's domain in `me`, as [IndieAuth](https://indieauth.spec.indieweb.org/) does.

The new parts are the configuration document, the scope syntax, the rules for paths and file types, and an endpoint that lets apps register links in `host-meta.json`. See [What is new and what is reused](#what-is-new-and-what-is-reused).

**Why now.** [RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) lets anyone comment without registering anywhere, by publishing the comment in a feed on their own domain. [SocialSpec](https://socialspec.org) builds profiles, contact details and follow lists from files on the user's domain. Both assume that the user can put files on their domain, and both leave open how an app does that. Today the answer is "upload them yourself", with FTP, a CMS plugin or a static site generator. No app can write to a user's domain without app-specific setup, so every app ends up hosting the user's data itself, which is exactly the centralization these specs try to avoid. A protocol that any app can use with nothing but the user's domain name closes that gap. It has to be secure enough to give third-party apps write access to the domain that is the user's identity, and simple enough to install on a $2 shared host.

## Goals and non-goals

**Goals**

- An app needs nothing but the user's domain: no pre-registration, no API key, no app-specific setup by the owner.
- Least privilege: an app writes only its own directory, plus the specific well-known files the owner approved one by one.
- Safe on ordinary web hosting: uploaded files can never execute on the server or run script with the owner's origin in a browser.
- Works from browser apps (CORS), native apps and browser apps in local development (loopback redirects), and server-side apps alike.
- Installable on cheap shared hosting with nothing but PHP on Apache or a compatible web server (no database, no build step), and with one click on Cloudflare.

**Non-goals**

- **Website content.** Apps never write HTML, JavaScript or CSS. PUPS stores data and media, not web pages.
- **Shared or access-controlled storage.** Files are public, or, in the [private areas](#private-areas), readable only by apps the owner approved. There is no sharing with other people, no access lists and no encryption: private files are protected from the public, not from the host.
- **Multi-user hosting.** One installation has exactly one owner, and the owner is the domain.
- **Directory listings, search, quotas per app, synchronization.** Clients keep track of their own files.
- **Feed formats, federation and notifications.** Those belong to other specs (RFC-4, SocialSpec, …).

## What is new and what is reused

PUPS invents as little as possible. Most of it is existing standards, some of them with stricter rules, and a few pieces that no existing spec covers. For readers nothing is new: they fetch public files over plain HTTP.

**Used as they are.** These specs are used without changes, and existing libraries for them work.

| Spec | Used for |
| --- | --- |
| [OAuth 2.1](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/), authorization code flow with [PKCE](https://www.rfc-editor.org/rfc/rfc7636) | Asking the owner for permission. See [Authorization](#authorization). |
| [RFC 6750](https://www.rfc-editor.org/rfc/rfc6750) bearer tokens, [RFC 7009](https://www.rfc-editor.org/rfc/rfc7009) revocation | Authenticating uploads, disconnecting apps. |
| [RFC 9207](https://www.rfc-editor.org/rfc/rfc9207) `iss` parameter | Protecting apps that talk to many servers against mix-up attacks. |
| [RFC 8414](https://www.rfc-editor.org/rfc/rfc8414) server metadata | Generic OAuth tooling. PUPS clients don't need it. |
| [RFC 6415](https://www.rfc-editor.org/rfc/rfc6415) `host-meta.json`, with [RFC 7033](https://www.rfc-editor.org/rfc/rfc7033) links, titles and order | Discovery, and the domain's index of feeds, pages and site properties. |
| HTTP `PUT`, `DELETE` and conditional requests ([RFC 9110](https://www.rfc-editor.org/rfc/rfc9110#name-conditional-requests)) | Uploads, and safe concurrent changes with ETags. |
| HTTP caching ([RFC 9111](https://www.rfc-editor.org/rfc/rfc9111)) | Fresh reads: shared caches don't store responses to requests with `Authorization`. |
| [CORS](https://fetch.spec.whatwg.org/#http-cors-protocol), `Content-Security-Policy: sandbox`, `X-Content-Type-Options: nosniff` | Browser apps, and serving uploads safely. |
| `me` from [IndieAuth](https://indieauth.spec.indieweb.org/) | Sign-in with the domain. |
| Passkeys ([WebAuthn](https://www.w3.org/TR/webauthn-3/)), `__Host-` cookies | The owner login. |

**Used with stricter rules.** These specs leave choices open. PUPS makes them, mostly for security, and an implementation must follow them even where the base spec would allow more.

| Spec | PUPS's rules |
| --- | --- |
| OAuth 2.1 | Public clients only, PKCE with S256 only, codes valid for 60 seconds, refresh tokens rotated on every use with a short [grace period](#refresh) for lost responses. |
| [Client ID Metadata Documents](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/) | The client host is the app's identity, and names its directory. `client_uri` and `logo_uri` must be on that host. See [Client identity](#client-identity). |
| [RFC 8252](https://www.rfc-editor.org/rfc/rfc8252) loopback redirects | Accepted for every app, `localhost` is not. |

**New in this RFC.** No existing spec covers these:

- **The configuration document** and the link relation that points to it from `host-meta.json`: where to write (`storage_base`), where files are read (`public_base`), the well-known paths and the limits. See [Discovery](#discovery).
- **Path scopes**: `write:<path>` and `identify`, and the rule that an app writes only its own directory, `/appdata/<client-host>/`, plus well-known paths the owner approved one by one. See [Scopes](#scopes).
- **The list of well-known paths**: the main feed, the SocialSpec files, the site's images and pages, and the site properties and link relations that belong to them. See [Well-known paths](#well-known-paths).
- **Rules for paths, file types and serving**: an allowlist of data and media types, with the headers every upload is served with, so that uploads can't run on the server or in the browser. See [Paths](#paths), [File types](#file-types) and [Serving files](#serving-files).
- **Fresh reads**: a bearer token on a `GET` bypasses caches, but grants nothing. See [Fresh reads](#fresh-reads).
- **Private areas**: `/appdata/<client-host>/private/` and `/pups/private/`, read with a token whose scope covers them, and the `read:` scope for private paths. See [Private areas](#private-areas).
- **The POST fallback** for hosts that block `PUT` and `DELETE`. See [POST fallback](#post-fallback).
- **The host-meta endpoint**, through which apps add links to and set properties in `host-meta.json` without being able to touch the owner's entries. See [Editing host-meta.json](#editing-host-metajson).
- **Requirements for the owner's side**: what the consent screen shows, step-up confirmation for well-known paths, and the two deployment modes. See [Consent](#consent), [Owner authentication](#owner-authentication) and [Deployment modes](#deployment-modes).

A developer who knows OAuth and HTTP therefore needs to learn only the last list. An app can use any OAuth 2.1 client library that supports PKCE, and needs its own code only for discovery, scopes and the upload rules.

## Roles

- **Owner**: the person who controls the domain, e.g. `tim.example.com`, and runs the server. The domain is the owner's identity.
- **Server** (PUPS): the authorization server and upload API, with its owner pages (login, consent, app management).
- **App** (client): software that wants to write files to the owner's domain: a browser app, a native app or a server-side service. It is identified by its `client_id` URL.
- **Reader**: anyone who fetches the published files. Readers need no token, and nothing in this proposal is new to them.

## Overview

1. **Discover.** The user types their domain into the app. The app fetches `https://tim.example.com/.well-known/host-meta.json`, follows the PUPS link to the configuration document and learns the endpoints and the allowed paths.
2. **Authorize.** The app redirects the browser to the authorization endpoint with its `client_id`, PKCE challenge and the scopes it needs.
3. **Identify the app.** The server fetches the app's client metadata document from the `client_id` URL and checks the redirect URI.
4. **Consent.** The owner logs in and sees the app's real domain and the paths it wants to write; they approve, narrow or deny.
5. **Exchange.** The app receives a code, checks the issuer, and redeems the code for an access token, a refresh token and the owner's identity `me`.
6. **Upload.** The app `PUT`s files with its bearer token, using ETags to avoid overwriting concurrent changes.
7. **Read.** Anyone reads the published files at their public URL, without a token. An app that needs the latest version, e.g. to change a file, reads it at `storage_base` with its token, which bypasses caches.

![Sequence diagram: the app discovers the PUPS server through host-meta.json and the configuration document, redirects the owner's browser to the authorization endpoint, the server fetches the app's client metadata and shows the consent screen, the browser returns to the app with a code, and the app redeems it for tokens and uploads files.](diagrams/RFC-6-authorization-flow.svg)

The diagram numbers its steps on its own: steps 1–3 are discovery, 4–9 authorization with the app's identification and the owner's consent, 10 the code exchange and 11 an upload. The owner's domain and the PUPS server are often the same host; they differ when PUPS runs on a subdomain such as `storage.tim.example.com` (see [Serving discovery documents](#serving-discovery-documents)).

## Discovery

### Starting point

The app starts from what the user typed: `tim.example.com`, `https://tim.example.com` or `https://tim.example.com/some/page`. It normalizes it to `https://<host>/`: HTTPS, the host in lowercase (IDNs in their ASCII form), no port, path, query or fragment. Discovery always uses HTTPS. Clients follow at most 3 redirects, only to HTTPS URLs.

### `host-meta.json`

The app fetches `https://tim.example.com/.well-known/host-meta.json` ([RFC 6415](https://www.rfc-editor.org/rfc/rfc6415), JSON form) and takes the first link whose `rel` is `https://bigwide.net/ns/personal-storage`:

```json
{
  "links": [
    {
      "rel": "https://bigwide.net/ns/personal-storage",
      "type": "application/json",
      "href": "https://tim.example.com/personal-storage.json"
    }
  ]
}
```

- Only the JSON form is used. Apps never request or parse the XML `host-meta`, and servers need not serve it.
- The link relation is a URI, so it needs no IANA registration. **`https://bigwide.net/ns/personal-storage` is a placeholder** until the relation has a permanent home.
- `host-meta.json` must be served on the owner's domain itself. The configuration document and all endpoints may live on another host, e.g. `storage.tim.example.com` or `id.tim.example.com`.
- Other links in the file are not PUPS's business; apps ignore them. PUPS itself may add links and properties on behalf of apps (see [Editing host-meta.json](#editing-host-metajson)).

### Configuration document

The `href` points to the configuration document. Its path is chosen by the server; `/personal-storage.json` is only an example.

```json
{
  "issuer": "https://tim.example.com",
  "authorization_endpoint": "https://tim.example.com/oauth/authorize",
  "token_endpoint": "https://tim.example.com/oauth/token",
  "revocation_endpoint": "https://tim.example.com/oauth/revoke",
  "storage_base": "https://tim.example.com/files/",
  "public_base": "https://tim.example.com/files/",
  "scopes_supported": ["identify", "write:/appdata/"],
  "well_known_paths": [
    { "path": "/.well-known/following.opml", "url": "https://tim.example.com/.well-known/following.opml" },
    { "path": "/.well-known/about.md", "url": "https://tim.example.com/.well-known/about.md",
      "properties": ["https://bigwide.net/ns/name", "https://bigwide.net/ns/title", "https://bigwide.net/ns/summary", "https://bigwide.net/ns/theme-color", "https://bigwide.net/ns/language"] },
    { "path": "/.well-known/contact.vcf", "url": "https://tim.example.com/.well-known/contact.vcf" },
    { "path": "/feed.json", "url": "https://tim.example.com/feed.json", "rels": ["https://bigwide.net/ns/feed"] },
    { "path": "/feed/", "url": "https://tim.example.com/feed/", "rels": ["https://bigwide.net/ns/feed"] },
    { "path": "/pups/images/", "url": "https://tim.example.com/pups/images/", "properties": ["https://bigwide.net/ns/avatar", "https://bigwide.net/ns/banner"] },
    { "path": "/pups/pages/", "url": "https://tim.example.com/pups/pages/", "rels": ["https://bigwide.net/ns/webpup-pages"],
      "properties": ["https://bigwide.net/ns/webpup-footer"], "link_properties": ["https://bigwide.net/ns/webpup-hidden"] }
  ],
  "host_meta_endpoint": "https://tim.example.com/pups/host-meta",
  "max_upload_bytes": 26214400,
  "spec_version": "0.1"
}
```

| Field | Meaning |
| --- | --- |
| `issuer` | **Required.** The OAuth issuer identifier: an HTTPS URL without query or fragment. Returned as `iss` in the authorization response. |
| `authorization_endpoint`, `token_endpoint`, `revocation_endpoint` | **Required.** The OAuth endpoints. |
| `storage_base` | **Required.** The base URL for writes. A file with path `/p` is written at `storage_base` + `p`. Ends with `/`. |
| `public_base` | **Required.** The base URL where the same files are read publicly. It may differ from `storage_base`, e.g. a CDN. Ends with `/`. See [Public URLs](#public-urls). |
| `scopes_supported` | **Required.** `identify` and the scope prefix `write:/appdata/`. Well-known scopes follow from `well_known_paths`. |
| `well_known_paths` | The files and directories outside `/appdata/` that apps may ask for. See [Well-known paths](#well-known-paths). Missing: none. |
| `host_meta_endpoint` | Where apps add links to and set properties in `host-meta.json`. Missing: the server cannot write `host-meta.json`. |
| `post_endpoint` | Present only if the server cannot accept `PUT` and `DELETE`. Clients must then send all writes there. See [POST fallback](#post-fallback). |
| `mention_endpoint` | Present if the server has a built-in receiver for [mentions (RFC-11)](https://github.com/bigwide-net/RFCs/blob/main/RFC-11-Mentions.md) that writes the owner's inbox. See [Built-in receiver](#built-in-receiver). |
| `comment_endpoint` | Present if the server accepts [comments (RFC-4)](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) on posts of the owner's feeds and writes them into the owner's inbox. See [Built-in receiver](#built-in-receiver). |
| `max_upload_bytes` | **Required.** The largest file the server accepts, in bytes. |
| `spec_version` | **Required.** The version of this proposal the server implements: `0.1`. |

Clients ignore fields they don't know. The configuration document is authoritative: if it disagrees with the RFC 8414 metadata below, the configuration document wins.

### OAuth metadata

The server also serves [RFC 8414](https://www.rfc-editor.org/rfc/rfc8414) metadata at `<issuer>/.well-known/oauth-authorization-server`, for generic OAuth tooling. It includes `code_challenge_methods_supported: ["S256"]`, `authorization_response_iss_parameter_supported: true`, `token_endpoint_auth_methods_supported: ["none"]` and `client_id_metadata_document_supported: true`. PUPS clients don't need it.

### Serving discovery documents

Both documents are served as `application/json` with `Access-Control-Allow-Origin: *`, so browser apps can read them.

`host-meta.json` must always be served on the owner's domain, but PUPS doesn't have to be the software that serves it. If PUPS runs on the same web server as the domain, it serves the file itself. If PUPS runs on another host (e.g. `storage.tim.example.com`) while the website at `tim.example.com` is served by something else (a static host, a CMS), PUPS can't write there: the owner must place `host-meta.json` on the website's server and configure it to send the CORS header. The server's setup page checks that the file is reachable and links to the configuration document.

## Client identity

Apps are identified by **Client ID Metadata Documents** ([draft-ietf-oauth-client-id-metadata-document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/)). There is no registration step.

- The `client_id` is an HTTPS URL with a path, on the app's own domain: `https://app.example/oauth/client.json`. It has no port, user info, query or fragment, and its host is lowercase.
- The URL returns a JSON document with [RFC 7591](https://www.rfc-editor.org/rfc/rfc7591) fields:

```json
{
  "client_id": "https://app.example/oauth/client.json",
  "client_name": "Example Reader",
  "client_uri": "https://app.example/",
  "logo_uri": "https://app.example/logo.png",
  "redirect_uris": ["https://app.example/callback"]
}
```

- `client_id` in the document must equal the URL it was fetched from, exactly.
- The app's identity is the **client host**, the host of `client_id`: it is the only thing about the app the server has verified, by fetching the document from there. The consent screen shows it, never a host taken from the document.
- `client_uri` and `logo_uri`, if present, must be on the client host itself, optionally with or without a leading `www.`: with `client_id` on `app.example.com`, both `https://app.example.com/` and `https://www.app.example.com/` are fine, and with `client_id` on `www.app.example.com`, so is `https://app.example.com/`. Any other host, including a parent domain or another subdomain, is rejected. A parent domain may belong to someone else: a client on `alice.github.io` could otherwise name `https://github.io/` and show GitHub's logo, and nobody would notice the subdomain. An app that wants its main domain on the consent screen hosts its client metadata document there.
- Every app publishes its own document on its own domain. A `client_id` hosted on someone else's domain is not that app's identity, and the consent screen would show the wrong domain.
- `redirect_uris` are compared to the request by exact string match. Any app may list loopback redirects (`http://127.0.0.1/callback` or `http://[::1]/callback`); the port is ignored when comparing, per [RFC 8252](https://www.rfc-editor.org/rfc/rfc8252#section-7.3). `localhost` is not accepted. Loopback redirects serve native apps, which listen on a local port, and the development of browser apps, which run on a local web server. The server can't tell the two apart and doesn't try: in both cases the code is delivered to the owner's own computer, and PKCE makes sure that only the app that started the flow can redeem it.
- The server fetches the document during authorization and may cache it, following HTTP cache headers, for at most 24 hours. The fetch is security-sensitive; see [CIMD fetch](#cimd-fetch-ssrf).
- The **client host** is the host of `client_id`. It names the app's own directory, `/appdata/<client-host>/`. Apps on the same host share it: they are the same party.

## Authorization

### Request

The app creates a random `code_verifier` (43–128 characters), `code_challenge = BASE64URL(SHA256(code_verifier))` and a random `state`, then sends the browser to:

```
GET /oauth/authorize?response_type=code
  &client_id=https://app.example/oauth/client.json
  &redirect_uri=https://app.example/callback
  &scope=identify%20write:/appdata/app.example/%20write:/feed.json
  &state=…
  &code_challenge=…
  &code_challenge_method=S256
```

The server checks:

1. `response_type=code`, a `code_challenge`, and `code_challenge_method=S256`. `plain` and requests without PKCE are rejected.
2. The client metadata document can be fetched and is valid, and `redirect_uri` is listed in it.
3. Every requested scope is valid for this client and this server (see [Scopes](#scopes)).

If the client or the redirect URI is invalid, the server shows an error page and **never redirects**, so the endpoint can't be used as an open redirector. Other errors are returned to the redirect URI as OAuth errors (`invalid_request`, `invalid_scope`, …).

### Consent

The owner authenticates (see [Owner authentication](#owner-authentication)) and sees the consent screen:

- **The app's real domain in large type**, the client host, with the logo, and the question whether the owner meant to log in with the app, named by its `client_name` (or the client host if it has none): "Did you mean to log in with “Downread”?". The name and logo are self-declared; the question lets the owner notice an app they didn't open, without suggesting that the app is suspicious.
- A **"new app"** notice if this `client_id` has never been approved before.
- A note that **the app runs on this computer** if the redirect URI is a loopback address.
- The scopes in **at most five groups**, introduced with the client host ("app.example may then"), in this order:
  1. **Identity**: "learn that you are tim.example.com". It is always shown and can't be unchecked, because every token response carries `me`. With `identify` alone the whole screen reads "Sign in to app.example as tim.example.com".
  2. **App data**: "store its own data", for `write:/appdata/<client-host>/`.
  3. **Feeds**: "manage your feeds", for `/feed.json`, `/feed/` and paths inside it, and `/.well-known/following.opml`.
  4. **Profile**: "manage your profile", for all other well-known paths outside `/pups/private/`.
  5. **Private data**: "read your private data" for `read:` scopes, or "manage your private data" if the request contains a `write:` scope, for paths in `/pups/private/`.

  A group appears only if the request contains one of its scopes (identity always does). Feeds, profile and private data need step-up confirmation. The screen does not mark them with technical warnings such as "outside the app's own directory": the group names say what the app may do.
- A checkbox per group, which grants or drops all of the group's scopes together, and Approve / Deny.
- The details of each group in an **expandable section, collapsed by default**: each scope in plain words, e.g. "its own folder (`/appdata/app.example/`)", "your feed (`/feed.json`)" or "your profile (`/.well-known/about.md`)", and the link relations and properties the scopes allow in `host-meta.json`.

The approve form carries a CSRF token, and the page is served with `X-Frame-Options: DENY` and `Content-Security-Policy: frame-ancestors 'none'`. It is never CORS-enabled.

A server may skip the consent screen for a request that an existing grant already covers and that contains no well-known scopes. It still requires owner authentication.

### Response

On approval the server redirects to:

```
https://app.example/callback?code=…&state=…&iss=https%3A%2F%2Ftim.example.com
```

`iss` ([RFC 9207](https://www.rfc-editor.org/rfc/rfc9207)) is the issuer. The app must check that `state` matches and that `iss` equals the `issuer` of the configuration document it discovered, **before** redeeming the code. This prevents mix-up attacks, where an app that talks to several servers is tricked into sending a code to the wrong one.

On denial the server redirects with `error=access_denied`. If the owner drops every scope, that counts as denial.

## Scopes

Scopes are plain OAuth scope strings, separated by spaces. [RFC 9396](https://www.rfc-editor.org/rfc/rfc9396) `authorization_details` is not used.

| Scope | Grants | Step-up |
| --- | --- | --- |
| `identify` | Nothing to write. The token response carries `me`. | No |
| `write:/appdata/<client-host>/` | Create, replace and delete files in the app's own directory. | No |
| `write:<well-known path>` | Create, replace and delete the declared well-known file, or files in the declared well-known directory. In `/pups/private/`, also read them. | Yes |
| `read:<private path>` | Read the declared private file in `/pups/private/`, or files in the declared private directory. | Yes |

**Syntax.** `write:<path-prefix>`. The prefix starts with `/` and uses only unreserved URL characters, `/` and percent-encoding. A prefix ending in `/` covers the whole subtree; any other prefix covers exactly one file. `write` always includes create, replace and delete, and, for [private](#private-areas) files, reading. Public files need no read scope. The only read scope is `read:<path-prefix>`, for a private path in `/pups/private/`: it lets the app read the files there, without writing them.

**Only two kinds of write scope exist**:

1. The app's own directory, `write:/appdata/<client-host>/`, where `<client-host>` is the host of the requesting `client_id`. A request for another app's directory, for `/appdata/` itself or for anything deeper than the app's directory root is rejected with `invalid_scope`.
2. A well-known path declared in `well_known_paths`: exactly the declared `path`, or, if the declared path is a directory, a directory or file inside it. `write:/feed/2026/` is valid if `/feed/` is declared.

Everything else is rejected, including `write:/`. Clients request only scopes that the configuration document allows; the server answers anything else with `invalid_scope`.

**The granted scope is enforced, not the requested one.** It is returned in the token response, and may be smaller than the request when the owner dropped scopes. Apps must handle that.

## File layout

The server offers no directory listing. A client that needs to know what it has stored keeps its own index, e.g. in a file in its own directory.

### App data

By default, an app stores everything in `/appdata/<client-host>/`, e.g. `/appdata/app.example/`. How it organizes that directory is up to the app. The consent screen calls it "the app's own data", and granting it is a single click.

The app's directory is **public**, with one exception: its subdirectory `private/`, e.g. `/appdata/app.example/private/`, which only the app can read (see [Private areas](#private-areas)). All other files are public like any other, and URLs are easy to guess. Apps must not store anything there that the user would not publish.

### Well-known paths

Some files have a location fixed by another spec: a SocialSpec profile must be at `/.well-known/about.md`, and the owner's main feed is at `/feed.json`. These are **well-known paths**. The server declares the ones it offers in `well_known_paths`, and an app may write one only after the owner granted its scope, with step-up confirmation.

Each entry is an object:

| Field | Meaning |
| --- | --- |
| `path` | **Required.** The scope path. Ends with `/` for a directory. |
| `url` | The public URL of the file or directory. Default: `public_base` + `path` without its leading `/`. Needed for files that must live outside `public_base`, e.g. at the domain root. |
| `rels` | The link relations apps may register in `host-meta.json` for files under this path. Default: none. |
| `properties` | The top-level `host-meta.json` properties apps with this scope may set. Default: none. |
| `link_properties` | The properties apps with this scope may set on the links they register for files under this path (see [Links](#links)). Default: none. |

**Server-side rules from the list.** For each entry, the list of well-known paths (see below) also says which of its `rels` are **single-valued**, so that an existing link is never replaced (see [Links](#links)), and which of its `properties` take a **URL** as their value (see [Properties](#properties)). Only the server applies these rules, so they are not part of the configuration document: the server takes them from the list. An implementation may let the owner configure them along with the declared paths.

Rules:

- A server declares only paths it can actually serve with the required headers (see [Serving files](#serving-files)), and only paths from the [list of well-known paths](#the-list-of-well-known-paths) below, which will be maintained on bigwide.net. The owner may remove entries.
- **Under `/.well-known/`, only single files may be declared**, never directories, and only files from that list. Never `host-meta`, `host-meta.json`, `webfinger`, `oauth-authorization-server`, `openid-configuration`, `security.txt`, `change-password`, or anything under `acme-challenge/` (an app that could write there could obtain TLS certificates for the domain).
- The file-type allowlist applies to well-known paths just as to app data.
- Apps request each well-known scope explicitly, create nothing next to a well-known file, and should write it with `If-Match`, because the owner may also edit it by hand.
- **Well-known files are shared.** An app's own directory has exactly one writer, the app, which may change its files as it likes. A well-known path, in contrast, can be granted to any number of apps, and they all write the same files, together with the owner. Every app keeps everything in such a file that it doesn't mean to change, including fields it doesn't know. Which parts of a file an app may change is decided by the spec that defines the file or its use, e.g. [RFC-7](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md) for the main feed. Where no spec says so, an app changes only what it created itself.

### The list of well-known paths

A server may declare the following paths. All of them live at the domain root, so the `url` of each is `https://<owner host>` followed by the path. A server that can't write and serve the domain root doesn't declare them (except `/pups/images/` and `/pups/pages/`, see below), and neither does a server whose website already serves one of the paths itself. WordPress, for example, serves its RSS feed at `/feed/`.

| Path | Contents | Defined by | Consent screen | Link relations | Properties |
| --- | --- | --- | --- | --- | --- |
| `/.well-known/following.opml` | The people the owner follows | [FollowSpec](https://followspec.org) | the list of people you follow | — | — |
| `/.well-known/about.md` | The owner's profile | [AboutSpec](https://aboutspec.org) | your profile | — | `https://bigwide.net/ns/name`, `https://bigwide.net/ns/title`, `https://bigwide.net/ns/summary`, `https://bigwide.net/ns/theme-color`, `https://bigwide.net/ns/language` |
| `/.well-known/contact.vcf` | The owner's contact details | [ContactSpec](https://contactspec.org) | your contact details | — | — |
| `/feed.json` | The owner's main feed | this RFC, [RFC-7](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md) | your feed | `https://bigwide.net/ns/feed` | — |
| `/feed/` | The files of the main feed, and the owner's other feeds | this RFC, RFC-7 | the files of your feed, and your other feeds | `https://bigwide.net/ns/feed` | — |
| `/pups/images/` | The site's profile images: avatar and banner | this RFC | your site's profile images | — | `https://bigwide.net/ns/avatar` (URL), `https://bigwide.net/ns/banner` (URL) |
| `/pups/pages/` | The site's pages and its footer, as Markdown | this RFC | your site's pages | `https://bigwide.net/ns/webpup-pages`, with the link property `https://bigwide.net/ns/webpup-hidden` | `https://bigwide.net/ns/webpup-footer` (URL) |
| `/pups/private/inbox.json` | The owner's inbox: mentions, replies and comments awaiting approval | [RFC-12](https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md) | your inbox | — | — |

Paths inside `/pups/private/` are added to this list by the specs that define them, such as `/pups/private/inbox.json`, the inbox feed of [Inbox Feeds (RFC-12)](https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md). They have no `url`, link relations or properties, see [Private areas](#private-areas).

The link relation `https://bigwide.net/ns/feed` registers a feed in `host-meta.json` (see [RFC-7](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md#registering-feeds-in-host-metajson)). It is not single-valued: a domain can have several feeds, and neither is `https://bigwide.net/ns/webpup-pages` (see [Pages](#pages)). Only `avatar`, `banner` and `webpup-footer` take a URL (see [Site properties](#site-properties)). The details on the consent screen describe a scope for a listed path with the text from the table, e.g. "your feed (`/feed.json`)". A scope inside `/feed/` is described by its path, e.g. "`/feed/photos.json`". `/feed.json`, `/feed/` and `/.well-known/following.opml` are in the consent screen's group "manage your feeds", the others in "manage your profile" (see [Consent](#consent)).

**`/feed.json`** is the owner's main feed: a [JSON Feed 1.1](https://www.jsonfeed.org/version/1.1/) document, with the extensions of the bigwide.net RFCs where they are used, e.g. [RFC-1](https://github.com/bigwide-net/RFCs/blob/main/RFC-1-Source-for-JSON-Feeds.md) sources and [RFC-3](https://github.com/bigwide-net/RFCs/blob/main/RFC-3-Podcasting-for-JSON-Feeds.md) podcasting. Readers and apps find it on the domain without any discovery, and it can serve as the author feed of [RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) comments. Its `feed_url` is `https://<owner host>/feed.json`. Like every `.json` file, it is served as `application/json`.

**`/feed/`** holds the files that belong to the main feed: images and other media that its items reference, such as RFC-3 podcast episodes, and further JSON Feed documents that it points to, such as older pages linked with `next_url`. Its media go into `/feed/images/` and `/feed/media/`, its icons into `/feed/images/icons/`, within the file-type allowlist. Apps store nothing in `/feed/` that a feed doesn't use, and follow the layout of RFC-7.

Several apps may write the main feed. [Feed Publishing with PUPS (RFC-7)](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md) defines the conventions they follow: how the feed is set up, which fields belong to whom, how items are posted, edited and deleted, and how older items move into archives in `/feed/`. All writes use `If-Match` (see [Concurrency](#concurrency)).

`/feed/` also holds the owner's other feeds, each with a name: the feed `photos` is `/feed/photos.json`, and its files are in `/feed/photos/`. An app that writes only such a feed asks for its scopes, `write:/feed/photos.json write:/feed/photos/`. The main feed's scope `write:/feed/` covers all of them. RFC-7 defines the layout and the conventions, which are the same as for the main feed.

**`/pups/images/`** holds the images that describe the site as a whole: the avatar and the banner that a profile or a web page of the domain shows, e.g. `https://tim.example.com/pups/images/avatar-k3x9.png`. Apps store only images there (`.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.avif`, `.svg`). Like feed icons (RFC-7), every new image gets a new file name, and the app deletes the old file after it has set the property to the new URL. `/pups/` itself belongs to the server, which may serve its own pages and endpoints there (e.g. the `host_meta_endpoint`); only the subdirectories `/pups/images/` and `/pups/pages/`, and paths inside `/pups/private/` (see [Private areas](#private-areas)), may be declared. A server that can't serve the domain root may still declare them without a `url`: the files are then at `public_base` + `pups/images/` and `pups/pages/`, which is fine for links and for properties that take a URL.

**`/pups/pages/`** holds the site's pages: Markdown files (`.md`) that a web page of the domain shows next to its feeds, such as "Now", "Uses" or an imprint, e.g. `https://tim.example.com/pups/pages/now.md`, and the images they show, in `/pups/pages/images/`. Apps store nothing else there. A page becomes part of the site when it is linked in `host-meta.json` (see [Pages](#pages)); the footer is a page too, linked by the property `https://bigwide.net/ns/webpup-footer`.

### Site properties

The top-level properties of `host-meta.json` that describe the site, for profiles, feed readers and web pages that render the domain (e.g. a single-page site that shows the owner's feeds). All are optional; a renderer falls back to its own defaults, never to guesses. Apps set them through the [host-meta endpoint](#editing-host-metajson) with the scope of the listed path; `post-url-template` has no scope, so apps can't set it.

| Property | Value | Scope |
| --- | --- | --- |
| `https://bigwide.net/ns/name` | The owner's name, e.g. `Tim Jansen` | `/.well-known/about.md` |
| `https://bigwide.net/ns/title` | The site's title, if it isn't the owner's name, e.g. `Tim's Notes` | `/.well-known/about.md` |
| `https://bigwide.net/ns/summary` | One line about the site or its owner | `/.well-known/about.md` |
| `https://bigwide.net/ns/theme-color` | The site's accent color, a CSS color, e.g. `#b0412e` | `/.well-known/about.md` |
| `https://bigwide.net/ns/language` | The site's main language, a BCP 47 tag, e.g. `en` | `/.well-known/about.md` |
| `https://bigwide.net/ns/avatar` | URL of a square image (at least 400 × 400 pixels) in `/pups/images/` | `/pups/images/` |
| `https://bigwide.net/ns/banner` | URL of a wide image (3:1, at least 1500 × 500 pixels) in `/pups/images/` | `/pups/images/` |
| `https://bigwide.net/ns/webpup-footer` | URL of a Markdown file in `/pups/pages/` that is shown at the bottom of every page of the site, e.g. a copyright line and a link to the imprint | `/pups/pages/` |
| `https://bigwide.net/ns/post-url-template` | The URL of a post's page on the site, as a template, e.g. `https://tim.example.com/p/{feed}/post/{index}`. See [Post URLs](#post-urls) | none: set by the owner, or by the server that installs the site's renderer |

`avatar`, `banner` and `webpup-footer` take a URL: the public URL of a file inside the app's granted scope, i.e. in `/pups/images/` or `/pups/pages/` (see [Properties](#properties)). A profile's text, its contact details and the people it follows stay in the SocialSpec files; these properties only add what those files don't have.

### Post URLs

The site property `https://bigwide.net/ns/post-url-template` says where the site's renderer shows a single post of a feed ([RFC-7](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md)). Apps that write the feeds use it to give new posts their `url` (RFC-7, [Post URLs](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md#post-urls)), so that readers link to the post's page on the owner's site.

```json
"https://bigwide.net/ns/post-url-template": "https://tim.example.com/p/{feed}/post/{index}"
```

- The value is an absolute `https` URL with variables in braces:
  - `{feed}`: the path of the feed file from the domain root, without `.json`: `feed` for `/feed.json`, `feed/photos` for `/feed/photos.json`;
  - `{index}`: the item's `_archive.index` ([RFC-13](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md));
  - `{id}`: the item's `id`, percent-encoded like a URI path segment.
- It contains `{index}` or `{id}`, or both. The other characters are used as they are.
- It describes the site, not the upload server, so it is in `host-meta.json` and not in the [configuration document](#configuration-document): the renderer may be anything that shows the feeds, e.g. a single-page site or a static site generator.
- No scope allows apps to set it. The owner sets it, or the server that installs the renderer: a server that installs a renderer at the domain root sets the property to the renderer's template, and removes it, if it still has that value, when it removes the renderer.
- Without the property, the site has no post pages that apps know of.

### Pages

A link with the rel `https://bigwide.net/ns/webpup-pages` makes a Markdown file in `/pups/pages/` a page of the site. A renderer shows each page with its own URL, for example as a tab.

```json
{ "rel": "https://bigwide.net/ns/webpup-pages", "type": "text/markdown", "href": "https://tim.example.com/pups/pages/now.md" },
{ "rel": "https://bigwide.net/ns/webpup-pages", "type": "text/markdown", "href": "https://tim.example.com/pups/pages/uses.md",
  "titles": { "default": "Uses", "de": "Werkzeuge" } },
{ "rel": "https://bigwide.net/ns/webpup-pages", "type": "text/markdown", "href": "https://tim.example.com/pups/pages/imprint.md",
  "properties": { "https://bigwide.net/ns/webpup-hidden": "true" } }
```

- **Order.** The pages are in the order of their links in `links`. [RFC 7033](https://www.rfc-editor.org/rfc/rfc7033#section-4.4.4) allows a JRD's order to express a preference. Apps change it with the `order` action of the [host-meta endpoint](#endpoint).
- **Title.** The link's `titles` ([RFC 7033](https://www.rfc-editor.org/rfc/rfc7033#section-4.4.4.4)): the site's language, else `default`, else any. Without `titles`, a renderer uses the first heading of the file, else its file name.
- **Hidden pages.** With the link property `https://bigwide.net/ns/webpup-hidden` set to `"true"`, the page has its URL but isn't listed, e.g. an imprint that only the footer links to.
- **Links between pages.** A Markdown link to the file of a page, such as `[Imprint](/pups/pages/imprint.md)`, leads to that page.

### Public URLs

The public URL of a file is `public_base` + its path, or, for a file under a well-known entry with a `url`, that `url` plus the rest of the path. An app uses public URLs whenever it links to its files, e.g. as the `source` of an RFC-4 webmention.

`public_base` should be on the owner's own host. A different host (a public R2 domain, a CDN host) works for readers, but breaks anything that ties a file to the owner's identity by host name, such as RFC-4's `requireSocialSpec` (see [Feed Submissions](#using-pups-for-feed-submissions-rfc-4)).

### Private areas

Two areas hold files that only the owner's apps can read. They are for personal data that belongs on the owner's domain but not in public: drafts, settings, read positions, or an inbox of notifications.

| Area | For | Who can read and write |
| --- | --- | --- |
| `/appdata/<client-host>/private/` | The app's own private data | The app, with its scope `write:/appdata/<client-host>/`. No other scope is needed. |
| `/pups/private/` | Private files that an extension of this RFC defines, e.g. an inbox feed that several apps read | Apps that hold a `read:` or `write:` scope for the declared path |

Every server offers both areas, and must meet every rule in this section.

**Paths in `/pups/private/`.** Like the other well-known paths, each one is defined by a spec, the extension that uses it, and listed in the [list of well-known paths](#the-list-of-well-known-paths). A server declares the ones it offers in `well_known_paths`, without `url`, `rels`, `properties` or `link_properties`, because private files are never linked. Apps request them with `read:` or `write:` scopes, which are in the consent group "private data" and need step-up confirmation. Like other well-known paths, they are shared: every app that holds the scope reads and writes the same files.

**Reading.** A private file is read with `GET` or `HEAD` at `storage_base` + its path, with a bearer token, exactly like a [fresh read](#fresh-reads). The server decides in this order, before it looks for the file, so that its answer never tells whether a file exists to someone who may not read it:

1. No bearer token: `401` with `WWW-Authenticate: Bearer`.
2. An invalid, expired or revoked token: `401` with `WWW-Authenticate: Bearer error="invalid_token"`.
3. A valid token whose scopes don't cover the path, by a `write:` scope or, in `/pups/private/`, a `read:` scope: `403` with `error="insufficient_scope"`.
4. The file doesn't exist: `404`.
5. Otherwise `200`, with the file and the same headers as a fresh read.

Writing works as for any other file, with the same scopes, and `write:` covers reading as well.

**Never public.** Private files have no public URL, and are never served without a token that covers them, whatever URL the request uses: not at `public_base`, not at a well-known `url`, and not through a fallback to another server. A server that falls back to the website's origin for files it doesn't have must not do so for private paths.

**Storage.** Private files must be stored where nothing serves them without PUPS: not in a public bucket, and not in a directory that the web server serves on its own. A server whose `public_base` serves its storage directly, such as a public bucket, keeps private files in a separate storage.

**Caching.** Every response for a private path, errors included, carries `Cache-Control: no-store` (`private, no-store` for a `200`) and `Vary: Authorization`, so that no cache stores a private file, or a `404` that would later hide it from the owner. A server behind a CDN that may not honour these headers, or that ignores `Vary`, configures the CDN to bypass its cache for both private areas.

**Links and properties.** Private files can't be the `href` of a link or the value of a property in `host-meta.json` (see [Editing host-meta.json](#editing-host-metajson)).

**Not encrypted.** "Private" means hidden from the public. Whoever runs the server or the hosting can read private files. Apps that store data the host must not see encrypt it before uploading it.

## Tokens

### Code exchange

```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=…&client_id=…&redirect_uri=…&code_verifier=…
```

```json
{
  "access_token": "…",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "…",
  "scope": "identify write:/appdata/app.example/",
  "me": "https://tim.example.com/"
}
```

- All clients are public clients: no client secrets. They are authenticated by PKCE and the redirect URI.
- Codes are single-use and expire after 60 seconds. Redeeming a code twice fails and **revokes every token issued from it**.
- Access tokens are opaque and live 1 hour. They are valid only at this server's upload API and links endpoint; apps must never send them anywhere else.
- `me` is the owner's identity, see [Sign-in with your domain](#sign-in-with-your-domain). It is in every token response, including refreshes.
- For an `identify`-only grant the server may omit the refresh token.

### Refresh

`grant_type=refresh_token&refresh_token=…&client_id=…` returns a new access token and a **new refresh token**, which replaces the old one. Refresh tokens live 180 days. The app's scope can't grow through a refresh.

**Reuse detection.** **Using a refresh token that has already been replaced revokes the whole grant**, because it means that the token was copied. The only exception is the grace period below.

**Grace period.** A refresh response can get lost: the server has rotated the token, but the connection dropped before the app received the new one. With strict reuse detection, the app's retry would revoke the grant. Therefore the replaced refresh token stays usable for a short **grace period** after the refresh, as long as nothing issued by that refresh has been used, neither the new refresh token nor the new access token.

- The grace period is at least 60 seconds and at most 5 minutes. Apps rely on 60 seconds only.
- A refresh with the replaced token during the grace period succeeds and returns a fresh pair. The unused tokens of the earlier refresh are revoked. Requests with them fail as with any revoked token (`invalid_grant` at the token endpoint, `401` at the upload API), but don't revoke the grant.
- Once a token issued by the refresh has been used, or the grace period has ended, using the replaced token revokes the whole grant, as above.

The grace period keeps reuse detection effective. A thief who refreshes with a copied token either uses the result, and the app's next refresh with the same token revokes the grant, or doesn't use it right away, and the app's refresh within the grace period revokes the thief's tokens. What it costs is a short window in which a refresh token whose successor is still unused can be redeemed twice.

The server records for each grant the current refresh token, the token it replaced, when it was replaced, and whether anything issued by the replacement has been used.

**Apps refresh one at a time.** The grace period is for retrying a lost response, not for concurrency. An app that refreshes from several places at once, e.g. several browser tabs, makes each refresh revoke the tokens of the one before, and revokes the grant as soon as one of them has been used. Such apps must serialize refreshes and share the result. An app that loses a refresh response retries promptly with the same refresh token, within 60 seconds.

### Revocation

`POST /oauth/revoke` ([RFC 7009](https://www.rfc-editor.org/rfc/rfc7009)) revokes a token. Apps should call it when the user disconnects. The owner can revoke a whole grant from the apps page, which invalidates all its tokens immediately.

## Upload API

All writes carry `Authorization: Bearer <access_token>`, and reads may carry it to bypass caches. Tokens are never accepted in the URL or the body. Paths are relative to `storage_base`.

| Request | Headers | Success | Meaning |
| --- | --- | --- | --- |
| `PUT <storage_base><path>` | `Content-Type`, `If-None-Match: *` | `201 Created` + `ETag` | Create; `412` if the file exists. |
| `PUT <storage_base><path>` | `Content-Type`, `If-Match: "<etag>"` | `200 OK` + `ETag` | Replace only if unchanged; `412` otherwise. |
| `PUT <storage_base><path>` | `Content-Type` only | `200`/`201` + `ETag` | Unconditional write. Allowed, discouraged. |
| `DELETE <storage_base><path>` | `If-Match` optional | `204 No Content` | Delete; `404` if missing, `412` if the ETag doesn't match. |
| `GET`/`HEAD <storage_base><path>` | none | `200` + `ETag` | Public read, no token needed. May come from a cache. |
| `GET`/`HEAD <storage_base><path>` | `Authorization` | `200` + `ETag` | Fresh read: the latest version, never from a cache. See [Fresh reads](#fresh-reads). |
| `GET`/`HEAD <storage_base><private path>` | `Authorization` with a covering scope | `200` + `ETag` | Private read; `401` without a valid token, `403` without a covering scope. See [Private areas](#private-areas). |

The body of a `PUT` is the raw file. `Content-Length` should be sent; the server rejects bodies over `max_upload_bytes` with `413` without storing anything, also for chunked uploads. A failed write never leaves a partial file: the server writes atomically.

### Fresh reads

Caches between the app and the server can serve an old version of a file for a while after a write: a CDN in front of the whole domain, a hosting provider's caching proxy, the browser's cache. Users then don't see their own changes, and read-modify-write fails with `412` again and again, because every attempt reads the same old ETag. Therefore a `GET` or `HEAD` at `storage_base` may carry the app's bearer token, and is then a **fresh read**:

- **Any valid access token of this server** is accepted for public files, whatever its scopes, `identify` included. Public files are public anyway, so the token grants nothing; it only makes the request one that caches pass on. Private files need a token whose scope covers them, see [Private areas](#private-areas). Shared caches don't store responses to requests with `Authorization` ([RFC 9111](https://www.rfc-editor.org/rfc/rfc9111#section-3.5)), and common caching proxies pass such requests through.
- **An invalid, expired or revoked token** gets `401` with `WWW-Authenticate: Bearer error="invalid_token"`, as for writes. The server never falls back to a public read, so the app learns that it must refresh.
- **The latest version.** The server answers from its storage, never from a cache of its own (CDN edge, page cache, object cache), and the response reflects every write that completed before the request was received. Its ETag is the one that a following `If-Match` is compared with.
- **Headers.** The response carries `Cache-Control: private, no-store` instead of the usual caching header, and otherwise the same headers as any uploaded file (see [Serving files](#serving-files)). Conditional requests (`If-None-Match` → `304`) and ranges work as in public reads.
- **Only `Bearer` counts.** Another `Authorization` scheme, e.g. `Basic` from a password-protected site, is not a token: the server ignores it, and the request is a public read.
- **Without a token**, the request is a public read as before. Its response should carry `Vary: Authorization`, so that a cache that stored it doesn't use it for a request with a token.
- **Routing.** A server whose web server serves the files directly, without PUPS, must route `GET` and `HEAD` requests that carry `Authorization` to PUPS.

Apps use fresh reads when they read a file in order to change it, and when they show the owner what they have just written. Readers and public URLs are unaffected: a CDN on `public_base` still needs its purge (see [Caching](#serving-files)), and the browser's own cache is bypassed by the app (e.g. `fetch(…, { cache: 'no-store' })`).

### Concurrency

ETags make read-modify-write safe when several apps, or the owner, change the same file, e.g. a feed that gets a new item:

1. `GET` the file from `storage_base` with the token (a [fresh read](#fresh-reads)) and remember its `ETag`.
2. Change it and `PUT` it with `If-Match: "<etag>"`.
3. On `412`, start again at 1.

Apps creating a file use `If-None-Match: *`. Use only ETags from `storage_base` (from a write response or a `GET`/`HEAD` there). ETags from `public_base` may come from a CDN and need not match.

### Paths

The path is percent-decoded once and must be valid UTF-8. It is rejected with `400` if:

- it is longer than 512 bytes, ends with `/`, or has an empty segment;
- a segment is `.` or `..`, or **starts with a dot** (this blocks `.htaccess`, `.user.ini`, `.git`), except for a path that equals a declared well-known file under `/.well-known/`;
- it contains `\`, `:`, NUL or other control characters;
- a segment ends with `.` or a space (Windows hosts strip them, turning `feed.php.` into `feed.php`).

The server normalizes the path first and checks the scope on the normalized path only, comparing segment by segment. A path outside every granted scope gets `403`. The server must not follow symbolic links out of its storage.

**Dot segments in the request URL.** HTTP clients, CDNs and server runtimes often resolve `.` and `..` segments in a request URL, also in the form `%2e`, before the request reaches the server, as the [WHATWG URL standard](https://url.spec.whatwg.org/#path-state) requires. In that case, the server never sees the dot segments. It treats the request as a request for the resolved path, applies the rules and the scope check to that path, and answers accordingly, typically with `403` when the resolved path is outside the scope. That is safe, because the scope check always applies to the path that is actually written. The `400` rule applies whenever the server does receive a dot segment, in particular in the `path` parameter of the [POST fallback](#post-fallback), which nothing resolves on the way.

### Errors

Errors follow RFC 6750 and carry a JSON body `{"error": "<code>", "message": "<text for humans>"}`:

| Status | `error` | When |
| --- | --- | --- |
| `400` | `invalid_path` | The path breaks the rules above. |
| `401` | `invalid_token` | Missing, expired or revoked token. With `WWW-Authenticate: Bearer error="invalid_token"`. |
| `403` | `insufficient_scope` | The path is outside the granted scope. With `WWW-Authenticate: Bearer error="insufficient_scope"`. |
| `404` | `not_found` | `DELETE` of a missing file, or a read of a missing private file. |
| `412` | `precondition_failed` | `If-Match` or `If-None-Match` failed. |
| `413` | `too_large` | Over `max_upload_bytes`. |
| `415` | `unsupported_type` | Extension not allowed, or `Content-Type` doesn't match it. |
| `507` | `insufficient_storage` | The server's storage is full or the owner's quota is reached. |

### POST fallback

Shared hosts behind a web application firewall often block `PUT` and `DELETE`. Such a server declares `post_endpoint` in its configuration document, and clients must then send every write there:

```
POST <post_endpoint>?method=PUT&path=%2Ffeed%2Fimages%2F2026%2Fsunset.jpg
POST <post_endpoint>?method=DELETE&path=%2Ffeed%2Fimages%2F2026%2Fold.jpg
```

- `method` is `PUT` or `DELETE`; `path` is the path relative to `storage_base`, percent-encoded.
- The body is the raw file, not multipart. `Authorization`, `Content-Type`, `If-Match` and `If-None-Match` work exactly as above, and status codes, errors and ETags are the same.

## File types

The server enforces a **fixed allowlist of extensions** on every write, before storing a single byte. There is no "everything that isn't blocked" rule, so obscure server-side extensions can never slip through.

**The extension decides the type.** A static web server serves a file with the type mapped from its extension, not the type it was uploaded with. So the extension is authoritative everywhere: the upload's `Content-Type` must be one of the types accepted for the extension (parameters such as `charset` are ignored), and the file is always served with the type from this table, also by servers that could store the uploaded type. A file without an allowed extension can't be written. Extensions are compared case-insensitively; apps should use lowercase.

| Extension | Served as | Accepted upload `Content-Type` |
| --- | --- | --- |
| `.json` | `application/json` | `application/json`, `application/feed+json` |
| `.xml` | `application/xml` | `application/xml`, `text/xml`, `application/rss+xml`, `application/atom+xml` |
| `.opml` | `text/x-opml` | `text/x-opml`, `application/xml`, `text/xml` |
| `.md` | `text/markdown` | `text/markdown`, `text/x-markdown` |
| `.txt` | `text/plain` | `text/plain` |
| `.csv` | `text/csv` | `text/csv` |
| `.vcf` | `text/vcard` | `text/vcard`, `text/x-vcard` |
| `.svg` | `image/svg+xml` | `image/svg+xml` |
| `.png` | `image/png` | `image/png` |
| `.jpg`, `.jpeg` | `image/jpeg` | `image/jpeg` |
| `.gif` | `image/gif` | `image/gif` |
| `.webp` | `image/webp` | `image/webp` |
| `.avif` | `image/avif` | `image/avif` |
| `.mp3` | `audio/mpeg` | `audio/mpeg` |
| `.m4a` | `audio/mp4` | `audio/mp4`, `audio/x-m4a` |
| `.ogg`, `.opus` | `audio/ogg` | `audio/ogg`, `audio/opus` |
| `.mp4` | `video/mp4` | `video/mp4`, `audio/mp4` |
| `.webm` | `video/webm` | `video/webm`, `audio/webm` |

Text types are served with `; charset=utf-8`, and text files must be UTF-8.

**Never accepted, anywhere:**

- **Web content**: HTML (`.html`, `.htm`, `.xhtml`), JavaScript (`.js`, `.mjs`) and CSS (`.css`). Apps don't write web pages.
- **Server-executable and server-config files**: `.php`, `.phtml`, `.php3`–`.php8`, `.phar`, `.pht`, `.phps`, `.inc`, `.jsp`, `.jspx`, `.asp`, `.aspx`, `.cgi`, `.pl`, `.py`, `.rb`, `.sh`, `.shtml`, `.htaccess`, `.htpasswd`, `.user.ini`. The allowlist already excludes them; this list exists for tests and clear error messages.
- **A blocked extension anywhere in the file name**, e.g. `feed.php.json`. Apache's `mod_mime` honours every extension in a name, so `AddHandler` would execute such a file.

## Serving files

How uploaded files are served is the second half of the protection. Every uploaded file, at its public URL and at `storage_base`, is served with:

| Header | Value | Why |
| --- | --- | --- |
| `Content-Type` | from the extension table | The browser treats the file as data. |
| `X-Content-Type-Options` | `nosniff` | The browser never guesses a more dangerous type. |
| `Content-Security-Policy` | `sandbox` | Opened directly, the file runs in an opaque origin: even if it contains script, the script can't touch the owner's site, cookies or owner pages. |
| `Access-Control-Allow-Origin` | `*` | Browser apps and readers can fetch the files (RFC-4 and SocialSpec require it). |
| `Access-Control-Expose-Headers` | `ETag, Location, WWW-Authenticate` | |
| `ETag`, `Last-Modified` | | Conditional requests. |

**Why the sandbox is needed even without HTML.** SVG can contain script. XML documents opened directly in a browser execute `<script>` elements in the XHTML namespace and apply XSLT stylesheets referenced by `<?xml-stylesheet?>`, and OPML and RSS are XML. Without the sandbox, an uploaded OPML file could run script with the owner's origin. Embedded with `<img>` or as a CSS background, SVG never runs script anyway; JSON served as `application/json` with `nosniff` never executes.

**A server that can't guarantee the sandbox header must not accept `.svg`, `.xml` and `.opml`** at all. Since RSS feeds need `.xml`, a server should refuse to run on a host where it can't set response headers.

**Service workers.** A request for an uploaded file with the header `Service-Worker: script` is refused with `403`. Uploads never contain JavaScript, so this is defence in depth against a file being registered as a service worker for the owner's origin.

**Caching.** Feeds and profiles change, and other servers fetch them right after an app writes them (an RFC-4 server fetches the author feed as soon as it receives the webmention). Files should be served with short cache lifetimes or `Cache-Control: no-cache`, and servers behind a CDN should purge a file's public URL when it changes. Apps that need the latest version themselves use [fresh reads](#fresh-reads), which bypass every cache.

## CORS

Browser apps are first-class clients, and missing CORS headers are the most common failure in practice, so CORS is part of conformance.

- **CORS-enabled**: `host-meta.json`, the configuration document, the RFC 8414 metadata, the token and revocation endpoints, the upload API, `post_endpoint`, `host_meta_endpoint`, and all uploaded files.
- **Preflight**: `OPTIONS` → `204` with `Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: GET, HEAD, PUT, DELETE, POST, OPTIONS`, `Access-Control-Allow-Headers: Authorization, Content-Type, If-Match, If-None-Match`, `Access-Control-Max-Age: 86400`.
- **Every response**, errors included, carries `Access-Control-Allow-Origin: *` and `Access-Control-Expose-Headers: ETag, Location, WWW-Authenticate`.
- **Not CORS-enabled**: the authorization endpoint and all owner pages (login, setup, apps).

The wildcard is safe because authentication uses bearer tokens, never cookies, and no CORS response allows credentials.

## Sign-in with your domain

Every successful authorization proves that the person controls the domain: only the owner can approve at the issuer that the domain's `host-meta.json` points to. Apps may use PUPS as "sign in with your domain", like IndieAuth. A feed-based social app, for example, writes to the user's feed and learns who the user is through the same login. This is intended, and has consequences on both sides.

### What the server provides

- **`me`** in every token response: the canonical owner URL, `https://<host>/`, where `<host>` is the domain whose `host-meta.json` points to this server. With an auth subdomain (`id.tim.example.com`), `me` is still `https://tim.example.com/`.
- **The `identify` scope** for apps that need only sign-in. Its access token can't write anything.
- **One identity per installation.** The owner is the domain. Path-based identities (`tim.example.com/~tim`) and several people per domain are not supported.

### What apps must do

1. **Normalize the identity**: the domain the user entered, as `https://<lowercase host>/`. `www.tim.example.com` and `tim.example.com` are different identities, unless one redirected to the other during discovery.
2. **Verify the chain**: `iss` in the redirect equals the discovered `issuer`, and `me` equals the normalized domain. If `me` differs, run discovery on `me` and accept it only if it leads to the same issuer.
3. **Use only their own code exchange.** Identity comes from the app's own flow: its own `client_id`, PKCE and redirect. An app must never accept an access token handed to it from elsewhere as proof of identity; any client could have obtained it. If an app's backend needs the identity, the backend redeems the code.
4. **Key accounts on `me`**, never on the issuer or the endpoints. Moving to another server implementation, or adding an auth subdomain, changes the issuer, not the identity.
5. **Manage sessions themselves.** Revoking a grant ends write access but does not log the user out of the app. To re-check identity, the app runs the flow again.

### Implications for owners

- **Whoever controls the domain is the user.** If the domain expires or is transferred, the new holder can sign in to every app as the old owner. Apps should not offer account recovery that bypasses this; they may let users link a second identity.
- **The owner login becomes more valuable.** A compromised owner login means impersonation everywhere, not just file tampering. Domains that also run web applications should use an auth subdomain (see [Deployment modes](#deployment-modes)).
- **Privacy.** Apps learn the owner's domain, which is often the person's real name. That is the purpose, and the consent screen says so.

## Editing host-meta.json

`host-meta.json` is the domain's index. An app that publishes, say, a bigwide.net feed may want the feed to be discoverable from the domain, through a link or a top-level property. Apps can't write `/.well-known/host-meta.json` themselves (it is never a declared well-known file), so PUPS edits it for them, through `host_meta_endpoint`.

**The file is the source of truth.** PUPS keeps no separate registry. The owner or their CMS may edit the file at any time, and PUPS never changes the owner's links. Properties are shared (see below).

- **Read-modify-write.** PUPS reads the file, changes only the requesting app's own entries, and writes it back conditionally (e.g. with an ETag condition, or a lock, a hash check and an atomic rename). If the file changed in between, it rereads and retries.
- **Everything else is preserved**: other links and properties, `subject` and `aliases`. If the file is not valid JSON, PUPS refuses with `409` and writes nothing. If it is missing, PUPS creates it with its own PUPS link.
- **Whitelisted only.** Apps may only add the link relations and set the properties that are allowed for their scopes (see the rules below). There is no free-form editing.
- **Not always available.** If PUPS can't write the file, e.g. because the website is served by another host, the configuration document omits `host_meta_endpoint`.

### Endpoint

`host_meta_endpoint` takes `POST` only, with a bearer token and a JSON body. Four actions exist:

```json
{ "action": "add", "rel": "https://app.example/rel/comments", "href": "https://tim.example.com/files/appdata/app.example/comments.json", "type": "application/feed+json" }
{ "action": "remove", "rel": "https://app.example/rel/comments", "href": "https://tim.example.com/files/appdata/app.example/comments.json" }
{ "action": "set-property", "property": "https://bigwide.net/ns/name", "value": "Tim" }
{ "action": "order", "rel": "https://bigwide.net/ns/webpup-pages", "hrefs": ["https://tim.example.com/pups/pages/uses.md", "https://tim.example.com/pups/pages/now.md", "https://tim.example.com/pups/pages/imprint.md"] }
```

`add` also takes `titles` and `properties` for the link:

```json
{ "action": "add", "rel": "https://bigwide.net/ns/webpup-pages", "href": "https://tim.example.com/pups/pages/imprint.md", "type": "text/markdown",
  "titles": { "default": "Imprint", "de": "Impressum" }, "properties": { "https://bigwide.net/ns/webpup-hidden": "true" } }
```

Responses: `201` added or set, `204` removed or ordered, `400` invalid request, `403` outside scope, rel or property not allowed, `409` conflict or invalid file, `413` limit reached.

### Links

- **Ownership marker.** Every link PUPS adds carries `"properties": {"https://bigwide.net/ns/pups#client": "<client_id>"}`. Links without it belong to the owner and are never changed or removed by PUPS.
- **Own files only.** `href` must be the public URL of a file inside the app's granted scope, never a [private](#private-areas) file. An app can never point the owner's links to another domain or to another app's files.
- **Allowed rels.** For a file under a well-known path, only the `rels` declared for that path. For a file in `/appdata/`, only URI rels on the client host, with or without `www.` (`https://app.example/rel/…`). Nobody can register the PUPS rel itself or identity rels such as WebFinger's `http://webfinger.net/rel/profile-page`, `me`, `authorization_endpoint` or `openid`.
- **Conflicts.** For rels that the list of well-known paths defines as single-valued, an existing link (the owner's or another app's) is not replaced; the request gets `409`.
- **Titles and properties.** `titles` maps a BCP 47 language tag or `default` to a title, as in [RFC 7033](https://www.rfc-editor.org/rfc/rfc7033#section-4.4.4.4): at most 20 entries, each title a string of at most 255 characters without control characters. `properties` may hold only the `link_properties` declared for the well-known path of the `href`, with string values of at most 2,048 characters without control characters; links to `/appdata/` take none. Nobody can set the ownership marker this way.
- **Changing a link.** `add` for a link that exists already (same `rel` and `href`) and belongs to the app replaces its `type`, `titles` and link properties with those of the request; omitted fields are removed. If the link belongs to the owner or another app, it stays as it is: without `titles` and `properties` the request succeeds, otherwise it gets `409`.
- **Order.** `order` arranges the links of one rel. `hrefs` must list every link of that rel in the file exactly once, the owner's and other apps' included; otherwise the app's view is out of date, and the request gets `409`. PUPS then puts those links into the given order, in the positions they occupy in `links`. All other links stay where they are, and the links themselves don't change: reordering the owner's links doesn't change them. The app must hold a well-known scope whose entry declares the rel; for `/appdata/` rels, it may order only rels on its own client host.
- **Limits.** At most 1,000 links in the file and 10 `/appdata/` links per app.

### Properties

Top-level `properties` in `host-meta.json` ([RFC 6415](https://www.rfc-editor.org/rfc/rfc6415#appendix-A)) map a URI to a string. PUPS lets apps set a few **shared, well-known properties** that describe the domain as a whole, such as the owner's profile name.

- **Well-known only.** An app may set only the `properties` declared for a well-known path in `well_known_paths`, and only if it holds that path's scope. There are no app-specific properties: an app that needs more, or structured data, writes its own file and links to it. Nobody can set PUPS's own properties (`https://bigwide.net/ns/pups#…`) or properties that name the domain's identity or its OAuth server.
- **No owner.** Properties are shared by the owner and every app that holds the scope. Any of them may set or replace the value; the last write wins. Apps can't remove properties; only the owner can. PUPS doesn't track who set a property.
- **Values.** A string of at most 2,048 characters, without control characters. Setting `null` or an empty string is not possible. If the list of well-known paths says that a property's value is a URL, it must be the public URL of a file inside the app's granted scope, and not a private file, as for link `href`s.

### Consent and lifecycle

- **Consent.** Adding a rel or setting a property for a well-known path is covered by that path's scope and its step-up confirmation; there is no extra scope. The details of the scope's group on the consent screen list the rels and properties it allows.
- **Revocation.** When a grant is revoked, PUPS removes the app's `/appdata/` links. Its well-known links stay, because the files stay too; they lose the ownership marker and become the owner's. Properties are never removed on revocation.
- **Owner control.** The owner can remove any link or property on the apps page.

An app that removes a page deletes its file and removes its link. Apps can't remove the footer property; an app that removes the footer empties its file.

## Owner authentication

How the owner logs in is up to the server, within these rules:

- **One owner per installation.** First-run setup is protected by a single-use bootstrap secret; afterwards the setup page is unreachable.
- **Passkeys are preferred**, with a password as fallback. A stolen password works until it is changed; an abused passkey prompt works once, at that moment.
- **Failed logins are rate-limited**, e.g. 5 per 15 minutes, then with growing delays.
- **Step-up confirmation**: every grant of a well-known scope and every revocation needs a fresh confirmation at that moment, a passkey assertion with user verification or the password. A valid session alone is not enough, so a script on the owner's origin can't approve such grants silently. Grants of the app's own directory and `identify` don't need it.
- **Sessions only on an auth subdomain.** See below.

## Deployment modes

Where the owner pages live decides how well they are isolated from the rest of the website.

**Same-domain mode.** The OAuth endpoints and owner pages share the website's origin (`https://tim.example.com/oauth/…`). Any script that runs on the website, e.g. through an HTML-injection bug in a CMS, wiki or comment form on the same domain, runs with the same origin as the owner pages. Therefore, in this mode:

- **The server never sets a cookie**: no session, no "remember me", no cookie-based single sign-on in front of it. Every owner action (approving, viewing the apps page, revoking) is confirmed with passkey or password in the same request. Without an ambient credential, a script on the website has nothing to abuse. The consent form's CSRF token is a signed hidden field, bound to the authorization request.
- Cookie `Path` attributes are not a boundary: same-origin scripts can send requests that carry any cookie of the origin.
- It is still possible for a page on the website to show a fake password prompt under the owner's real domain, or to start a passkey prompt the owner might accept. Passkeys limit the damage; the sandbox headers on uploaded files make sure that uploads can't be that page.

Same-domain mode is acceptable when the domain serves only static files. Servers in this mode should warn the owner that other web applications on the same domain put the owner's login at risk.

**Auth subdomain mode.** The OAuth endpoints and owner pages live on their own host, e.g. `id.tim.example.com`, while `host-meta.json`, the website and the files stay on the main domain. Only `issuer` and the endpoint URLs change; `me` stays `https://tim.example.com/`. The browser's same-origin policy now isolates the owner pages from everything on the main domain, so the server may keep an owner session: a host-only cookie with the `__Host-` prefix, `HttpOnly; Secure; SameSite=Lax`, 30 days at most. This mode is **strongly recommended** whenever the domain also runs web applications. Step-up confirmation applies in both modes.

## Security considerations

PUPS hands write access to the domain that is the owner's identity to third-party apps, on hosting that was not built for it. Each requirement below applies to every implementation and must be covered by a conformance test.

### Threat model

| Attacker | Wants to | Main defences |
| --- | --- | --- |
| A malicious or compromised app | Write outside its area, execute code on the server, run script on the owner's origin, keep access after revocation | Scopes, path rules, extension allowlist, sandbox headers, revocation, refresh-token rotation |
| A script on the owner's website (XSS in a CMS etc.) | Approve grants or revoke apps silently, steal the owner's login | No cookies in same-domain mode, step-up confirmation, auth subdomain, passkeys |
| A malicious client metadata host | Use the server to reach internal networks (SSRF), impersonate another app | CIMD fetch restrictions, `client_uri` and `logo_uri` on the client host only, client host on the consent screen |
| A network or mix-up attacker | Steal codes or tokens, inject a code from another server | HTTPS only, PKCE S256, `iss` check, exact redirect URIs |
| Anyone on the internet | Read private files | Token and scope check on every read, private files kept out of public storage, `no-store` on every private response |
| Someone who copied a token | Use it elsewhere, or as proof of identity | Short-lived access tokens, rotation with reuse detection, `me` only from the app's own exchange |
| An attacker who gets the domain | Sign in as the owner everywhere | None in the protocol: domain control is identity |

### Transport and OAuth

- **HTTPS only.** The server refuses to run, or to issue tokens, over plain HTTP. Only loopback redirect URIs use `http`. HSTS is recommended.
- **PKCE S256 only.** `plain` and requests without `code_challenge` are rejected.
- **Exact redirect URIs** from the CIMD document; only the port of loopback redirects may vary. Invalid clients or redirect URIs get an error page, never a redirect.
- **Codes**: single-use, 60 seconds; a reused code revokes every token issued from it.
- **Refresh tokens**: rotated on every use; reuse of a replaced token revokes the whole grant, except during the grace period of 60 seconds to 5 minutes, and only while nothing issued by the replacement has been used.
- **Issuer identification**: `iss` in every authorization response; apps compare it with the discovered issuer before redeeming the code.
- **Tokens**: at least 256 bits from a cryptographically secure random generator; only SHA-256 hashes are stored; compared in constant time; never logged, never in URLs.
- **No token passing.** Access tokens are for this server only. Apps must not accept tokens as proof of identity.

### CIMD fetch (SSRF)

Fetching a URL that anyone can put in an authorization request makes the server an HTTP client for strangers. The server fetches client metadata documents only:

- over HTTPS, with a 5-second timeout and a 64 KB size limit;
- to public addresses: private, loopback, link-local and other non-public addresses are blocked, **checked after DNS resolution**, and the connection uses the checked address (no DNS rebinding);
- without redirects to other hosts;
- never to the server's own URLs.

### Consent and app identity

- `client_name` and `logo_uri` are self-declared. The consent screen shows the client host prominently, asks whether the owner meant to log in with the app, and the "new app" notice says when an app is unknown.
- `client_uri` and `logo_uri` must be on the client host, with or without a leading `www.`. A parent domain is never accepted: a client on a shared hosting subdomain (`alice.github.io`) could otherwise present itself with the platform's URL and logo, and owners don't read subdomains. A client ID on someone else's domain is not that app's identity.
- The consent page carries a CSRF token and can't be framed (`X-Frame-Options: DENY`, `frame-ancestors 'none'`).
- Every scope outside the app's own directory is in the feeds or profile group and needs step-up confirmation. Write access includes deletion: an app with a well-known scope can delete or replace the owner's profile or feed.

### Uploaded content

- **Fixed allowlist** of data and media types; the upload's `Content-Type` must match the extension; no HTML, JavaScript or CSS; no server-executable or server-config files; blocked extensions are rejected anywhere in the name.
- **Path safety**: decoded once, normalized before the scope check, traversal and dot-segments rejected (except declared `/.well-known/` files), no backslashes, colons, control characters or trailing dots and spaces, at most 512 bytes. No symbolic links are followed out of storage.
- **Every file is served** with the type of its extension, `X-Content-Type-Options: nosniff` and `Content-Security-Policy: sandbox`. A server that can't guarantee this rejects `.svg`, `.xml` and `.opml`.
- **No service workers** from uploaded files: `Service-Worker: script` requests get `403`.
- **No server-side execution** in the storage directory: on Apache, an `.htaccess` there removes all handlers (`RemoveHandler`, `SetHandler none`, `php_flag engine off` where allowed), disables `Options Indexes` and `MultiViews`, and denies dotfiles. Where a well-known path lies outside that directory, e.g. under `/.well-known/`, the allowlist is the only protection against execution, and the server must make sure the sandbox headers are sent there too.
- **`/.well-known/` is dangerous ground.** Only single files from the bigwide.net list may be declared there, never `acme-challenge/` or other paths that prove domain control to third parties.

### Abuse by approved apps

An approved app can publish anything it likes in its own directory, under the owner's domain: spam, malware disguised as media, illegal content. The protocol can't prevent that; it limits it to apps the owner approved, and lets the owner remove them. Servers therefore should:

- record which grant last wrote each file, and let the owner delete all files of an app from the apps page;
- support an overall storage quota, answering `507` when it is reached;
- rate-limit writes per grant.

### Owner login and identity

- Rate-limited login; passkeys preferred; step-up for well-known grants and revocations.
- No cookies in same-domain mode; in auth subdomain mode, `__Host-` cookies with `HttpOnly; Secure; SameSite=Lax`.
- The bootstrap secret is single-use, and setup is unreachable once the owner exists.
- A compromised owner login allows impersonation in every app that uses sign-in with the domain, not just file tampering. Servers should tell the owner so during setup.

### Privacy

- Everything uploaded outside the [private areas](#private-areas) is public, including the rest of `/appdata/`. Apps must not upload private data anywhere else, and must make that clear to the user. Unlisted URLs are not secret.
- Private files are protected from the public, not from whoever runs the server or the hosting: they are not encrypted. Apps that need more encrypt the data themselves before uploading it.
- Apps learn the owner's domain, which often is their name.
- SocialSpec's `contact.vcf` typically contains an e-mail address or phone number; apps that write it should show the user what will be published.

## Using PUPS for Feed Submissions (RFC-4)

An RFC-4 client needs an **author feed** on the user's domain. With PUPS:

1. The user enters their domain. The client discovers PUPS and asks for `write:/appdata/<client-host>/`. That is a one-click grant, and the token response's `me` tells the client the user's domain.
2. The client keeps the author feed in its own directory, e.g. `/appdata/app.example/comments.json` (JSON Feed) or `comments.xml` (RSS). RFC-4 allows one feed per target site, or one per comment, which keeps each file small. Alternatively, the client asks for `write:/feed.json` (with step-up confirmation) and adds the comments to the owner's [main feed](#the-list-of-well-known-paths), where the owner's readers see them too.
3. For each comment, the client adds the item with `inReplyTo` and, on the feed, the socialspec account `@tim.example.com`, then writes the feed with `If-Match` (or `If-None-Match: *` for a new feed).
4. It sends the webmention with `source` = the feed's **public URL** + `#guid`.
5. To edit, it changes the item and resends the webmention; to delete, it removes the item (or the feed file) and resends it.

Points to watch:

- **Authorship needs the owner's host.** RFC-4's `requireSocialSpec` requires the socialspec account's host to equal the author feed's host. The feed is at `public_base`, so `public_base` must be on the host named in `me`. A client should compare them and warn the user if they differ, since targets that require SocialSpec will reject the comment.
- **Freshness.** The RFC-4 server fetches the author feed seconds after the upload. A CDN or a long cache lifetime on `public_base` can serve the old version and make the submission fail with `item-not-found`; see [Caching](#serving-files).
- **CORS.** Uploaded files carry `Access-Control-Allow-Origin: *`, so author feeds can also be read by browser clients, although RFC-4 doesn't require it.
- **Types.** JSON Feeds are written as `.json` with `application/feed+json` or `application/json`, RSS as `.xml`; both are served as the generic type of their extension, which RFC-4 servers must accept.

## Built-in receiver

A PUPS server may receive what other sites send to its owner, and write it into the owner's [inbox (RFC-12)](https://github.com/bigwide-net/RFCs/blob/main/RFC-12-Inbox-Feeds.md), `/pups/private/inbox.json`, with `pups` as its `writer`. It needs no grant, and nothing to set up. It has two parts, with two endpoints, because they mean different things: a mention asks for attention, a comment to be published.

- **Mentions.** The server receives [RFC-11](https://github.com/bigwide-net/RFCs/blob/main/RFC-11-Mentions.md) mentions at `mention_endpoint`, and links it from `host-meta.json` with the relation `https://bigwide.net/ns/mention`, as RFC-11 allows for a receiver that the PUPS server runs itself. It adds the link once; if the owner removes it, it stays removed.
- **Comments.** The server accepts [RFC-4](https://github.com/bigwide-net/RFCs/blob/main/RFC-4-Feed-Submissions.md) comments at `comment_endpoint`, for posts in the owner's feeds that it serves. A client that publishes a feed opts in by writing `_submissions` into its feed file, with `url` set to `comment_endpoint`, a `comments` policy, and `republish` set to what the server does, so far `no`: the comments go into the inbox, and the owner decides what happens to them. The server enforces the rules of the feed's policy, and refuses comments for feeds whose `_submissions` names another endpoint. For a comment on an item in an archive, it uses the `_submissions` of the feed file at the archive's `_archive.updates_url` ([RFC-13](https://github.com/bigwide-net/RFCs/blob/main/RFC-13-JSON-Feed-Archives.md)), as [RFC-7 "Receiving comments"](https://github.com/bigwide-net/RFCs/blob/main/RFC-7-Feed-Publishing-with-PUPS.md#receiving-comments-rfc-4) describes.

The server declares `/pups/private/inbox.json` in `well_known_paths`, so that the owner's readers can ask for `read:/pups/private/inbox.json`. A client learns from `mention_endpoint` that the mention link in `host-meta.json` belongs to the PUPS server, and from `comment_endpoint` where comments on the owner's feeds go. Both are optional; a server without a receiver leaves them out.

## Using PUPS for SocialSpec

[SocialSpec](https://socialspec.org) defines three files at fixed locations on the user's domain:

| Spec | File | Type |
| --- | --- | --- |
| [FollowSpec](https://followspec.org) | `/.well-known/following.opml` | `text/x-opml` |
| [AboutSpec](https://aboutspec.org) | `/.well-known/about.md` | `text/markdown` |
| [ContactSpec](https://contactspec.org) | `/.well-known/contact.vcf` | `text/vcard` |

A server that can write and serve the domain root declares them as well-known files with their `url` at the root, as in the example configuration. An app that edits the user's profile asks for exactly the files it needs, e.g. `write:/.well-known/about.md`, which needs step-up confirmation. Images referenced from `about.md` go in the app's own directory or a declared well-known directory. All three files are served with `Access-Control-Allow-Origin: *`, as SocialSpec requires, and with the sandbox headers like any upload.

If PUPS can't write the domain root (e.g. because the website runs on another host, or the server has no write access to `/.well-known/`), it doesn't declare these files, and apps tell the user that their profile must be published by other means.

## Relation to other specs

- **IndieAuth** also identifies apps by URL and returns `me`. PUPS uses the IETF CIMD draft instead, which is the direction OAuth itself has taken, and discovers the server through `host-meta.json` instead of HTML links, so that the user's site needs no HTML at all. A future version could additionally accept IndieAuth clients.
- **Micropub** lets apps create posts that the server turns into web pages. PUPS stores files that the app has fully formed, and never produces HTML, which keeps the server small and the security model simple.
- **remoteStorage** is per-user storage for app data, with read access and its own discovery via WebFinger. PUPS is mostly public, with only small private areas for the owner's apps, and designed to publish files at the locations other specs expect.
- **Solid** offers private, access-controlled storage with linked-data semantics. PUPS deliberately does much less, so that it can run on shared PHP hosting.

## Conformance

A test runner takes a base URL and a test client with its own CIMD document, and checks at least:

- Discovery: `host-meta.json` links to the configuration document; required fields are present; endpoints are reachable; CORS headers on both.
- The full code flow with PKCE, including `iss` in the redirect and `me` in the token response.
- Rejected: wrong `code_verifier`, `plain` PKCE, missing PKCE, reused code (and revocation of its tokens), expired code, `redirect_uri` not in the CIMD document, `client_id` mismatch, `client_uri` or `logo_uri` on another host than the client host (a parent domain, another subdomain; only a `www.` difference is allowed), a `client_id` resolving to a private address.
- Scopes: another app's directory, `write:/`, undeclared paths → `invalid_scope`; narrowed grants are enforced.
- Refresh rotation and reuse detection; revoked grants fail immediately.
- Refresh grace: reusing a replaced refresh token within 60 seconds, before anything issued by the replacement was used, succeeds, and the unused tokens of the earlier refresh are then rejected without revoking the grant; reuse after the replacement's refresh or access token was used, or after the grace period, revokes the grant.
- A loopback redirect URI listed in the CIMD document is accepted with any port; `localhost` is not.
- Uploads: inside scope succeeds, outside → `403`, no or expired token → `401`; `If-None-Match: *` on an existing file → `412`; stale `If-Match` → `412`; fresh `If-Match` → `200` with a new ETag.
- Paths: `..`, `%2e%2e`, backslashes, dotfiles, trailing dots → `400`. Dot segments are tested through the `path` parameter of `post_endpoint` when it is declared; sent in the URL, they may be resolved on the way (see [Paths](#paths)), so the test accepts either `400` or the answer for the resolved path, but never a write outside the scope; a declared `/.well-known/` file is writable, an undeclared one and `host-meta.json` are not.
- Types: `.php`, `.phtml`, `feed.php.json`, `.htaccess`, HTML, JavaScript, CSS, a mismatched `Content-Type` → `415`, nothing stored; oversized uploads → `413`, nothing stored.
- Serving: every uploaded file has the type of its extension, `nosniff`, `sandbox` and CORS headers; `Service-Worker: script` → `403`.
- Fresh reads: `GET` and `HEAD` at `storage_base` with a valid token of any scope, `identify` included, return the latest write with its ETag and `Cache-Control: private, no-store`, also right after a write; an invalid, expired or revoked token → `401`; without a token, the public read is unchanged and carries `Vary: Authorization`.
- CORS on preflights and on every response, including errors.
- All upload tests repeated through `post_endpoint` when it is declared.
- Private areas: a private file can be written and read back with a token whose scope covers it; without a token → `401`, with an invalid token → `401`, with a valid token of another app or without a covering scope → `403`, a missing file with a covering token → `404`, all with `Cache-Control: no-store` and `Vary: Authorization`; the file can't be read at `public_base`, at a well-known `url` or through any other route without a covering token; a `read:` token can read but not write; host-meta links and URL properties pointing to private files → `403`.
- An `identify`-only token can't write.
- Same-domain mode sets no cookie on any response.
- `host-meta.json` links: owner entries untouched, foreign `href`s and reserved rels rejected, concurrent changes retried, invalid JSON refused with `409`, limits enforced, `/appdata/` links removed on revocation.
- Link titles and properties: undeclared link properties and the ownership marker → `403`; overlong titles or values → `400`; `add` replaces the app's own link's titles and properties, and gets `409` for another's link.
- `order`: the links of the rel are reordered in their positions and nothing else moves; a list that misses a link, has one twice or names one that doesn't exist → `409`; a rel without a granted scope → `403`.
- `host-meta.json` properties: only properties declared for a granted well-known path can be set, others → `403`; there is no way for apps to remove a property; overlong values, control characters and URL values outside the scope are rejected; other links and properties are untouched.

## Open questions

- **Names and URIs.** The link relation `https://bigwide.net/ns/personal-storage`, the ownership property `https://bigwide.net/ns/pups#client`, the site properties `https://bigwide.net/ns/name`, `title`, `summary`, `theme-color`, `language`, `avatar`, `banner` and `webpup-footer`, the pages relation `https://bigwide.net/ns/webpup-pages` and its link property `webpup-hidden`, and the feed relation `https://bigwide.net/ns/feed` are placeholders, to be published on bigwide.net.
- **The list of well-known paths.** This RFC defines the initial list. Open: its home on bigwide.net, how new entries are proposed and accepted, and which link relations future entries bring.
- **Sharing private files.** Should a later version let the owner share a private file with other people, e.g. with [Identity Tokens (RFC-10)](https://github.com/bigwide-net/RFCs/blob/main/RFC-10-Identity-Tokens.md)?
- **Sender-constrained tokens.** Should a later version support [DPoP](https://www.rfc-editor.org/rfc/rfc9449), so that a copied token is useless without the app's key?
- **More types.** Are there file types that SocialSpec or feed apps need beyond the allowlist (e.g. `.ics` calendars, `.pdf`)?
