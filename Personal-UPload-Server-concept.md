# Personal Upload Server (PUPS) — Implementation Brief

Nov 12, 2026 · @Tim Jansen

## Purpose and scope

Build PUPS (Personal Upload Server), a small, self-hosted server that lets third-party apps get scoped credentials to upload files to a person's own domain, using standard OAuth 2.1 with PKCE and Client ID Metadata Documents (CIMD). It is the write side for socialspec-style personal publishing (e.g. bigwide.net feeds): an app only needs the user's domain, needs no pre-registration, and ends up with a token that can PUT files under the paths the owner approved.

There are two implementations of the same protocol. Apps must not be able to tell them apart.

1. **Phase 1 — Cloudflare Worker** (build first): Worker + R2 for files + KV for OAuth state, distributed as a public repo with a Deploy to Cloudflare button.
2. **Phase 2 — PHP drop-in**: a ZIP the owner uploads to ordinary shared web hosting (IONOS, Strato, all-inkl, Namecheap, DreamHost, …), using SQLite and the local filesystem.

**In scope:** discovery, authorization and consent, token issuance/refresh/revocation, authenticated uploads (PUT/DELETE) with ETag concurrency control, CORS for browser apps, an owner login, an app-management page, and a reference test client.

**Out of scope:** website content (apps never write HTML or JavaScript), multi-user hosting (one installation = one owner), the socialspec feed formats themselves, reading/serving private files (published files are public), and federation or notifications.

## Architecture

Everything an app touches lives on the owner's domain; storage is never exposed to apps directly, and only published files are readable by anyone.

&#91;embedded content: architecture · same for Worker and PHP\]

Numbered steps: 1 discover endpoints · 2 authorize, then exchange the code for tokens · 3 the server fetches the app's client metadata · 4 the owner logs in and approves · 5 the app uploads with its bearer token · 6 anyone reads published files without a token.

## Protocol specification

The protocol is plain OAuth 2.1 (authorization code + PKCE S256) with CIMD client IDs, plus one discovery document and a PUT-based upload API. Both implementations must pass the same conformance tests.

### 1. Discovery

The app starts from the owner's domain or URL (e.g. `tim.example`), normalizes it to `https://tim.example/`, and discovers the server in two steps:

1. **host-meta, JSON form only (RFC 6415):** `GET https://tim.example/.well-known/host-meta.json`. Apps never request or parse the XML `host-meta`, and servers need not serve it.
2. **Config document:** the app takes the link whose `rel` is `https://bigwide.net/ns/personal-storage` (placeholder; a URI-form relation needs no IANA registration) and fetches the JSON config its `href` points to.

`/.well-known/host-meta.json`:

```json
{
  "links": [
    {
      "rel": "https://bigwide.net/ns/personal-storage",
      "type": "application/json",
      "href": "https://tim.example/personal-storage.json"
    }
  ]
}
```

Config document (`personal-storage.json`, path is a placeholder):

```json
{
  "issuer": "https://tim.example",
  "authorization_endpoint": "https://tim.example/oauth/authorize",
  "token_endpoint": "https://tim.example/oauth/token",
  "revocation_endpoint": "https://tim.example/oauth/revoke",
  "storage_base": "https://tim.example/files/",
  "public_base": "https://tim.example/files/",
  "scopes_supported": ["identify", "write:/appdata/"],
  "well_known_paths": ["/feeds/", "/profile.json"],
  "host_meta_endpoint": "https://tim.example/pups/host-meta",
  "max_upload_bytes": 26214400,
  "spec_version": "0.1"
}
```

- Both documents are served as `application/json` with `Access-Control-Allow-Origin: *`. Clients stay on HTTPS and follow at most 3 redirects.
- The config may live on another host (e.g. `storage.tim.example`); only `host-meta.json` must be served on the owner's domain itself.
- The server also serves RFC 8414 metadata at `/.well-known/oauth-authorization-server` (the Cloudflare library does this automatically) for generic OAuth tooling. The config document is authoritative.
- `storage_base` is where apps write (authenticated). `public_base` is where the same files are read publicly; it may differ (e.g. a public R2 domain or CDN).
- `post_endpoint` (optional) appears only when the server cannot accept PUT/DELETE; see 6a.

### 2. Client identity (CIMD)

- `client_id` is an HTTPS URL with a non-root path, e.g. `https://app.example/oauth/client.json`, per the IETF draft *OAuth Client ID Metadata Document* (draft-ietf-oauth-client-id-metadata-document-03).
- Every client MUST publish its own metadata document, and its `client_id` URL MUST be on the client's own domain. Client IDs hosted on someone else's domain are rejected.
- `client_uri` MUST be on the same domain as `client_id` (same registrable domain, e.g. `app.example` and `www.app.example`); the server rejects the document otherwise. The consent screen shows this domain as the app's identity.
- The document is JSON with RFC 7591 fields: `client_id` (must equal the URL), `client_name`, `client_uri`, `logo_uri`, `redirect_uris`.
- The server fetches it during authorization, rejects it if `client_id` does not match or `redirect_uri` is not listed, and caches it (respecting HTTP cache headers, max 24 h).
- Native/desktop apps may register loopback redirects (`http://127.0.0.1/callback`), with any port accepted at runtime per RFC 8252.

### 3. Authorization request

The app generates `code_verifier` (43–128 chars), `code_challenge = BASE64URL(SHA256(verifier))` and a random `state`, then redirects the browser to:

```
GET /oauth/authorize?response_type=code
  &client_id=https://app.example/oauth/client.json
  &redirect_uri=https://app.example/callback
  &scope=write:/feeds/bigwide/
  &state=…
  &code_challenge=…
  &code_challenge_method=S256
```

The server validates the request and client, requires owner login, shows the consent screen (see User flows), and on approval redirects to `redirect_uri?code=…&state=…&iss=https://tim.example` (RFC 9207 issuer identification). On denial: `error=access_denied`.

### 4. Scopes

- Format: `write:<path-prefix>`; several scopes are space-separated. `<path-prefix>` is relative to `storage_base`, starts with `/`, and if it ends with `/` covers the whole subtree; otherwise exactly one file.
- `write` includes create, replace and delete. No read scope: everything stored is public.
- `write:/` is not a valid scope: only an app's own /appdata/ and the declared well-known paths can be granted.
- The owner may narrow scopes on the consent screen; the granted scope is returned in the token response and is what is enforced.

Scopes are plain OAuth scope strings; RFC 9396 `authorization_details` is not used. Paths in scopes are limited to unreserved URL characters, `/` and percent-encoding.

Only two kinds of path scope exist: the client's own `/appdata/<client-host>/`, and the well-known paths the server declares in its config (`well_known_paths`). The server rejects any other scope.

### 4a. File layout

Clients organize their own files. The server offers no directory listing; a client that needs to know what it has stored keeps track of it in its own files (e.g. an index file in its app directory).

- **Default — app data:** a client stores files only under `/appdata/<client-host>/`, where `<client-host>` is the host of its `client_id` (e.g. `/appdata/app.example/`). The matching scope is `write:/appdata/app.example/`, which the consent screen shows as "the app's own data". The server MUST reject a request for another client's app directory. HTML is never allowed here; SVG is. Everything under `/appdata/` is served sandboxed (see 6b).
- **Exception — well-known files:** files whose location is fixed by a spec (e.g. feed or profile files defined by bigwide.net) may be written outside `/appdata/` only under a path listed in the config's `well_known_paths` and only when the owner has granted a scope for it. The same extension allowlist applies as in `/appdata/` (see 6b), so a feed directory can hold its images and other media. Clients request each such scope explicitly and create nothing else next to it.
- No app may write HTML or JavaScript, and no app may write files the server could execute (see 6b).
- The server enforces scopes and file types; the rest are client obligations. The consent screen visibly marks every scope outside the client's own app directory and requires step-up confirmation for it.

### 5. Token endpoint

```
POST /oauth/token  (application/x-www-form-urlencoded)
grant_type=authorization_code&code=…&client_id=…&redirect_uri=…&code_verifier=…
```

Response:

```json
{
  "access_token": "…",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "…",
  "scope": "write:/feeds/bigwide/"
}
```

- Codes are single-use and expire after 60 s. Access tokens live 1 h; refresh tokens 180 days and rotate on every use (`grant_type=refresh_token`).
- No client secrets: all clients are public clients authenticated by PKCE and redirect URI.
- The app checks that `iss` from step 3 equals the discovered `issuer` before redeeming the code.
- `POST /oauth/revoke` (RFC 7009) revokes a token; the owner can also revoke from the apps page.

### 6. Upload API

All write requests carry `Authorization: Bearer <access_token>`. Paths are relative to `storage_base`.

| Request | Headers | Success | Meaning |
| --- | --- | --- | --- |
| `PUT /files/<path>` | `Content-Type`, `If-None-Match: *` | `201 Created` + `ETag` | Create; fails with `412` if the file exists |
| `PUT /files/<path>` | `Content-Type`, `If-Match: "<etag>"` | `200 OK` + `ETag` | Replace only if unchanged; `412` otherwise |
| `PUT /files/<path>` | `Content-Type` only | `200`/`201` + `ETag` | Unconditional write (allowed, discouraged) |
| `DELETE /files/<path>` | `If-Match` optional | `204 No Content` | Delete |
| `HEAD` / `GET /files/<path>` | none | `200` + `ETag` | Public read (no token needed) |

Errors use RFC 6750 conventions: `401` + `WWW-Authenticate: Bearer error="invalid_token"` for missing/expired tokens, `403` + `error="insufficient_scope"` when the path is outside the granted scope, `413` above `max_upload_bytes`, `400` for invalid paths, `412` for failed preconditions.

Path rules: UTF-8, percent-decoded once, segments must not be empty, `.` or `..`, no backslashes or control characters, max 512 bytes. Paths ending in `/` cannot be written, and no segment may start with a dot (this also blocks .htaccess and .user.ini). There is no directory listing (see 4a).

### 6a. POST fallback for writes

Some hosts block PUT and DELETE, which is common on shared PHP hosting behind a WAF. Such a server declares `post_endpoint` in the config; when it is present, clients MUST send all writes there. Servers that accept PUT/DELETE omit the field.

```
POST <post_endpoint>?method=PUT&path=/feeds/bigwide/feed.xml
POST <post_endpoint>?method=DELETE&path=/feeds/bigwide/old.xml
```

- `method` is `PUT` or `DELETE`; `path` is the path relative to `storage_base`, percent-encoded.
- The body is the raw file bytes (not multipart). `Authorization`, `Content-Type`, `If-Match` and `If-None-Match` work exactly as in the table above, and status codes, errors and ETags are identical.
- CORS: the preflight for `post_endpoint` allows POST with the same request headers.

### 6b. File types and executable content

The server enforces a file-type allowlist on every write, by extension, and the `Content-Type` must match the extension. Anything else is rejected with `415 Unsupported Media Type` before a single byte is stored.

**The extension decides the served type.** On static web servers, which is how the PHP drop-in serves files, the `Content-Type` a reader receives comes from the server's extension mapping, not from the `Content-Type` sent with the upload. So the extension is authoritative everywhere: the upload's `Content-Type` is only checked against it (mismatch → `415`), and the Worker also serves the type mapped from the extension rather than whatever was uploaded, so both implementations behave the same. Clients MUST pick the correct extension (`feed.xml`, `subs.opml`, `data.json`); a file without a known extension cannot be written. The PHP `.htaccess` adds `AddType` lines for types hosters often miss (e.g. `text/x-opml` for `.opml`, `image/avif`, `image/webp`, text/markdown).

The allowlist is a fixed set of extensions; there is no generic "anything not blocked" rule, so new or obscure server-side extensions can never slip through.

| Area | Allowed extensions |
| --- | --- |
| `/appdata/<client-host>/` and well-known paths declared in the config | `.json`, `.opml`, `.md`, `.xml`, `.txt`, `.csv`, `.svg`, `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.avif`, `.mp3`, `.m4a`, `.ogg`, `.opus`, `.mp4`, `.webm` |
| Anything else | Nothing |

**SVG** is allowed but only ever served sandboxed. Embedded with `<img>` or as a CSS background, browsers never run SVG scripts anyway; opened directly or embedded with `<object>`/`<iframe>`, the `Content-Security-Policy: sandbox` header on the SVG response puts it in an opaque origin, so its scripts cannot reach the site. If a server cannot guarantee that header (see the PHP installer check), it must reject `.svg`, `.xml` and `.opml` uploads instead of serving them unsandboxed.

HTML (`.html`, `.htm`, `.xhtml`), JavaScript (`.js`, `.mjs`) and CSS are never accepted anywhere: apps do not write web pages.

- **Never writable by any app:** server-executable or server-config files, e.g. `.php`, `.phtml`, `.php3`–`.php8`, `.phar`, `.pht`, `.inc`, `.jsp`, `.jspx`, `.asp`, `.aspx`, `.cgi`, `.pl`, `.py`, `.rb`, `.sh`, `.shtml`, `.htaccess`, `.htpasswd`, `.user.ini`. The allowlist is the real protection; this list drives tests and error messages. A blocked extension anywhere in the name is rejected too (`feed.php.json`), because Apache's `AddHandler` executes such files.
- **Sandbox headers:** every uploaded file, without exception, is served with `Content-Security-Policy: sandbox` and `X-Content-Type-Options: nosniff`. SVG, XML and OPML can still carry script (OPML is XML), so the sandbox stays even though HTML and JavaScript are excluded.
- **Why OPML and XML need it:** browsers render XML documents opened directly and execute `<script>` elements in the XHTML namespace inside them, and they apply XSLT stylesheets referenced by `<?xml-stylesheet?>`. Without the sandbox, an OPML file could run script with the site's origin. JSON served as `application/json` with `nosniff` never executes.
- **Worker:** public reads go through the Worker (or a Cloudflare Transform Rule on the R2 custom domain) so these headers are always set.
- **PHP:** the `files/` folder gets an `.htaccess` that removes script handlers (`RemoveHandler`, `SetHandler none`, `php_flag engine off` where supported) and denies dotfiles. Where well-known paths lie in the web root, the allowlist is the only protection there.

### 7. CORS (mandatory)

Every endpoint used by browsers must answer CORS correctly; this is the most common real-world failure, so it is part of conformance.

- `OPTIONS` preflight → `204` with `Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: GET, HEAD, PUT, DELETE, OPTIONS`, `Access-Control-Allow-Headers: Authorization, Content-Type, If-Match, If-None-Match`, `Access-Control-Max-Age: 86400`.
- All responses (including errors) carry `Access-Control-Allow-Origin: *` and `Access-Control-Expose-Headers: ETag, Location, WWW-Authenticate`.
- Wildcard origin is safe because auth is via bearer token, never cookies. The `/oauth/authorize` page and owner pages are NOT CORS-enabled.

### 8. Using authorization as identity

Every successful authorization also proves that the person controls the domain, because only the owner can approve at the issuer that the domain's `host-meta.json` points to. Apps may therefore use it as "sign in with your domain", as IndieAuth does. Example: an RSS-based social app uploads into the user's feed and, through the same login, learns who the user is (the domain). This is intended, and it has consequences on both sides.

#### What the server adds

- **`me` in the token response:** the canonical owner URL, e.g. `"me": "https://tim.example/"`, meaning the domain whose `host-meta.json` points to this config. With the auth subdomain, `me` is still the main domain, not `id.tim.example`.
- **Identity-only scope `identify`:** for apps that only need sign-in. The token response carries `me`, and the access token cannot write anything. Like grants for an app's own `/appdata/`, it needs no step-up confirmation.
- **Consent screen:** always states that the app will learn the owner's domain; with `identify` alone it reads "Sign in to app.example as tim.example".
- **One identity per installation:** the owner is the domain. Path-based identities (`tim.example/~tim`) and several people per domain are not supported.

#### What apps must do

1. **Normalize the identity:** the domain the user entered, as `https://<lowercase host>/` with no port or path. `www.tim.example` and `tim.example` are different identities unless one redirects to the other during discovery.
2. **Verify the chain:** `iss` in the redirect equals the config's `issuer`, and `me` from the token response equals the normalized domain. If they differ, run discovery on `me` and accept it only if it leads to the same issuer.
3. **Use only your own code exchange:** identity comes from the app's own flow (its own `client_id`, PKCE and redirect). Never accept an access token handed in from elsewhere as proof of identity; tokens are meant for the storage server and could have been obtained by any other client. If an app's backend needs the identity, the backend performs the code exchange.
4. **Key accounts on `me`, never on the issuer or endpoints:** moving from the Worker to PHP, or adding the auth subdomain, changes the issuer but not the identity.
5. **Sessions are the app's business:** revoking the grant ends write access but does not log the user out of the app. Apps that want to re-check identity run the flow again; for an already-approved app that needs only a valid owner session in subdomain mode, while in same-domain mode the owner confirms with password or passkey every time.

#### Implications for owners

- **Whoever controls the domain is the user.** If the domain expires or is transferred, the new holder can sign in to every app as the old owner. Apps should not offer recovery that bypasses this; if they want recovery, they can let users link a second identity.
- **Higher stakes for the owner login.** A compromised owner login or issuer now means impersonation everywhere, not just file tampering. Domains that also run web applications should use the auth subdomain (Worker), and the PHP installer warning mentions impersonation, not only file access.
- **Privacy:** apps learn the owner's domain, which is often the person's real name. That is the purpose, but the consent screen must make it visible.

### 9. Editing host-meta.json (links and properties)

Apps never write `/.well-known/host-meta.json` themselves (dot-segments are rejected). Instead PUPS edits the file on their behalf, so an app that publishes, for example, a socialspec about page can make sure it is properly declared, with a link or a top-level property.

**The file is the source of truth.** PUPS keeps no separate registry. The owner or their CMS may edit the file at any time, and PUPS must never change the owner's links. Properties are shared (see below).

- **Read-modify-write:** PUPS reads the file, changes only the requesting app's own entries, and writes it back conditionally: the Worker uses an R2 `onlyIf` ETag condition; PHP takes a lock, checks that the file's hash is unchanged, writes a temp file and renames it. If the file changed in between, PUPS rereads and retries.
- **Preserve everything else:** other links and properties, `subject` and `aliases` are kept. If the file is not valid JSON, PUPS refuses with `409` and never writes it.
- **Whitelisted only:** apps may only add the rels and set the properties allowed for their scopes; there is no free-form editing.
- **Missing file:** if no `host-meta.json` exists, PUPS creates it, including its own PUPS link.
- **Out of reach:** if PUPS cannot write the file (e.g. the website is served by another host that the Worker does not control, or the PHP install cannot write to the domain root), the config omits `host_meta_endpoint` and apps know editing is unavailable.

**Endpoint.** The config declares `host_meta_endpoint`. It takes POST only, so it works on hosts that block PUT/DELETE:

```json
{ "action": "add", "rel": "…", "href": "https://tim.example/feeds/bigwide/about.json", "type": "application/json" }
{ "action": "remove", "rel": "…", "href": "https://tim.example/feeds/bigwide/about.json" }
{ "action": "set-property", "property": "…", "value": "Tim" }
```

Responses: `201` added or set, `204` removed, `400` invalid request, `403` outside scope or rel/property not allowed, `409` conflict or invalid file, `413` limit reached.

**Rules for links:**

- **Ownership marker:** every link PUPS adds carries a property naming the app, e.g. `"properties": {"https://bigwide.net/ns/pups#client": "https://app.example/oauth/client.json"}`. Links without this property belong to the owner and are never changed or removed by PUPS.
- **Same-domain targets only:** `href` must point to a file on the owner's domain inside the app's granted scope (its own `/appdata/` or a granted well-known path). An app can never point the owner's links at another domain.
- **Rel namespaces:** links into a well-known path may use only the rels PUPS associates with that path in its config; links into `/appdata/` may use only URI rels on the app's own domain (e.g. `https://app.example/rel/…`). Nobody can register the PUPS rel itself or common identity rels such as WebFinger's profile page.
- **Conflicts:** for single-valued well-known rels, an existing link (the owner's or another app's) is not replaced; the request gets `409`.
- **Limits:** at most 1,000 links in the file overall and 10 `/appdata/` links per app.

**Rules for properties** (top-level `properties`, URI → string, RFC 6415):

- **Well-known only:** properties are a few shared, well-known values such as the owner's profile name. An app may set only the properties PUPS associates with a well-known path in its config, and only if it holds that path's scope. There are no app-specific properties: an app that needs more, or structured data, writes its own file and links to it. Nobody can set PUPS's own properties (`https://bigwide.net/ns/pups#…`) or properties naming the domain's identity or OAuth server.
- **No owner:** properties are shared by the owner and every app holding the scope; any of them may set or replace a value, last write wins. Apps cannot remove properties; only the owner can. PUPS does not track who set a property.
- **Values:** a string of at most 2,048 characters without control characters; no `null` and no empty string. If the property's spec defines its value as a URL, it must be the public URL of a file inside the app's granted scope, as for link `href`s.

**Consent and lifecycle:**

- **Consent:** adding a well-known rel or setting a well-known property is covered by that path's well-known scope and its step-up confirmation; no extra scope. The consent screen lists the rels and properties the scope allows.
- **Lifecycle:** when a grant is revoked, the app's `/appdata/` links are removed. Its well-known links stay, because the files stay too, and lose their ownership marker, so they become the owner's. Properties are never removed on revocation. The owner can remove any link or property from `/apps`.

## Phase 1: Cloudflare Worker

One Worker in TypeScript serves every endpoint; `@cloudflare/workers-oauth-provider` (v1, split API) does the OAuth server work, and the code we write is the consent UI, owner login, scope enforcement and the R2 upload handler.

### Stack and bindings

| Item | Choice | Notes |
| --- | --- | --- |
| Runtime | Cloudflare Workers, TypeScript, Wrangler | `wrangler.jsonc` checked in |
| OAuth server | `@cloudflare/workers-oauth-provider` v1 | Enable CIMD explicitly; requires compatibility flag `global_fetch_strictly_public` |
| OAuth state | KV namespace `OAUTH_KV` | Grants, hashed tokens, cached client metadata (library-managed) |
| Owner data | KV namespace `OWNER_KV` (or D1 if needed) | Owner passkey/password record, sessions, setup-done flag |
| Files | R2 bucket `FILES` | Key = path under `storage_base`, no leading slash |
| Secrets | `BOOTSTRAP_SECRET`, `SESSION_SIGNING_KEY` | Prompted by the deploy button via `.dev.vars.example` |
| Vars | `MAX_UPLOAD_BYTES`, `PUBLIC_BASE` | Defaults in `wrangler.jsonc` |

### Routes

| Route | Handler |
| --- | --- |
| `GET /.well-known/host-meta.json, /personal-storage.json` | host-meta.json and config document, built from request origin + vars |
| `/.well-known/oauth-authorization-server`, `/oauth/token`, `/oauth/revoke` | workers-oauth-provider |
| `GET/POST /oauth/authorize` | Our consent handler (uses `parseAuthRequest`, `lookupClient`, `completeAuthorization`) |
| `PUT/DELETE/OPTIONS /files/*` | Our upload handler (protected API route; token already validated by the library) |
| `GET/HEAD /files/*` | Public read from R2 with `ETag`, `Last-Modified`, `Content-Type`, CORS |
| `GET/POST /setup` | First-run owner registration (bootstrap secret) |
| `GET/POST /login`, `POST /logout` | Owner login |
| `GET /apps`, `POST /apps/revoke` | List grants, revoke |

### Upload handler

- Get the granted scopes from the library's auth context; reject with `403 insufficient_scope` unless one scope prefix covers the normalized path.
- Map `If-None-Match: *` and `If-Match` to R2 `put()` `onlyIf` conditions; a failed condition (`put` returns null) → `412`.
- Stream the body to R2; enforce `Content-Length` ≤ `MAX_UPLOAD_BYTES` before writing, reject chunked uploads without length or count bytes while streaming.
- Store `Content-Type` in `httpMetadata`; return the R2 ETag quoted.

### Owner login

Code against an interface `getOwner(request): Promise<Owner | null>` with two implementations:

1. **Built-in (default):** `/setup` accepts `BOOTSTRAP_SECRET` once, then registers a passkey (WebAuthn, e.g. `@simplewebauthn/server`) with an optional password fallback (PBKDF2/scrypt via WebCrypto). Session = signed, `HttpOnly; Secure; SameSite=Lax` cookie, 30 days, in subdomain mode only; in same-domain mode no session exists and every action is confirmed. After setup, `/setup` returns 404.
2. **Cloudflare Access (optional):** if `ACCESS_TEAM_DOMAIN` and `ACCESS_AUD` are set, trust the `Cf-Access-Jwt-Assertion` header after verifying it against the team's JWKS, and skip the built-in login. Only available in subdomain mode, because the Access cookie would otherwise be sent with requests made by scripts on the website.

### Distribution

- Public GitHub repo with README and a Deploy to Cloudflare button (`https://deploy.workers.cloudflare.com/?url=<repo>`); the button provisions `OAUTH_KV`, `OWNER_KV` and `FILES` from `wrangler.jsonc` and prompts for secrets.
- README covers: custom domain (requires DNS on Cloudflare), serving `/.well-known/host-meta.json` on the owner's main domain if the Worker runs on a subdomain, making the R2 bucket public or serving reads through the Worker, and how to pull upstream updates into the user's copy.
- Keep all user-specific settings in vars/secrets so updating is a clean merge.

## Phase 2: PHP drop-in

The PHP version implements the identical protocol with no framework and no Composer requirement at runtime, so it installs by uploading a folder to shared hosting. Start it only after the Phase 1 conformance suite exists, and run that suite against it.

### Constraints

- PHP 8.1+, PDO SQLite, `hash`, `openssl`/`random_bytes`; no other extensions assumed. Vendor any libraries (e.g. a single-file WebAuthn library such as lbuchs/WebAuthn) into the ZIP.
- Apache with `.htaccess` (most hosts) or LiteSpeed; document nginx rules separately.
- Installs into a subfolder, e.g. `/socialspec/`; all endpoints work from there, and discovery points to them.

### Layout

- `index.php` — single front controller, routes all requests.
- `.htaccess` — rewrite everything to `index.php`, pass `Authorization` through (`RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]`), deny direct access to `data/`.
- `data/` — SQLite database (`store.sqlite`) and config; moved outside the web root when the host allows it.
- `files/` — uploaded files, served directly by the web server for public reads.
- `install.php` — environment checks and first-run setup; locks itself after success.

### Behaviour that differs from the Worker

- **Authorization header:** read `HTTP_AUTHORIZATION`, then `REDIRECT_HTTP_AUTHORIZATION`, then `getallheaders()`. If none reaches PHP even with the `.htaccess` rule, the installer reports it and refuses to finish. Tokens are never accepted in the URL.
- **Blocked methods:** if the host blocks PUT or DELETE, the config declares `post_endpoint` (e.g. `https://tim.example/socialspec/post`) and writes use the POST fallback from section 6a. The installer detects this and sets it automatically.
- **ETags:** compute as a quoted SHA-256 of the file content (or size+mtime+inode hash for large files); use `flock()` on a lock file per path to make check-and-write atomic.
- **Atomic writes:** write to a temp file in the same directory, then `rename()`.
- **OAuth server:** implemented by hand (no library): codes, hashed tokens and grants in SQLite; CIMD fetch via cURL with private-IP blocking, 5 s timeout, 64 KB max.
- **Login:** password via `password_hash()` (Argon2id if available, else bcrypt) by default; passkey optional. Rate-limit failed logins in SQLite.
- **Updates:** the apps page checks a static JSON on GitHub for the latest version and shows a notice.

### Installer checks

`install.php` must report clearly and refuse to finish on: no HTTPS, PHP too old, no PDO SQLite, `data/` not writable, `Authorization` header not reaching PHP, response headers not settable via .htaccess (mod\_headers missing, so the sandbox CSP cannot be guaranteed). It also tests PUT and DELETE (enabling `post_endpoint` if they are blocked) and writes `/.well-known/host-meta.json` at the domain root; if that folder is not writable, it shows the exact file to upload by hand.

## Security requirements

These apply to both implementations and each must have a test.

- **HTTPS only.** Refuse to run or issue tokens over plain HTTP (loopback redirect URIs excepted).
- **Tokens:** at least 256 bits from a CSPRNG; store only SHA-256 hashes; constant-time comparison; never log tokens or put them in URLs.
- **PKCE:** S256 only; reject `plain` and requests without a challenge.
- **Codes:** single-use, 60 s lifetime; a reused code revokes every token issued from it.
- **Refresh tokens:** rotate on use; reuse of an old refresh token revokes the whole grant.
- **Redirect URIs:** exact string match against the CIMD document (loopback port excepted).
- **CIMD fetch (SSRF):** HTTPS only, block private, loopback and link-local addresses (including after DNS resolution), 5 s timeout, 64 KB limit, no redirects to other hosts.
- **Consent page:** CSRF token on the approve form; `X-Frame-Options: DENY` / `frame-ancestors 'none'` against clickjacking; show the client's real domain prominently, since `client_name` and the logo are self-declared.
- **Owner login:** rate-limit failures (e.g. 5 per 15 min, then backoff); session cookies (subdomain mode only) `HttpOnly; Secure; SameSite=Lax`.
- **Path safety:** normalize before scope checks; reject traversal; scope check on the normalized path only.
- **Content:** see 6b. Only data and media types are accepted, never HTML, JavaScript or server-executable files, and every file is served with the `Content-Type` of its extension, `X-Content-Type-Options: nosniff` and `Content-Security-Policy: sandbox`.
- **Step-up confirmation:** every grant of a well-known path scope and every revoke needs a fresh confirmation at that moment: a passkey assertion with user verification, or password re-entry where no passkey is registered. A valid session alone is not enough, so a script running on the site's origin cannot approve these silently. Grants limited to the client's own app directory, and `identify`, stay one click.
- **Cookie paths are not a boundary:** `Path` on the session cookie is hygiene only; same-origin scripts can still send requests that carry it. Isolation comes from the sandbox headers (6b) and step-up confirmation.
- **Optional auth subdomain (Worker only):** the OAuth and owner endpoints (`/oauth/*`, `/login`, `/setup`, `/apps`) can be placed on their own host, e.g. `id.tim.example`, while the website and files stay on the main domain. `host-meta.json` and the files stay where they are; only the config's `issuer` and endpoint URLs change. The session cookie is then host-only with the `__Host-` prefix on that host, and the browser's same-origin policy isolates owner pages from everything on the main domain. Same-domain mode is acceptable when the domain serves only static files; the subdomain is strongly recommended when the domain also runs web applications (CMS, wiki, comment forms, anything server-side), because an HTML-injection bug in any of them would otherwise run script with the owner's origin. Worker: a second custom domain on the same Worker, routed by host. PHP: not supported, since typical shared hosting offers no easy way to add subdomains; the PHP drop-in always runs in same-domain mode, and its installer warns that other web applications on the same domain put the owner session at risk. Step-up confirmation applies in both modes.
- **No cookies in same-domain mode:** when the owner pages share the website's origin, the server never sets a cookie (no session, no "remember me", no Cloudflare Access). Every owner action (approving an app, viewing `/apps`, revoking) is confirmed with password or passkey in the same request. Without an ambient credential, a script on the website has nothing to abuse for silent approvals. Only subdomain mode may keep an owner session.
- **What same-domain mode still does not prevent:** a page on the website can show a fake password prompt under the owner's real domain, or start a passkey prompt the owner might accept. Passkeys are strongly preferred over passwords here: a stolen password works forever, an abused passkey prompt only once and only in that moment. The sandbox headers (6b) therefore remain necessary.
- **No service workers from uploaded files:** a service worker needs a script, and uploads never include JavaScript. As defense in depth, uploaded files requested with the header `Service-Worker: script` are still refused with `403` (Worker: in the read handler; PHP: an `.htaccess` rewrite condition on that header).
- **Setup:** the bootstrap secret is single-use; `/setup` is unreachable after the owner exists.

## User flows

The owner's Cloudflare or hosting account is used only to deploy; the owner's identity toward apps is their own installation, with exactly one owner per install.

### First-run setup (owner, once)

1. **Worker:** click the Deploy to Cloudflare button, log in to Cloudflare, enter `BOOTSTRAP_SECRET`; attach a custom domain. **PHP:** upload the ZIP's folder, enable HTTPS at the hoster, open `install.php` and fix any reported problem.
2. Open `/setup`, enter the bootstrap secret, register a passkey (and optionally a password).
3. Confirm discovery works: the setup page fetches `/.well-known/host-meta.json` on the owner's domain and checks that it links to the config; if not, it shows the exact file to place there.

### Connecting an app (owner, per app)

1. In the app, type the domain (e.g. `tim.example`).
2. The browser lands on `/oauth/authorize` on the owner's domain; if not logged in, the owner authenticates with passkey or password.
3. The consent screen shows: app name and logo, the app's real domain in large type, "new app" if never seen before, a line saying the app will learn the owner's domain, each requested scope in plain words ("create, replace and delete files under /feeds/bigwide/"), checkboxes to drop individual scopes, and Approve / Deny.
4. On Approve, the browser returns to the app, which redeems the code and can upload immediately.
5. Later the app refreshes tokens silently; the owner is not asked again unless the app requests broader scopes.

### Managing apps (owner)

`/apps` lists each grant with app name, domain, scopes, date approved and last used; **Revoke** deletes the grant and all its tokens at once, after step-up confirmation. A **Sign out everywhere** button ends all owner sessions (subdomain mode only).

## Milestones, tests and open questions

### Milestones

1. **M1 — Upload core:** Worker with R2, public reads, PUT/DELETE with ETag preconditions, CORS, path rules; temporary static bearer token from a secret.
2. **M2 — OAuth:** workers-oauth-provider wired in, CIMD enabled, discovery document, scope enforcement replaces the static token.
3. **M3 — Owner:** `/setup` with bootstrap secret, passkey + password login, consent screen, `/apps` with revoke.
4. **M4 — Reference client:** a single static HTML page (hosted with its own CIMD document) that discovers, authorizes, uploads a file, updates it with `If-Match`, and shows a 412 conflict.
5. **M5 — Distribution:** deploy button, README, Cloudflare Access option, optional auth subdomain.
6. **M6 — PHP drop-in:** same conformance suite passing on PHP 8.1 Apache and on one real shared host.

### Conformance suite

A test runner (Vitest + a fake CIMD client) that takes a base URL and runs against either implementation. It must cover at least:

- [ ] host-meta.json links to the config; config fields present and endpoints reachable
- [ ] Full code + PKCE flow, including `iss` in the redirect
- [ ] Wrong `code_verifier`, reused code, expired code → rejected
- [ ] `redirect_uri` not in CIMD document, `client_id` mismatch, private-IP `client_id` → rejected
- [ ] Refresh rotation and reuse detection
- [ ] PUT inside scope succeeds; outside scope → 403; no/expired token → 401
- [ ] `If-None-Match: *` on existing file → 412; stale `If-Match` → 412; fresh `If-Match` → 200 with new ETag
- [ ] Traversal paths (`..`, encoded `%2e%2e`, backslashes) → 400
- [ ] Disallowed types (.php, .phtml, feed.php.json, .htaccess, HTML or JavaScript anywhere, any path outside /appdata/ and the declared well-known paths) → 415 or 403, nothing stored; sandbox headers present on every uploaded file; oversized upload → 413, nothing stored; all upload tests repeated through post\_endpoint when declared
- [ ] Preflight and all responses carry the required CORS headers, including on errors
- [ ] Revoked grant → all its tokens fail immediately; token response carries me matching the discovered domain; an identify-only token cannot write; same-domain mode sets no cookie on any response; uploaded files fetched with Service-Worker: script → 403; host-meta link add/remove leaves owner entries untouched, rejects foreign hrefs and reserved rels, retries on concurrent changes, refuses invalid JSON and enforces the limits; host-meta properties: only properties of a granted well-known path can be set, apps cannot remove properties, invalid values are rejected, other entries untouched

### Open questions

- Final link relation URI (`https://bigwide.net/ns/personal-storage` is a placeholder) and config path; to be published on bigwide.net.
- List of well-known files (exception 1 in 4a) and how each is described on the consent screen; to be defined on bigwide.net.
- Check during M2 that workers-oauth-provider v1 allows custom scope validation and exposes granted scopes to the API handler as assumed above; adjust if not. Same check for adding me to the token response (section 8).
