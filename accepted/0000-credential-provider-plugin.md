# Credential Provider Plugin Protocol for Secure NPM Authentication

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Detailed Explanation](#detailed-explanation)
  - [How it works (high level)](#how-it-works-high-level)
  - [Key components](#key-components)
- [Rationale and Alternatives](#rationale-and-alternatives)
- [Implementation](#implementation)
  - [Plugin Discovery](#plugin-discovery)
  - [Protocol](#protocol)
    - [Request kinds](#request-kinds)
    - [`get` request](#get-request)
    - [`get` success response](#get-success-response)
    - [`login` request](#login-request)
    - [`logout` request](#logout-request)
    - [Error response](#error-response)
    - [Timeout and retry](#timeout-and-retry)
  - [Caching](#caching)
  - [Example flows](#example-flows)
- [Security Considerations](#security-considerations)
- [Prior Art](#prior-art)
- [Unresolved Questions and Bikeshedding](#unresolved-questions-and-bikeshedding)
- [Acknowledgments](#acknowledgments)

## Summary

This RFC proposes the implementation of a credential provider plugin protocol
for the NPM CLI to enable secure, dynamic retrieval of authentication tokens.
The protocol allows external credential providers to supply short-lived tokens
at runtime, eliminating the need to store secrets in plaintext within `.npmrc`
files or environment variables.

## Motivation

Currently, NPM requires authentication tokens to be stored in plaintext within
`.npmrc` files or injected via environment variables.

This practice violates enterprise security policies and is a known vector for
exploitation.

NPM also lacks a native hook to automatically acquire or refresh short-lived
credentials. This discourages short token lifetimes (for example, <= 60 minutes)
because developers are forced into manual copy/paste rotation.

A standardized credential provider protocol fixes this at the tooling layer and
enables modern enterprise authentication workflows (brokered auth,
device-bound tokens, Multi-Factor Authentication (MFA), Conditional Access (CA),
and similar controls) without compromising developer experience.

### Goals

- Enable runtime token acquisition via a standardized provider interface.
- Support short-lived tokens without manual rotation.
- Support scoped registries and multiple registry configurations.
- Support third-party registry authentication using npm CLI and third-party
  credential providers.
- Preserve graceful fallback behavior if no provider is available.

### Non-Goals

- Mandate a single identity provider or authentication mechanism (protocol is
  Identity Provider (IdP)-agnostic).
- Define a universal keychain-based storage requirement for Node tooling
  (providers may use OS keychains or brokers internally).
- Persist returned tokens to disk (explicitly avoided).
- Reintroduce a pre/post-install script vector or enable execution of untrusted
  arbitrary binaries. This protocol is a narrowly scoped, user-controlled
  auth hook — not a general-purpose plugin system.


## Detailed Explanation

The proposed plugin protocol defines a standard interface for external
credential providers that can be invoked by the NPM CLI during authentication
workflows. When NPM needs credentials for a registry request (for example,
installing from or publishing to a private registry), the CLI invokes a
configured credential provider at runtime and receives a token in response.
This design avoids persisting tokens in `.npmrc` or relying on environment
variables as a long-term secret store. When a credential provider is
configured for a registry, that provider becomes the authoritative auth source
for that registry. If provider resolution or execution fails, npm may fall
back to legacy auth sources, but it must emit an explicit warning that auth
downgraded from credential provider mode.

### How it works (high level)

1. **Registry request requires auth**: NPM determines that a request to a
   registry requires authentication (install, publish, and similar operations).
2. **Plugin discovery**: NPM finds and resolves the ordered list of
   credential providers configured for the target registry. This happens once
   per registry per command, regardless of how many workspace members need
   that registry.
3. **Invoke provider**: NPM spawns the first provider as a child process,
   writes a JSON request to `stdin`, then closes the write end (sends EOF).
   The provider writes a JSON response to `stdout`, then exits. If the
   provider returns an error with kind `"url-not-supported"`, npm tries the
   next provider in the list. npm serializes provider invocations to avoid
   overlapping prompts and ambiguous-account errors.
4. **Use token in-memory**: NPM uses the returned token for the outgoing HTTP
   request without writing it to disk.
5. **Provider-owned token lifecycle**: NPM keeps a process-lifetime
   in-memory cache of provider responses (see [Caching](#caching)).
   The cache ensures the provider is typically spawned only once per
   registry per npm command. Token caching and
   refresh policy beyond the current process are owned by the provider.
6. **Failure handling**: If no provider is configured, NPM continues with
   existing credential behavior. If a provider is configured but fails during
   a `get` or `logout` request, NPM warns and falls back to legacy credential
   behavior. If a provider fails during a `login` request, NPM hard-fails —
   falling back would persist a plaintext token to `.npmrc`.

### Key components

#### Host (npm CLI)

- Decide when credential providers are invoked.
- Resolve the ordered provider list for the target registry.
- Try providers in configured order; advance on `"url-not-supported"` errors.
- Resolve and pass request context to the provider via `stdin` JSON.
- Declare the protocol version (`v`) in every request — no negotiation
  handshake.
- Serialize provider invocations (one at a time) — even in non-interactive
  mode — for deterministic fallback and clear account-ambiguity diagnostics.
- Use returned credentials in-memory only.
- Enforce retry limits and timeout; kill the provider on expiry.
- Never log credentials, tokens, or authorization headers.
- When provider execution fails on `get` or `logout`, fall back to legacy
  auth with an explicit warning. When provider execution fails on `login`,
  hard-fail — no fallback.

#### Plugin (credential provider)

- Acquire credentials for the target registry.
- Own token caching and refresh policy.
- Validate the declared protocol version (`v`) on every request — fail
  immediately if unsupported.
- Ignore unknown request fields (forward compatibility).
- Return credentials in a structured response shape.
- Return structured errors when credential acquisition fails.

## Rationale and Alternatives

1. **Plaintext `.npmrc` tokens** — Rejected. Tokens vulnerable to theft via
   malware or accidental exposure; violates enterprise security policies.
2. **Encrypt `.npmrc` tokens locally** — Rejected. Adds key management and
   cross-platform complexity; does not solve rotation or dynamic retrieval.
3. **Environment variables exclusively** — Rejected. Still violates secure
   storage policies; does not scale for short-lived tokens across dev machines
   and CI/CD.
4. **Direct command registration** — Not recommended as primary model.
   Providers configured as raw commands
   (e.g. `//<registryHost>:credentialProvider=<command>`). Drawbacks:
   - No integrity verification — npm trusts whatever the path resolves to.
   - No versioning or update detection.
   - Shell injection risk if command string passes through shell parsing.
   - No discoverability or ecosystem conventions.
5. **Long-lived bidirectional process** — Considered. Provider stays alive
   for the duration of the npm command; npm sends multiple requests on the
   same stdin/stdout stream. Amortizes startup cost and enables richer
   protocol features (refresh, batch). Not recommended for v1:
   - In-memory cache already limits spawns to 1-2 per registry per command.
   - Adds process lifecycle management (crash detection, reconnection,
     graceful shutdown) to both npm and every provider implementation.
   - Requires message framing (length-prefix or newline-delimited JSON) —
     one-shot uses EOF as the natural delimiter.
   - Each request is an isolated process — no corrupted state carries over
     from a previous failure.
   - Provider implementation reduces to: read stdin, write stdout, exit.
   - Git credential helpers and NuGet credential providers both use one-shot
     successfully at massive scale.

The plugin protocol is the most secure and flexible option, aligning with
prior art (pnpm tokenHelpers, NuGet credential providers, pip keyring,
Git credential helpers, Cargo credential providers).

## Implementation

### Plugin Discovery

#### Configuration

The user or global `.npmrc` stores provider identifiers, not shell commands. The value is an
ordered, comma-separated list of provider ids:

- `//<registryHost>:credentialProvider=<id>[,<id>...]` (per-registry)
- `@scope:credentialProvider=<id>[,<id>...]` (per-scope)
- `credentialProvider=<id>[,<id>...]` (global default)

Precedence: per-registry > per-scope > global default.

Only user or global `.npmrc` grants execution trust. `credentialProvider`
in project `.npmrc` is ignored entirely.

npm tries providers in the order listed. When a provider returns an `Err`
with kind `"url-not-supported"`, npm advances to the next provider. If all
providers return `"url-not-supported"`, npm falls back to legacy auth and
emits a warning. npm does not cache which provider succeeded for a given
registry; providers are tried in order on every npm command. This
matches Cargo's credential provider model. Provider-side token caching
ensures the successful provider returns near-instantly on subsequent
invocations, so the cost of re-trying the list is negligible in practice.

> **Design note:** NuGet takes a different approach — it caches a
> per-registry mapping of which provider last succeeded, so subsequent
> commands skip straight to the winning provider. This avoids re-trying the
> list but adds host-side state management. The Cargo model is simpler and
> avoids staleness issues when provider configurations change.

Example:

```ini
# ~/.npmrc

# Global default — applies to all registries unless overridden
credentialProvider=@corp/credprovider,@backup/credprovider

# Per-registry — overrides global default for this registry
//registry.example.com:credentialProvider=@corp/credprovider
//registry.example.com:credentialProviderAccountHint=user@corp.com

# Per-scope — overrides global default for @corp packages
@corp:credentialProvider=@corp/credprovider
```

#### Provider Binary Resolution

The provider reference in user/global `.npmrc` must be a provider package
name resolved from npm-managed global install locations.

npm must never resolve providers from `$PATH` or project `node_modules`.

Adding the provider to user/global `.npmrc` **is** the trust decision — the
same model as Git and Docker credential helpers.

The provider must be installable from a trusted source that does not itself
require the provider (public registry, local package source with global
install, or enterprise bootstrap script).

### Protocol

Providers are executed as a child process using a `stdin`/`stdout` JSON
protocol:

- npm spawns the provider executable with no command-line arguments and
  no shell interpolation.
- npm writes exactly one UTF-8 JSON request to `stdin`, then closes the write
  end (sends EOF).
- The provider writes exactly one UTF-8 JSON response to `stdout`, then
  exits.
- `stderr` is reserved for diagnostic logging — npm streams it to the user
  per loglevel but never parses it.
- Exit code 0 + recognized `Ok` response = success. Non-zero exit or `Err`
  response = failure.
- npm enforces a per-request-kind timeout (see [Timeout and retry](#timeout-and-retry)).
  On expiry, npm kills the process and treats it as failure. Configurable via
  `credentialProviderTimeoutMs` in user/global `.npmrc`.

This follows [Cargo's credential provider protocol](https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html)
— structured communication via `stdin`/`stdout`, `stderr` for diagnostics —
adapted to a one-shot spawn model similar to NuGet credential providers and
Git credential helpers.

#### Request kinds

| Kind | Required | Purpose |
|------|----------|---------|
| `get` | yes | Acquire credentials for a registry request (may be interactive or silent) |
| `login` | no | Force fresh authentication and persist — hooks into `npm login` |
| `logout` | no | Remove stored credentials — hooks into `npm logout` |

Providers must implement `get`. `login` and `logout` are optional — a provider
that does not support them must return `operation-not-supported`, and npm will
fall back to its default login/logout behavior (prompting for a token and
persisting it to `.npmrc`).

#### `get` request

```json
{
  "v": 1,
  "kind": "get",
  "registry": "https://pkgs.dev.azure.com/org/_packaging/feed/npm/registry/",
  "permission": "read-only",
  "accountHint": "user@example.com",
  "interactive": true,
  "retry": false,
  "logLevel": "info",
  "network": {
    "httpsProxy": "https://proxy.example.com:8080",
    "httpProxy": "http://proxy.example.com:8080",
    "noProxy": "localhost,127.0.0.1",
    "strictSsl": true,
    "caFile": "/path/to/ca-bundle.crt"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `v` | integer | yes | Protocol version. npm declares in every request — no handshake. Major version bump = breaking changes; new optional fields don't require a bump. Provider does not echo the version back. |
| `kind` | string | yes | Must be `"get"`. |
| `registry` | string | yes | Fully qualified absolute base URI. Resolved from the nerf-darted config key where the provider was found. |
| `permission` | string | yes | `"read-only"` (install, view, search) or `"read-write"` (publish, unpublish). Follows npm's existing token permission vocabulary (`npm token create --packages-and-scopes-permission`). Providers that don't support permission-scoped tokens may ignore this and return a token valid for both. |
| `accountHint` | string | no | Which account to use when multiple are available for the same registry. See below. |
| `interactive` | boolean | yes | `true` by default; `false` only when user passes `--no-interactive`. See below. |
| `retry` | boolean | yes | `true` on retry after registry auth failure (401/403). Provider should bypass its cache and re-acquire. |
| `logLevel` | string | yes | Current npm log verbosity (from `--loglevel` or npm config). Providers may use to control diagnostic output. |
| `network` | object | no | Proxy and TLS settings from npm config. See below. |

**accountHint resolution:** Sourced exclusively from the config key
`credentialProviderAccountHint` in user/global `.npmrc`
(e.g. `//<registryHost>:credentialProviderAccountHint=user@corp.com`).
This is a new optional, explicit config entry that users/admins add manually;
npm does not create it automatically and does not accept `--accountHint` CLI flags.
If the config key is absent, the field is omitted.
npm never writes `accountHint` to `.npmrc` — the config key is
user-managed only. Providers may ignore the hint.

**interactive semantics and `--no-interactive`:**

- npm defaults `interactive` to `true` on all commands.
- `--no-interactive` flag explicitly sets it to `false`. Available on any
  command that may invoke a provider (`install`, `publish`, `view`,
  `login`, etc.).
- **Why default `true`:** Eliminates the manual login-then-retry workflow.
  When a token expires mid-`npm install`, the provider can re-authenticate
  (device code, MFA, broker prompt) without forcing the user to abort, run
  `npm login`, and restart. Cargo's model — where expired tokens require a
  manual `cargo login` — is the UX gap we are closing.
- **Why not infer from TTY/CI:** TTY detection is unreliable (piped
  terminals, Docker, SSH, IDE terminals produce false negatives). CI env
  var sniffing adds heuristic complexity with no clear standard. The user
  knows their context — `--no-interactive` is the explicit opt-out.
- **CI guidance:** CI pipelines should pass the new `--no-interactive` flag
  (or its env var equivalent `npm_config_no_interactive=true`, per npm's
  standard config-to-env mapping). This is consistent with npm's existing
  CI guidance — use `npm ci` instead of `npm install`, etc. — where CI
  environments are expected to opt in to stricter, non-interactive behavior
  explicitly.
- **When `true`:** Provider may block for user interaction (browsers,
  device codes, MFA). Must not read from `stdin` — it belongs to the
  protocol stream.
- **When `false`:** Provider must not block waiting for user interaction.
  If it cannot acquire credentials silently, it must return an error.

**network fields:** `httpsProxy`, `httpProxy`, `noProxy`, `strictSsl`,
`caFile`. Absent fields mean "not configured." Forwarded because providers
make their own outbound HTTP requests (e.g. to an identity provider) and
cannot inherit these settings from the environment — npm's `cafile` and
`strict-ssl` are npm-specific config, not Node.js TLS defaults, and proxy
settings may differ from environment variables.

Providers must ignore unknown fields for forward compatibility.

npm expects providers to support the protocol version npm declares (`v`).
If a provider does not support the declared version, it must respond with an
`Err` of kind `"operation-not-supported"` with an appropriate message. npm will
fail with actionable guidance and must not retry with a lower version.

#### `get` success response

```json
{
  "Ok": {
    "kind": "get",
    "auth": {
      "type": "bearer",
      "token": "<token>"
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `Ok.kind` | string | yes | Must be `"get"`. |
| `Ok.auth` | object | yes | Credentials. See Auth Types below. |

The `auth` object uses an explicit `type` discriminator field:

**Bearer token** — npm sends `Authorization: Bearer <token>`:
```json
{
  "type": "bearer",
  "token": "xxxxxxxxxxxx"
}
```

**Basic auth** — npm encodes `username:password` to base64 and sends
`Authorization: Basic <base64>`:
```json
{
  "type": "basic",
  "username": "someusername",
  "password": "xxxxxxxxxxxx"
}
```

Provider returns plain values; npm handles encoding.

#### `login` request

When a credential provider is configured for a registry, `npm login`
delegates to the provider rather than performing its default behavior
(persisting tokens in `.npmrc`).

```json
{
  "v": 1,
  "kind": "login",
  "registry": "https://registry.example.com/",
  "accountHint": "user@example.com",
  "interactive": true,
  "logLevel": "info",
  "network": {
    "httpsProxy": "https://proxy.example.com:8080",
    "strictSsl": true
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `v` | integer | yes | Protocol version. |
| `kind` | string | yes | Must be `"login"`. |
| `registry` | string | yes | Target registry base URI. |
| `accountHint` | string | no | From `.npmrc` config (`credentialProviderAccountHint`). |
| `interactive` | boolean | yes | `true` by default; `false` with `--no-interactive`. |
| `logLevel` | string | yes | Current npm log verbosity. |
| `network` | object | no | Proxy and TLS settings (same shape as `get`). Needed for outbound requests to identity providers. |

`login` reuses the `get` request context with two intentional differences:
`retry` is always `true` (to force fresh authentication), and `permission` is
omitted (login establishes identity/session state; any persisted permission
scope is provider-defined). The provider executes the authentication flow
(OAuth, device code, SSO, MFA, or equivalent) and persists resulting
credentials in provider-managed secure storage (for example, OS keychain,
encrypted file, or broker cache). npm does not receive or persist credential
material during `login`; this operation is provider-managed end to end.

`login` receives the same context fields as `get` (`network`, `logLevel`,
`interactive`) because the provider makes the same outbound HTTP requests to
identity providers. `--no-interactive` on `npm login` passes
`interactive: false` — the provider should fail if it cannot authenticate
silently (e.g. no cached refresh token available).

Response on success:

```json
{
  "Ok": {
    "kind": "login"
  }
}
```

npm does not persist anything on login success — the provider owns
credential storage entirely. If the user wants `accountHint` passed on
subsequent commands, they configure it manually in `.npmrc`.

This matches Cargo's `login` request kind where `npm login --registry <url>`
delegates entirely to the provider, enabling modern auth flows (OAuth, SSO,
device code) that `npm login` cannot handle natively.

#### `logout` request

When a credential provider is configured for a registry, `npm logout`
delegates to the provider rather than removing `.npmrc` token entries.

```json
{
  "v": 1,
  "kind": "logout",
  "registry": "https://registry.example.com/",
  "logLevel": "info",
  "network": {
    "httpsProxy": "https://proxy.example.com:8080",
    "strictSsl": true
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `v` | integer | yes | Protocol version. |
| `kind` | string | yes | Must be `"logout"`. |
| `registry` | string | yes | Target registry base URI. |
| `logLevel` | string | yes | Current npm log verbosity. |
| `network` | object | no | Proxy and TLS settings (same shape as `get`). Optional; omitted if no proxy/TLS config is set. May be used for best-effort server-side token revocation. |

The provider removes stored credentials for that registry from its secure
storage. Providers may optionally attempt server-side token revocation (e.g. `POST /oauth/revoke`)
using the `network` context, but must not block logout on network failure — the user's intent
is to remove local credentials, and that must always succeed.

Response on success:

```json
{
  "Ok": {
    "kind": "logout"
  }
}
```

#### Behavior changes for `login` and `logout`

- `npm login` must not persist tokens to `.npmrc` when a provider is
  configured and returns `Ok`. The provider owns credential storage.
- `npm logout` must not remove `.npmrc` token entries when a provider is
  configured and returns `Ok`. It delegates to the provider.
- If no provider is configured, `npm login` and `npm logout` behave as today
  with no warning.
- If a provider is configured but returns `operation-not-supported` for the
  request kind, `npm login` and `npm logout` fall back to default behavior and
  must emit an explicit warning that the operation downgraded from credential
  provider mode.
- If a provider is configured and fails during `login` (error kind `"other"`,
  timeout, or non-zero exit), `npm login` must hard-fail — no fallback.
  Falling back would persist a plaintext token to `.npmrc`, which is the exact
  security regression this RFC exists to prevent.
- If a provider is configured and fails during `logout` (error kind `"other"`,
  timeout, or non-zero exit), `npm logout` falls back to removing the
  `.npmrc` token entry and must emit an explicit warning. Logout failure
  does not create a security hole — the fallback ensures local credentials
  are still cleared.

#### Error response

On failure, the provider responds with an `Err` object:

```json
{
  "Err": {
    "kind": "operation-not-supported",
    "message": "Protocol version 2 is not supported by this provider."
  }
}
```

| Error kind | Meaning | npm behavior |
|------------|---------|--------------|
| `"url-not-supported"` | Provider does not handle this registry. | Skip to next provider in the configured list. |
| `"operation-not-supported"` | Provider does not support this request kind. | `get`/`logout`: fall back to legacy auth with warning. `login`: fall back with warning. |
| `"other"` | Generic error. | `get`/`logout`: fall back to legacy auth with warning. `login`: hard-fail — no fallback. |

Fields:
- `kind` (string, required): Error category.
- `message` (string, required): Human-readable, suitable for display. Must not
  contain tokens, passwords, or PII.

#### Timeout and retry

**Per-request timeouts:**

All request kinds use a uniform timeout across request kinds. The default is
120 seconds; this accommodates interactive flows (device code, MFA) that may
occur even during `get` requests. If a provider exceeds the timeout, npm kills
the process and treats it as failure. Users can override the default with
`credentialProviderTimeoutMs` in user/global `.npmrc` (per-registry or global).

**Auth failure retry flow:**

When npm receives HTTP 401, 403, or a similar auth failure from a registry:

1. npm evicts the in-memory cached credential for that registry.
2. npm spawns the provider with a new `get` request with `retry: true`.
   The provider should invalidate its cached credentials and re-acquire.
3. If the provider returns new credentials, npm retries the failed registry
   request.
4. If the registry rejects again, npm fails the command — no further retries.

Retrying on both 401 and 403 improves on NuGet's credential provider
protocol, which only retries on 401 and misses Conditional Access step-up
scenarios that surface as 403.

**Retry limits:**
- Max 1 retry per auth failure per registry.

### Caching

- The credential provider owns long-lived token cache and refresh policy.
- npm must not implement long-lived caching — the CLI keeps a
  **process-lifetime in-memory cache** only.
- Cache key: provider id + registry base URI + permission.
- npm must not persist tokens to disk.
- On auth failure, npm evicts the in-memory cache for that registry
  and re-spawns the provider with `retry: true`, signaling it to bypass its
  own cache.

### Example flows

#### Desktop: login then use

Setup (once):
```sh
npm install -g @corp/credprovider  # install provider from public registry or local source
```

User `.npmrc` (`~/.npmrc`):
```ini
//registry.example.com:credentialProvider=@corp/credprovider
```

Project `.npmrc` (checked in with the repo):
```ini
registry=https://registry.example.com/
```

```sh
# optional
$ npm login --registry https://registry.example.com
# Provider spawned with login request — opens full auth flow (browser, device code, SSO)
# Provider stores credentials in OS keychain; npm persists nothing to .npmrc

$ npm publish
# npm resolves registry from project .npmrc; credential provider resolved from user .npmrc
# Provider spawned with get request — returns cached token instantly from keychain. If token is not cached or needs to be refreshed, provider spawned again with login request — opens full auth flow (browser, device code, SSO)
```

#### CI: managed identity / workload identity

Pipeline setup (once, in image or bootstrap step):
```sh
npm install -g @corp/credprovider
```

Pipeline `.npmrc`:
```ini
//registry.example.com:credentialProvider=@corp/credprovider
```

Pipeline run:
```sh
# Optional: some providers may require an explicit login step (e.g. service principal
# registration). Providers using ambient identity (managed identity, workload
# identity federation) could skip this.
$ npm login --registry https://registry.example.com --no-interactive

$ npm ci --no-interactive
# Provider acquires token silently via ambient or pre-authenticated identity
# Token used in-memory — nothing written to disk
# If silent acquisition fails, npm hard-fails with a diagnostic
```

## Security Considerations

#### Threat model

| Threat | Attack vector | Defense |
|--------|--------------|---------|
| Binary substitution at rest | Malware, compromised install script, shared workstation | Global prefix resolution + OS file permissions |
| Malicious lifecycle scripts | `postinstall` overwrites provider | OS file permissions + opt-in install scripts |
| PATH poisoning | Malicious `.env`, shell profile, CI manipulation | Global prefix resolution (no `$PATH` search) |
| Typosquatting | Similarly-named package on public registry | User/global `.npmrc` is the trust boundary |
| Malicious project `.npmrc` | Repo plants rogue provider config | Project `.npmrc` `credentialProvider` ignored |
| Credential exfiltration via lifecycle scripts | `postinstall` spawns provider binary or `npm`/`npx` to acquire token | Opt-in install scripts |

#### Process memory

Node.js (and any user-space process) holds secrets in plaintext memory while
running. An attacker with sufficient access to read process memory (debugger
attach, `/proc/<pid>/mem`, memory dump) can extract tokens regardless of how
they were acquired. This RFC does not attempt to defend against that vector —
it is an OS-level concern shared by every credential-bearing process. The
goal is to eliminate **persistent** plaintext storage (`.npmrc` files,
environment variables, shell history) that survives beyond the lifetime of a
single command.

## Prior Art

- **pnpm tokenHelpers** — pnpm supports `tokenHelpers` in `.npmrc` that invoke
  external commands to retrieve tokens. Similar concept but uses raw command
  strings without integrity verification.
- **NuGet credential providers** — .NET ecosystem uses a plugin protocol for
  credential acquisition with structured JSON communication over stdio.
  One-shot model with version declared per-request.
- **pip keyring** — Python's pip delegates credential storage/retrieval to the
  system keyring via a plugin interface.
- **Git credential helpers** — Git invokes configured helpers via stdio to
  acquire credentials for remote operations. Actions (get, store, erase)
  passed as CLI arguments. One-shot spawn model.
- **Cargo credential providers** — Rust's Cargo uses a stdin/stdout JSON
  protocol for credential provider plugins. Long-lived process, version
  hello on startup, multiple request kinds (get, login, logout). This design
  adapts Cargo's message shapes and request kinds to a one-shot spawn model.

## Unresolved Questions and Bikeshedding

- **Enterprise-managed path installs:** This RFC requires provider package
  names resolved from npm-managed global install locations and does not allow
  absolute executable paths. Is global install from local package sources
  sufficient for enterprise deployment needs, or should we add a future mode
  for enterprise-managed absolute-path providers? If so, what trust and
  integrity constraints would be required to avoid binary substitution risk?
- **Lockfile-derived trust for project-local providers:** Can we safely enable
  project/workspace-level credential provider configuration if
  `package-lock.json` SRI hashes enable project-local provider resolution
  (from `node_modules`) without the global install requirement? This would
  solve the security problem — the lockfile pins the provider's integrity
  before install, so a malicious project `.npmrc` cannot point to an
  arbitrary binary. However, it creates a chicken-and-egg problem: the
  provider must already be installed to authenticate to the registry, but
  installing the provider requires authenticating to the registry. Solving
  this would require `.npmrc` to support a separate unauthenticated registry configurations for
  credential provider installation — significant complexity for v1.
  Not recommended for v1.
- **`authChallenges` and `httpStatus`:** this design does not
  forward `WWW-Authenticate` header values or the HTTP status code to the
  provider. On retry, providers receive only `retry: true` and must do their
  best re-acquisition (silent refresh, broker call, etc.). If the new token
  is still rejected, npm fails. This is sufficient for the common case
  (expired tokens). Richer challenge forwarding and status codes can be added.

## Acknowledgments

This RFC builds on the original credential provider RFC contribution by
[@pwoosam](https://github.com/pwoosam)
([npm/rfcs#850](https://github.com/npm/rfcs/pull/850)), which established the
core concept of a stdio-based credential provider plugin for npm, including
provider discovery via `.npmrc`, the one-shot spawn model, structured JSON
request/response protocol, and a security-focused auth posture. That PR
serves as the foundation for this work.

The protocol design in this RFC draws extensively from the
[unofficial-npm-credential-provider-rfc](https://github.com/Filyus/unofficial-npm-credential-provider-rfc)
draft, which proposed a bidirectional JSON protocol with version negotiation,
structured error kinds (`url-not-supported`, `not-found`,
`operation-not-supported`, `other`), explicit request kinds (get, login,
logout, refresh, erase, get-batch), cache control fields
(`cache`, `expiresAt`, `operationIndependent`, `granularity`), the `erase`
request for credential rejection notification, retry context (`retry`,
`httpStatus`, `authChallenges`), the `interactive` field for CI/non-interactive
behavior, provider chaining via `url-not-supported`, per-request timeouts by
kind, and a comprehensive security threat model. The protocol shapes and
error taxonomy in this RFC are directly adapted from that draft.


Additional design influence from
[Cargo's credential provider protocol](https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html),
which established the pattern of structured JSON messages on stdin/stdout,
stderr for diagnostics, and request kinds (get, login, logout) for credential
provider plugins.

