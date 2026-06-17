# Credential Provider Plugin Protocol for Secure NPM Authentication

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
- Support third-party registry authentication using npm cli and third-party
  credential providers.
- Preserve graceful fallback behavior if no provider is available.

### Non-Goals

- Mandate a single identity provider or authentication mechanism (protocol is
  Identity Provider (IdP)-agnostic).
- Define a universal keychain-based storage requirement for Node tooling
  (providers may use OS keychains or brokers internally).
- Persist returned tokens to disk (explicitly avoided).

## Detailed Explanation

The proposed plugin protocol defines a standard interface for external
credential providers that can be invoked by the NPM CLI during authentication
workflows. When NPM needs credentials for a registry request (for example,
installing from or publishing to a private registry), the CLI invokes a
configured credential provider at runtime and receives a token in response.
This design avoids persisting tokens in `.npmrc` or relying on environment
variables as a long-term secret store. When a credential provider is
configured for a registry, that provider becomes the authoritative auth source
for that registry. npm must fail closed on provider errors and must not silently
downgrade back to legacy auth sources.

### How it works (high level)

1. **Registry request requires auth**: NPM determines that a request to a
   registry requires authentication (install, publish, and similar operations).
2. **Plugin discovery**: NPM finds, resolves, and performs integrity checks for
   the credential provider configured for the target registry.
3. **Invoke provider**: NPM spawns the provider as a child process, writes a
   JSON request with relevant runtime context information to `stdin`,
   and reads a JSON response from `stdout`. When
   multiple registries require credentials, npm serializes provider
   invocations (one at a time) to avoid overlapping interactive prompts.
4. **Use token in-memory**: NPM uses the returned token for the outgoing HTTP
   request without writing it to disk.
5. **Provider-owned token lifecycle**: NPM keeps a process-lifetime
   in-memory cache of provider responses (see [Caching](#caching)),
   while token caching and refresh policy are owned by the credential provider.
6. **Failure handling**: If no provider is configured, NPM continues with
   existing credential behavior. If a provider is configured but fails, NPM
   surfaces the provider error and fails the npm command for that auth context;
   it must not fall back to legacy sources for that registry.

### Key components

#### Host (npm CLI)

- Decide when credential providers are invoked.
- Resolve the provider for the target registry.
- Resolve configuration and pass it to the provider via `stdin` JSON.
- Declare the protocol version (`v`) in every request — no negotiation handshake.
- Serialize provider invocations per registry (one at a time).
- Use returned credentials in-memory only.
- Enforce retry, timeout, redirect, and logging policy.
- Make available appropriate logging/tracing messages for use and support efforts.
- Never fall back to legacy auth when a provider is configured.

#### Plugin (credential provider)

- Acquire credentials for the target registry.
- Own token caching and refresh policy.
- Validate the declared protocol version (`v`) on every request — fail
  immediately if unsupported.
- Handle account selection; fail safely when selection is ambiguous.
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

   May be appropriate for local development/testing but not production.

The plugin protocol is the most secure and flexible option, aligning with
prior art (pnpm tokenHelpers, NuGet credential providers, pip keyring,
Git credential helpers, Cargo credential providers).

## Implementation

### Plugin Discovery

#### Provider selection via `.npmrc`

`.npmrc` stores provider identifiers, not shell commands. Provider selection
by id:

- `credentialProviderId=<id>` (global default)
- `//<registryHost>:credentialProviderId=<id>` (per-registry)
- `@scope:credentialProviderId=<id>` (per-scope)

Resolution precedence: per-registry > per-scope > global default. Scope-level
configuration is useful when all packages under a scope resolve to the same
private registry.

Example:

```ini
//registry.example.com:credentialProviderId=corp-provider
@corp:credentialProviderId=corp-provider
```

All `credentialProvider*` nerf-darted keys must be added to npm's allow-list
(`nerfDarts` in `workspaces/config/lib/definitions/index.js`) so npm does not
emit "unknown config" warnings.

#### Provider id to executable resolution

The provider id is the npm package name. Credential providers expose a CLI
executable via `bin` in their `package.json`. npm resolves the executable via
the path + SHA-256 hash recorded during `npm credential-provider trust`.

The resolved executable must pass a hash integrity check before every spawn.

On Windows, globally installed npm packages create `.cmd` shims in the global
prefix bin directory (e.g. `%APPDATA%\npm\<provider-bin-name>.cmd`), which
npm uses as the executable path.

This resolution logic is **new behavior** specific to credential providers.
npm's existing package resolution does not perform hash-based integrity checks.
This is intentional — credential providers occupy a higher trust tier because
they handle authentication secrets.

#### Installation and registration

Installation and registration are distinct steps:

1. **Installation** (`npm install -g <id>` or `npm install <id>`): Places the
   package on disk and creates the `bin` shim. Installation alone does **not**
   grant trust.

2. **Registration** — a new `npm credential-provider trust <id>` subcommand
   (does not exist yet; prerequisite for shipping this feature):
   - Resolves the installed provider's `bin` entry to an absolute path.
   - Computes SHA-256 of the resolved executable.
   - Displays path, version, and hash for user confirmation.
   - Records the path + hash in user/global npm config.
   - This is the only mechanism that grants spawn trust.

   Re-registration is required after every provider update — intentional
   friction. In npm's ecosystem, silent upgrades are exactly how supply-chain
   attacks land (maintainer account takeover → patch release →
   auto-installed). Requiring an explicit trust step makes this detectable.

**Bootstrap constraint:** The provider must be installable from a source that
does not itself require the provider for authentication. This is not a
limitation — it's the correct trust boundary, matching every credential helper
ecosystem (Git, Docker, Cargo). Recommended distribution paths:
- Publish to the public npm registry (npmjs.org).
- Provide an OS-native installer (MSI, pkg, deb).
- Enterprise bootstrap script that installs from an unauthenticated endpoint.

Example:

```bash
npm install -g @corp/credprovider --registry https://registry.npmjs.org
npm credential-provider trust @corp/credprovider
```

```ini
//registry.example.com:credentialProviderId=@corp/credprovider
```

**Automatic discovery (rejected):** Scanning well-known directories (similar
to [.NET global tools](https://learn.microsoft.com/dotnet/core/tools/dotnet-tool-install#installation-locations))
was rejected — writable discovery paths become planting vectors, spawning
unknown binaries violates fail-closed, and hash integrity requires a deliberate
registration step that auto-discovery bypasses.

### Invoking the Credential Provider

When npm requires credentials for a registry request:

1. Provider invocation, if a provider is configured for the target registry.
2. Existing credential sources only when no provider is configured.

#### Protocol

Providers are executed as a child process using a `stdin`/`stdout` JSON
protocol:

- npm spawns the registered provider executable with no command-line arguments.
  On Unix, spawned directly without a shell. On Windows, `.cmd` shims require
  shell-based spawning.
- npm writes exactly one UTF-8 JSON request to `stdin`, then closes the write
  end (sends EOF).
- The provider writes exactly one UTF-8 JSON response to `stdout`.
- `stderr` is reserved for diagnostic logging — npm streams it to the user
  per loglevel but never parses it.
- Exit code 0 + recognized credential fields = success. Non-zero exit =
  failure (JSON body carries error details if valid).
- npm must not fall back to `.npmrc`, `NPM_TOKEN`, or other legacy auth when a
  provider is configured. Provider failure = npm command failure.
- When multiple registries need credentials, npm serializes invocations to
  prevent overlapping interactive prompts.

This follows [Cargo's credential provider protocol](https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html)
— structured communication via `stdin`/`stdout`, `stderr` for diagnostics.

#### Configuration passed to providers

npm resolves all configuration so providers do not have to parse `.npmrc`
files or read npm environment variables. Settings are passed in the request
JSON:

| npm config key | JSON field | Description |
|----------------|-----------|-------------|
| `registry` | `registry` | **Required.** Fully qualified registry base URI. |
| `loglevel` | `loglevel` | Current npm log verbosity. |
| `https-proxy` | `network.httpsProxy` | HTTPS proxy URL. |
| `proxy` | `network.httpProxy` | HTTP proxy URL. |
| `noproxy` | `network.noProxy` | Comma-separated bypass domains. |
| `strict-ssl` | `network.strictSsl` | Whether to require SSL cert validation. |
| `cafile` | `network.caFile` | Path to CA certificate bundle file. |

Proxy and TLS settings are forwarded because providers may make their own
outbound HTTP requests (e.g. to an identity provider) and should honor the
same network configuration as npm.

**Credential-provider-specific settings:**

| Config key | JSON field | Description |
|------------|-----------|-------------|
| `credentialProviderInteractive` | `interactive` | Whether prompts are allowed. `true` (default) or `false` for CI. |
| `credentialProviderTimeoutMs` | *(not passed)* | Max runtime in ms (default `120000`). Enforced externally by npm. |
| *(set by npm on retry)* | `forceRefresh` | Boolean signal to bypass cache. Defaults to `false`. |

#### Request schema

```json
{
  "v": 1,
  "registry": "https://pkgs.dev.azure.com/org/_packaging/feed/npm/registry/",
  "interactive": true,
  "loglevel": "info",
  "forceRefresh": false,
  "network": {
    "httpsProxy": "https://proxy.example.com:8080",
    "httpProxy": "http://proxy.example.com:8080",
    "noProxy": "localhost,127.0.0.1",
    "strictSsl": true,
    "caFile": "/path/to/ca-bundle.crt"
  }
}
```

Key field semantics:

- `v`: Integer protocol version. npm declares in every request — no
  handshake. A **major** version bump signals breaking changes; if the
  provider doesn't support the declared version, it must error with
  `retryable: false`. npm fails with actionable guidance (e.g. "update the
  provider or downgrade npm") and never retries with a lower version.
  Backward-compatible additions (new optional fields) don't require a bump.
  The provider does not echo the version back.
- `registry`: Fully qualified absolute base URI. Resolved from the
  nerf-darted config key where the provider was found. Always required.
- `interactive`: Boolean. CI environments set `false`.
- `forceRefresh`: npm sets `true` only on retry after auth failure.

Providers must ignore unknown fields for forward compatibility.

#### Successful response

The response contains a top-level `account` field and a `credentials` object
with an explicit `type` discriminator:

```json
{
  "account": "user@example.com",
  "credentials": {
    "type": "bearer",
    "token": "<token>"
  }
}
```

```json
{
  "account": "user@example.com",
  "credentials": {
    "type": "basic",
    "username": "<username>",
    "password": "<password>"
  }
}
```

| `credentials.type` | Required fields | HTTP header applied |
|--------------------|-----------------|---------------------|
| `"bearer"` | `token` | `Authorization: Bearer <token>` |
| `"basic"` | `username`, `password` | `Authorization: Basic base64(username:password)` |

Validation:
- Missing or unrecognized `credentials.type` = protocol error.
- Missing required fields for the declared type = protocol error.
- npm ignores unknown fields within `credentials` (forward compatibility).
- Providers should prefer bearer unless the registry requires basic.

#### Error response

On failure, the provider exits non-zero. If valid JSON is on `stdout`, npm
reads two fields — the entire error contract:

```json
{
  "message": "Interactive login is required but interactive mode is disabled.",
  "retryable": false
}
```

- `retryable` (boolean, required): `true` = transient (network timeout, cache
  lock) — npm may re-invoke. `false` = terminal (wrong mode, insufficient
  permissions) — no retry.
- `message` (string, required): Human-readable, suitable for display. Must not
  contain tokens, passwords, or PII.

npm must redact credential-shaped values before writing logs. Bounded output
limits (64 KiB stdout, 256 KiB stderr) — exceeded = kill provider.

#### Timeout and retry

**Timeout:** Default 120s (configurable via `credentialProviderTimeoutMs`).
On expiry, npm kills the provider and treats it as failure.

**Auth failure retry flow:**

npm should not interpret HTTP status codes to decide whether retry is
worthwhile — the provider owns the identity layer and can distinguish a
permanent 403 from a transient Conditional Access step-up.

On auth failure (401, 403, or similar):

1. npm re-invokes the provider with `forceRefresh: true`.
2. The provider either returns new credentials (e.g. after step-up auth) or
   a terminal error with `retryable: false`.
3. npm replays the registry request with new credentials.
4. If rejected again, npm fails without further retries.

This gives providers full control over token refresh, CA step-up (MFA,
device compliance), claims challenges, and account disambiguation.

**Retry limits:**
- Max 1 re-invocation per auth failure.
- Max 3 total invocations per registry per npm command (initial + up to 2
  retries).

### Caching

- The credential provider owns long-lived token cache and refresh policy.
- npm must not implement long-lived caching — the CLI keeps a
  **process-lifetime in-memory cache** only.
- A single `npm install` may trigger hundreds of registry requests across
  sequential phases (dependency resolution, then tarball fetching). Without
  process-lifetime caching, the provider would be spawned once per phase per
  registry.
- Cache key: provider id + registry base URI. If the protocol gains an
  `operation` field (read vs. write tokens), the key must include it.
- npm must not persist tokens to disk.
- `forceRefresh: true` evicts npm's in-memory cache and signals the provider
  to bypass its own cache. It is never set on the first call of a session —
  only on retry after an auth failure.

### Security Considerations

#### Provider Binary Integrity

A credential provider is an arbitrary executable that npm spawns and hands
registry context to. If an attacker can substitute that binary, they gain
arbitrary code execution in the user's security context.

Security invariants:

1. **Hash-at-registration, verify-at-every-spawn.** npm records SHA-256 at
   registration and re-hashes on every spawn (including mid-restore
   re-invocations). Mismatch = fail closed.
2. **Registration is the trust boundary.** Installation alone does not grant
   trust. Only explicit `npm credential-provider trust` authorizes spawning.
   Project `.npmrc` can name a provider but cannot grant trust.
3. **Absolute-path resolution, no PATH search.** npm spawns the registered
   absolute path — never searches `$PATH`.
4. **Restrictive file-system permissions.** Provider files writable only by
   installing user/admin. npm verifies at registration and warns/fails if
   too permissive.

#### Threat model

| Threat | Attack vector | Defense |
|--------|--------------|---------|
| Binary substitution at rest | Malware, compromised install script, shared workstation | Hash check (1) + permissions (4) |
| Mid-restore TOCTOU | Binary replaced between spawns | Hash check on every spawn (1) |
| Malicious lifecycle scripts | `postinstall` overwrites provider | Permissions (4) + hash (1) + `ignore-scripts` |
| PATH poisoning | Malicious `.env`, shell profile, CI manipulation | Absolute-path resolution (3) |
| Typosquatting | Similarly-named package on public registry | Registration boundary (2) |
| Malicious project `.npmrc` | Repo plants rogue provider config | Project config can name but not grant trust (2) |

#### Token Exposure

Required mitigations:
- Tokens never passed on command lines, persisted in `.npmrc`, or re-exported
  via environment variables.
- npm must not expose credentials to lifecycle scripts, `npm exec`, or child
  processes.
- npm must redact Authorization headers, tokens, and passwords from logs,
  error objects, and diagnostic reports.

#### Redirect Handling

- Strip `Authorization` on all cross-origin redirects (scheme, host, or port
  change).
- Same-origin redirects may preserve credentials only within the same
  registry boundary.
- Redirect to a different configured registry = new auth context = re-acquire
  credentials for that target.

## Prior Art

- **pnpm tokenHelpers** — pnpm supports `tokenHelpers` in `.npmrc` that invoke
  external commands to retrieve tokens. Similar concept but uses raw command
  strings without integrity verification.
- **NuGet credential providers** — .NET ecosystem uses a plugin protocol for
  credential acquisition with structured JSON communication over stdio.
- **pip keyring** — Python's pip delegates credential storage/retrieval to the
  system keyring via a plugin interface.
- **Git credential helpers** — Git invokes configured helpers via stdio to
  acquire credentials for remote operations.
- **Cargo credential providers** — Rust's Cargo uses a stdin/stdout JSON
  protocol for credential provider plugins, which this design closely follows.

## Unresolved Questions and Bikeshedding

- **`npm credential-provider trust` subcommand:** Does not exist yet. Required
  before this feature can ship. Must compute SHA-256, display for confirmation,
  and record in user config. UX details TBD (interactive confirmation prompt,
  `--yes` flag for CI, output format).
- **Lockfile-derived trust (future enhancement):** Can `package-lock.json`
  SRI eventually supplement explicit registration for project-level providers?
  Not recommended for v1.
- **Provider-id vs. raw command:** Should raw command strings be supported as a
  compatibility/migration path? If so, should they be marked legacy-only?
- **`npm login`/`npm logout` behavior** when a provider is configured:
  `npm login` currently persists tokens to `.npmrc` via
  `config.setCredentialsByURI()`, conflicting with provider-owned lifecycle.
  Options: disable when provider configured, or delegate to provider.
- **Cache key dimensions:** If an `operation` field is added (read-only token
  for install vs. read-write for publish), the cache key must include it.
- **Operation/command in request:** Should providers know whether npm is
  installing vs. publishing vs. `npx`? Enables scoped tokens but raises
  questions about transitive invocations.
- **Global kill-switch:** `--no-credential-provider` to disable all providers
  and fall back to legacy auth. Should emit a warning since it re-enables
  plaintext tokens.

## Acknowledgments

This RFC builds on protocol design work by
[@Filyus](https://github.com/Filyus)
([unofficial-npm-credential-provider-rfc](https://github.com/Filyus/unofficial-npm-credential-provider-rfc)),
particularly the bidirectional JSON protocol design, structured error kinds,
version negotiation concepts, and provider chaining model. Key ideas from that
draft — including per-scope/per-package granularity, explicit `refresh` and
`erase` request kinds, batch mode, and the security threat model around
project-local resolution — informed the design choices in this RFC.
