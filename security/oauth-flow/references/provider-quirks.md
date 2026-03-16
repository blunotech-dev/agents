# Provider Quirks

## Google
- Add `access_type=offline` to get a refresh token (only issued on first consent)
- Add `prompt=consent` to force re-issue of refresh token
- `id_token` always returned with `openid` scope — no separate userinfo call needed for basic profile
- Restrict to a Workspace domain: `hd=company.com`

## GitHub
- **No PKCE support** — omit `code_challenge` params
- **Not OIDC** — no `id_token`, no discovery document; call `GET /user` with the access token
- Access tokens **don't expire** by default (unless GitHub App is configured for expiring tokens)
- Token response is form-encoded by default — add `Accept: application/json`
- `user:email` scope needed for private emails; `user` only gives public email

## Microsoft (Entra ID / Azure AD)
- Discovery: `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration`
  - `common` for multi-tenant; your tenant ID for single-tenant
- Add `offline_access` scope for refresh tokens
- Access tokens expire in 1 hour; refresh tokens slide 24 hours
- Use `oid` claim (not `email`) as the canonical user identifier — email can change
- Multi-tenant: validate `tid` claim to block unexpected tenants

## Auth0
- Without `audience`: access token is opaque, valid only for `/userinfo`
- With `audience`: access token is a JWT you can validate locally
- Enable refresh token rotation in Auth0 dashboard (not on by default)
- Custom scopes must be defined in the Auth0 API settings first

## Generic OIDC Provider Checklist
1. Discovery doc: `{issuer}/.well-known/openid-configuration`
2. Check `code_challenge_methods_supported` — confirm S256 PKCE is listed
3. Check `jwks_uri` — needed for id_token signature validation
4. Confirm `grant_types_supported` includes `authorization_code`
5. Test whether `offline_access` scope issues a refresh token
6. Check `scopes_supported` for available scopes

## Non-OIDC Providers (GitHub, Stripe, Twitter/X)
No `id_token`, no discovery doc. After token exchange, call the provider's `/me` or `/user` endpoint to get identity. Use the provider-specific immutable ID field (not email) as your user identifier.