---
title: Credential Provider Plugin Protocol for Secure npm Authentication
number: null
status: proposed
created: null
accepted_at: null
implemented_at: null
withdrawn_at: null
implementation: null
---

# Credential Provider Plugin Protocol for Secure npm Authentication

## Summary

This RFC proposes a credential provider protocol for the npm CLI. Users configure an external executable for a registry, and npm requests credentials through a JSON exchange over standard input and output. Providers own authentication and secure storage; npm uses returned credentials only in memory. The protocol supports short-lived credentials without requiring users to persist tokens in `.npmrc` or environment variables.

## Motivation

Token-based registry authentication commonly requires users to place credentials in `.npmrc` or environment variables. This can conflict with organizational security policies, and manually replacing expired tokens discourages short token lifetimes.

Credential providers let npm use registry credentials acquired through an organization's existing authentication system. They support developer and CI workflows, multiple registries, and third-party registries without prescribing an identity provider or storage mechanism.

## Detailed Explanation

### Configuration and discovery

A provider is configured per registry in a user, global, or builtin configuration file:

```ini
//registry.example.com:credentialProvider=corp-npm-credprovider
```

These files are **trusted configuration sources**. All `credentialProvider*` settings from project configuration, direct CLI arguments, or `npm_config_*` environment variables are ignored with a warning. npm's existing `userconfig` mechanism may select the trusted user configuration file.

The value is an absolute executable path or a bare executable name, not an npm package identifier or shell command. Only one provider is supported per registry. Global-default and per-scope providers are not supported: the provider must be bound to a specific registry rather than follow a registry selected by a project.

If an auth token and a provider apply to the same registry, npm errors and does not use the token. Tokens and providers for different registries may coexist.

Provider resolution follows these rules:

- Absolute paths must identify an existing executable. Bare names are resolved to an absolute path using only absolute, non-empty `PATH` entries.
- Resolution must not search the current working directory.
- Relative paths, configured arguments, shell command strings, and home-directory expansion are not supported. Environment-variable expansion follows existing npm configuration behavior.
- npm spawns the resolved executable directly, with no arguments and no shell. Windows `.cmd` and `.bat` providers are not supported; script-based providers must supply a directly executable launcher.
- npm logs each resolved provider path on its first invocation per process, using existing npm logging conventions and the configured log level.

Configuring a provider grants it execution trust. npm does not install, update, or verify its provenance. Users and administrators must protect the executable and its containing directory from untrusted modification. Absolute paths are recommended, particularly for CI; project-controlled installation locations are discouraged.

### Command behavior

For registry operations without cached provider credentials, npm sends the request without provider authentication. If the registry returns `401` and the request is replayable, npm requests credentials through `get` with `retry: false` and retries the failed request. npm uses returned credentials as request-scoped authentication and caches successful responses for subsequent matching requests in the current process.

For any operation they do not support, providers must return `operation-not-supported`.

| Operation | Provider behavior | npm behavior on success |
|-----------|-------------------|-------------------------|
| `get` | Acquire credentials, silently or interactively as allowed. | Use and cache the credentials in memory, without writing them to configuration or disk. |
| `login` | Force fresh authentication and persist session state in provider-managed secure storage. | Do not receive or persist credential material. |
| `logout` | Remove the registry's stored credentials. Server-side revocation is optional and must not prevent local removal on network failure. | Do not remove `.npmrc` token entries; logout is delegated to the provider. |

Any provider failure during `get` or `login`, including an unsupported URL or operation, fails the command without falling back to legacy authentication. During `logout`, provider failure causes npm to remove legacy credentials from the user configuration and warn that provider-managed credentials may remain. Cancellation is an exception: npm terminates the provider without retry or legacy fallback.

Without a configured provider, existing authentication, login, and logout behavior is unchanged.

### Trusted publishing

Credential providers and trusted publishers are separate authentication paths. For `npm publish` and `npm stage publish`, supported trusted publishing is attempted first. npm must never send OIDC tokens, claims, or exchange responses to a provider.

If trusted publishing falls back to traditional authentication, the provider/token conflict rule applies, and provider acquisition follows the challenge-based flow.

`npm trust` uses the normal credential path: `list` requests `read-only` permission; create, update, and revoke request `read-write`. This proposal adds neither a `trust` protocol request nor new trusted-publisher types.

### Protocol

Each invocation handles one request:

1. npm writes exactly one UTF-8 JSON object to the provider's `stdin`, then closes the stream.
2. The provider writes exactly one UTF-8 JSON response object to `stdout`, then exits.
3. Exit code zero and a valid `ok` response indicate success. An `err` response or non-zero exit indicates failure.

`stdout` is reserved for the protocol. Instructions and diagnostics use inherited `stderr`; npm does not parse that stream or treat diagnostic output as failure.

npm serializes provider invocations across registries to avoid overlapping prompts.

#### Requests

A `get` request:

```json
{
  "versions": [1],
  "kind": "get",
  "registry": "https://registry.example.com/",
  "permission": "read-only",
  "interactive": true,
  "retry": false,
  "logLevel": "info"
}
```

| Field | Applies to | Required | Meaning |
|-------|------------|----------|---------|
| `versions` | All requests | Yes | Unique positive integers in descending preference order. v1 is `[1]`. |
| `kind` | All requests | Yes | `get`, `login`, or `logout`. |
| `registry` | All requests | Yes | Fully qualified absolute registry base URI, resolved from the registry-scoped configuration key that selected the provider. |
| `logLevel` | All requests | Yes | Current npm log verbosity. |
| `network` | All requests | No | Proxy and TLS configuration, as described below. |
| `permission` | `get` | Yes | `read-only` or `read-write`, following npm's token permission vocabulary. Providers without permission-scoped tokens may return credentials valid for both. |
| `accountHint` | `get`, `login` | No | User-managed `credentialProviderAccountHint` from trusted configuration. Omitted when absent; providers may ignore it. |
| `interactive` | `get`, `login` | Yes | Whether user interaction is permitted. |
| `retry` | `get`, `login` | Yes | For `get`, initially `false`, then `true` only after provider credentials are rejected. Always `true` for `login` to bypass cached authentication. |

`login` omits `permission`: it establishes identity or session state, and any persisted permission scope is provider-defined. `logout` omits `permission`, `accountHint`, `interactive`, and `retry`.

npm does not create or update `credentialProviderAccountHint` automatically and does not add an `--accountHint` flag.

**Interaction:** `interactive` defaults to `true`, without TTY or CI detection. The new `--no-interactive` option sets it to `false` on commands that may invoke a provider. CI callers should use that option or `npm_config_interactive=false`.

When interaction is allowed, providers may display authentication instructions on `stderr`; these should identify the requesting registry. Providers must not use `stdin` for interactive input because it carries the protocol request. When interaction is disabled, providers must return an error if silent authentication is unavailable rather than wait for user input.

**Network context:** Optional `network` fields are `httpsProxy`, `httpProxy`, `noProxy`, `strictSsl`, and `caFile`. Missing fields mean not configured. npm sources these fields exclusively from user/global `.npmrc`, never project configuration. Providers may need them for their own outbound identity-provider or revocation requests.

**Version negotiation:** Providers select the first supported entry from `versions` and return it as `ok.v`. npm rejects other selections. If none is supported, the provider returns `version-not-supported`, identifying supported versions in its human-readable message; npm fails with update guidance.

Providers must ignore unknown request fields. Breaking protocol changes require a major version; adding optional fields does not. The negotiation envelope remains stable across versions.

#### Responses

A response contains exactly one `ok` or `err` object.

Successful `get`:

```json
{
  "ok": {
    "v": 1,
    "kind": "get",
    "auth": {
      "type": "bearer",
      "token": "<token>"
    }
  }
}
```

`ok.v` and `ok.kind` are required and must match the negotiated version and request kind. `get` also requires `ok.auth`, with one of these shapes:

| Auth type | Required fields | npm behavior |
|-----------|-----------------|--------------|
| `bearer` | `type: "bearer"`, `token` | Send the token using bearer authorization. |
| `basic` | `type: "basic"`, `username`, `password` | Use npm's existing HTTP Basic authentication behavior to construct the authorization header. |

Providers return unencoded credential values; npm handles authorization-header construction. For Basic authentication, the header is `Authorization: Basic <base64(username:password)>`. The provider's `password` is not the Base64-encoded `_password` configuration value used by legacy npm configuration.

Successful `login` and `logout` responses contain only `ok.v` and the matching `ok.kind`; they do not return credentials.

Failure:

```json
{
  "err": {
    "kind": "operation-not-supported",
    "message": "This provider does not support login."
  }
}
```

`err.kind` and `err.message` are required strings. Messages must be human-readable and contain no secrets or personally identifiable information.

| Error kind | Meaning |
|------------|---------|
| `url-not-supported` | The provider does not handle the registry. Providers should use this for unrecognized hosts. |
| `version-not-supported` | No offered protocol version is supported. |
| `operation-not-supported` | The request kind is not supported. |
| `other` | Another provider failure. |

All errors follow the operation-specific failure rules in [Command behavior](#command-behavior).

#### Validation and process limits

npm rejects malformed responses: invalid UTF-8 or JSON, anything other than one JSON object with exactly one valid `ok` or `err` result, missing required fields, unsupported or inconsistent versions/kinds/auth types, or a non-zero exit after `ok`.

Provider `stdout` is limited to 1 MiB. npm kills a provider that exceeds this limit and treats its response as malformed. Diagnostics must identify the provider and validation failure without logging response content or secrets.

All request kinds have a default timeout of 120 seconds, accommodating interactive authentication. Trusted configuration may override it through `credentialProviderTimeoutMs`, globally or per registry. npm kills timed-out providers and applies the operation's failure rules. On command cancellation, npm terminates the provider without retry or legacy fallback. Timeouts and cancellation are process failures, not malformed responses.

### Credential scope and transport

Returned credentials are bound to the registry configuration key's scope: host, explicit port, and path. npm attaches them to matching requests without writing them into configuration.

Before invoking a provider or attaching cached provider credentials, npm must verify that the effective registry URL uses HTTPS. Non-HTTPS provider-backed `get` and `login` fail without invoking the provider; `logout` performs its legacy cleanup and warns that provider-managed credentials may remain.

On redirects, npm strips provider authorization when the hostname changes or HTTPS downgrades to HTTP. Other same-host HTTPS redirects retain existing behavior. These rules apply to provider credentials, not legacy HTTP authentication.

### Caching and retry

npm caches successful provider responses only for the process lifetime, keyed by resolved executable path, registry base URI, and permission. Providers own longer-lived secure storage and refresh policy.

There is at most one initial acquisition per registry and permission. Cached credentials are reused for matching requests. npm must coalesce concurrent challenges for the same cache key into one provider invocation.

A `401` rejecting cached provider credentials evicts all permission variants for that provider and registry. For a replayable request, npm invokes the provider with `retry: true` and retries only the failed request, with at most one refresh per failed registry request. If refreshed credentials are rejected, the command fails.

Only `401`, not `403`, triggers acquisition or refresh in v1. npm must never replay an entire command or Arborist operation to recover authentication. If the failed request cannot be safely replayed, the command fails without retrying it.

### Security boundary

Providers run with the user's identity and must not be treated as sandboxed. This proposal avoids npm-managed persistent credential storage; it does not protect against arbitrary code already running as the user or an attacker able to read process memory. Providers must avoid granting authority beyond the user's existing session.

Before spawning a provider, npm removes npm-owned credentials from its inherited environment: `npm_config_otp`, top-level `_auth`, nerf-darted `_auth`, `_authToken`, and `_password`, and `npm_config_proxy`/`npm_config_https_proxy`, which may contain credentials. Matching is case-insensitive and should reuse npm's existing private-configuration-key classification.

This filtering limits unnecessary disclosure but cannot identify secrets in arbitrary environment-variable substitutions. Other environment state is inherited; high-assurance callers should launch npm with a curated environment. npm must never log credentials or authorization headers.

## Rationale and Alternatives

The provider interface separates npm's registry requests from identity-provider-specific authentication and storage. Explicit executable configuration and a one-shot exchange keep the integration independent of package installation and avoid a shell-based extension point.

- **Existing credential configuration or wrappers:** `.npmrc` and environment variables do not provide dynamic acquisition. Encrypting local configuration adds key-management complexity without addressing rotation. Piping an ephemeral user config avoids persistent storage in some environments but is not portable across platforms, interactive commands, and nested npm invocations.
- **Package-managed or shell-command providers:** Global npm package resolution depends on the active installation prefix; `npx`/`npm exec` may resolve project dependencies or download code. Shell command strings introduce quoting and injection concerns. A directly executable path or name avoids these dependencies. Providers may still be distributed as npm packages, but npm does not manage their installation in v1.
- **Long-lived bidirectional providers:** These can amortize startup cost, but require framing and process-lifecycle management on both sides. A one-shot process uses EOF as its delimiter and has no cross-request process state.

### Credential acquisition timing

This proposal uses challenge-based acquisition: providers are invoked after a registry returns `401`, rather than pre-loading credentials before a command. This avoids invoking providers for unused registries or anonymous requests and supports request-level credential refresh.

Pre-loading credentials in the CLI was considered because it supplies credentials on the first request and supports non-replayable bodies. It requires identifying registries and permissions in advance, may invoke unused providers, and does not provide mid-command refresh.

Challenge-based acquisition adds an initial challenge round trip and requires safe request replay and redirect handling. Consumed streaming bodies cannot be retried. The implementation requires a shared process-lifetime auth store, cached authentication on subsequent requests, and coalescing of concurrent challenges. Integration may be central to `npm-registry-fetch` or use CLI-owned fetch injection through Arborist, pacote, and other registry consumers.

## Implementation

- **npm CLI and configuration:** Add trusted-source provider configuration, executable resolution, serialized process invocation, protocol validation, timeouts, environment filtering, and the `--no-interactive` option.
- **Authentication and registry consumers:** Add the shared in-memory credential store and request-time challenge handling across `npm-registry-fetch`, Arborist, pacote, and other consumers, including request-scoped authentication, concurrent-challenge coalescing, and safe request-level retries.
- **Commands:** Delegate `login` and `logout` when configured, preserve trusted-publishing precedence, and apply registry/token conflict and failure rules consistently.
- **Validation:** Cover configuration provenance, executable resolution, protocol versions and malformed responses, process limits and cancellation, command failure behavior, transport scoping, concurrency, and retries. Existing authentication without a provider must remain unchanged.

## Prior Art

- [Cargo credential providers](https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html) use structured JSON and `get`, `login`, and `logout` operations. This proposal adapts those concepts to a one-shot exchange.
- Git credential helpers use short-lived processes and standard streams for credential operations.
- NuGet credential providers use a long-lived plugin protocol; pnpm token helpers and pip keyring also delegate credential acquisition or storage to external components.

This proposal builds on [@pwoosam's credential provider RFC, npm/rfcs#850](https://github.com/npm/rfcs/pull/850), and the [unofficial npm credential provider RFC](https://github.com/Filyus/unofficial-npm-credential-provider-rfc). Their provider configuration, structured protocol, and authentication-operation designs informed this proposal.

## Unresolved Questions and Bikeshedding

- **Installation guidance:** Should npm recommend conventional provider installation locations, or limit guidance to protecting executables and avoiding project-controlled locations? Provider distribution, installation, updates, and removal remain the provider owner's responsibility in v1.
- **Future challenge context:** v1 forwards neither `WWW-Authenticate` values nor HTTP status codes. Rejected credentials are signaled only through `retry: true`; richer context may be considered later.
