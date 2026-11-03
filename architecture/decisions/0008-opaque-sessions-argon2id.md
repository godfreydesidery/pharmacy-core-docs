# 0008 Opaque session tokens and Argon2id passwords

- Status: accepted
- Date: 2026-11-02

## Context
The PRD requires forced logout and session revocation (AUTH-6). A JWT can't be revoked before it expires without a deny list, which brings back server state anyway.

## Decision
- **Tokens.** Sign-in issues an opaque token: `pcs_` + 32 random bytes, base64url.
- **Storage.** Only its SHA-256 hash is stored in `access.sessions`, with an expiry (12 h by default). Sign-out revokes it.
- **Passwords.** Hashed with Argon2id (Spring Security crypto with Bouncy Castle), at least 8 characters.
- **Per request.** The edge authenticates every request by hash lookup. The tenant is chosen per request, not baked into the token.

## Consequences
- Instant revocation. One session can be used across tenants, for users who belong to several pharmacies.
- One indexed lookup per request. It can be cached in Redis later if needed.
- SSO (OIDC) can be added later as another way to create a session.
