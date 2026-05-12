# ADR-011: Dual Authentication — ChatGPT OAuth and API Key

## Status
Accepted

## Date
2026-05-12

## Context
Codex needs to authenticate with OpenAI's APIs. The target user base is:
- **Developers**: already have `OPENAI_API_KEY`, prefer environment-variable-based auth
- **ChatGPT subscribers**: have a ChatGPT Plus/Pro/Team plan that includes Codex access; do not want to separately manage API keys
- **Enterprise**: need agent identity tokens (JWT) for service-to-service scenarios
- **Power users**: need token portability (keyring, file, ephemeral)

## Decision
Support four authentication methods via `CodexAuth` enum in `codex-rs/login/`:
1. **`ApiKeyAuth`**: reads `OPENAI_API_KEY` or `CODEX_API_KEY` environment variables
2. **`ChatgptAuth`**: OAuth 2.0 device code flow with PKCE; token stored per `AuthCredentialsStoreMode`; auto-refresh within 8-hour window
3. **`ChatgptAuthTokens`**: accepts pre-obtained ChatGPT tokens (for programmatic use)
4. **`AgentIdentityAuth`**: JWT-based for service-to-service scenarios

Credentials are stored in one of four modes: `file` (`$CODEX_HOME/auth.json`), `keyring` (OS keyring via `keyring` crate), `auto` (default: tries keyring, falls back to file), `ephemeral` (in-memory only).

## Alternatives Considered

### API key only
- Pros: Simplest implementation
- Cons: ChatGPT subscribers would need to create a separate API key; breaks the "just log in" UX
- Rejected: ChatGPT auth is a significant user segment

### OAuth only
- Pros: Secure, no long-lived secrets in files
- Cons: Developers running in CI/CD pipelines cannot use browser-based OAuth; `OPENAI_API_KEY` is the de-facto standard for programmatic access
- Rejected: CI/scripting use cases require API key auth

### Storing all credentials in plaintext config
- Pros: Simple, inspectable
- Cons: Config files are often checked into version control accidentally; OS keyring is more secure
- Rejected: Keyring-first with file fallback is a better security posture

## Consequences
- `codex login` and `codex logout` CLI commands manage the ChatGPT OAuth flow
- API key auth reads from environment (12-factor app compatible)
- `cli_auth_credentials_store` config key selects the storage mode
- `SessionConfiguredEvent` includes the active authentication method for observability
