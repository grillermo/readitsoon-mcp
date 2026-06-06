# MCP Kindle Send + OAuth 2.1 — Design

**Date:** 2026-06-06
**Status:** Approved design, pre-implementation

## Goal

Let a user say to Claude Code "send that markdown file to my Kindle." A remote HTTP
MCP server (TypeScript) exposes a `send_markdown_to_kindle` tool. readitsoon (Rails)
gains an OAuth 2.1 Authorization Server and a token-guarded send endpoint, reusing its
existing markdown→EPUB→Kindle pipeline.

## Decisions

- **Transport:** Remote HTTP (hosted), Streamable HTTP MCP transport.
- **OAuth roles:** Claude Code = OAuth client. TS MCP server = Resource Server. readitsoon = Authorization Server.
- **Token validation:** Opaque access tokens + RFC 7662 introspection (TS server calls readitsoon, short-TTL cache).
- **Client registration:** RFC 7591 Dynamic Client Registration (hand-built; Doorkeeper lacks it).
- **Identity:** The kindle email = the `Email` record. Single-email identity.
- **OTP delivery:** OTP inside an `.epub` via the existing Kindle pipeline, with an in-flow
  step instructing the user to whitelist readitsoon's sender first.
- **Resend OTP:** Link on the OTP verify screen, enabled 30s after last send; server-enforced 30s gate.
- **AS implementation:** Doorkeeper gem for `/oauth/authorize`, `/oauth/token`,
  `/oauth/introspect`, PKCE, AS metadata, token storage; custom shims for DCR and the OTP login.
- **Reuse:** Reuse readitsoon's EPUB/delivery/OTP services. New code lives under an `Mcp::` controller namespace.

## Components

### readitsoon (Rails) — `Mcp::` namespace + Doorkeeper

- **Doorkeeper** provides `/oauth/authorize`, `/oauth/token`, `/oauth/introspect`, PKCE,
  token storage, and AS metadata at `/.well-known/oauth-authorization-server`.
- **`Mcp::RegistrationsController`** — RFC 7591 `POST /oauth/register` (DCR). Public client + PKCE.
- **`Mcp::AuthSessionsController`** (+ views) — the custom login Doorkeeper hands off to via
  `resource_owner_authenticator`:
  - `new` — enter kindle email + whitelist-the-sender instructions.
  - `create` — issue OTP, deliver OTP epub.
  - OTP verify screen with `resend` action (30s server-enforced gate).
  - `verify` — check OTP, mark session authenticated, return to Doorkeeper for consent + redirect.
- **`Mcp::SendController`** — `POST /mcp/send`, Bearer-guarded by Doorkeeper. Accepts pure
  markdown + title + author; enqueues delivery via the existing pipeline.

### readitsoon-mcp (TypeScript) — Streamable HTTP MCP server

- `@modelcontextprotocol/sdk` server, remote HTTP transport.
- Exposes `/.well-known/oauth-protected-resource` (RFC 9728) pointing at the readitsoon AS.
- Returns `401 + WWW-Authenticate` when token missing/invalid.
- Per-request token validation via readitsoon `/oauth/introspect`, short-TTL cache.
- One tool: `send_markdown_to_kindle`.

## End-to-end flow

1. Claude Code connects → no token → `401` + `WWW-Authenticate` → protected-resource metadata.
2. Claude Code discovers readitsoon AS → `POST /oauth/register` (DCR) → `client_id`.
3. Claude Code opens browser to `/oauth/authorize` (PKCE `code_challenge`).
4. Doorkeeper `resource_owner_authenticator` → no session → `Mcp::AuthSessions#new`:
   enter kindle email → whitelist instructions → send OTP epub → verify screen
   (resend enabled after 30s) → correct OTP → session authenticated → consent → redirect with `code`.
5. Claude Code `POST /oauth/token` (code + PKCE verifier) → opaque access token (+ refresh).
6. Claude Code calls tool with Bearer token → MCP server introspects → `POST /mcp/send`
   → `EpubCreator.convert` + `DeliveryService.call` → EPUB to kindle.

## The MCP tool: `send_markdown_to_kindle`

- `markdown` (required) — raw markdown.
- `filename` (required) — basename → EPUB title.
- `author` (optional) — caller may override; Claude infers from the markdown content
  (frontmatter/byline/title block) when present; falls back to the authed email's local-part if omitted.
- Kindle email is NOT a param — derived from the introspected token's identity. Cannot send to arbitrary addresses.

## Reuse map (readitsoon)

| Need | Reuse |
|------|-------|
| markdown→EPUB | `EpubCreator.convert` |
| EPUB→Kindle email | `DeliveryService.call` |
| OTP issue/verify | `Email#issue_sign_in_otp!` / `#valid_sign_in_otp?` / `#consume_sign_in_otp!` |
| OTP-as-epub delivery | `OtpEpubBuilder` + `OtpSignInDeliveryService` |
| abuse guard | `SignInIpGuard` on auth-session + register endpoints |

New code: Doorkeeper config + migrations, `Mcp::RegistrationsController` (DCR),
`Mcp::AuthSessionsController` (+ views, resend), `Mcp::SendController`, routes, metadata wiring.

## Error handling

- Invalid/expired token → `401` from MCP server → triggers re-auth.
- OTP wrong/expired → re-prompt; `SignInIpGuard` ban on repeated failure.
- Resend before 30s → server rejects (`429`).
- Delivery failure / monthly limit → reuse existing `articles.send` limit logic, surfaced as tool error.
- Empty markdown → `400`.

## Testing

- **readitsoon:** request specs per `Mcp::` controller (DCR; auth-session OTP + resend + 30s gate;
  send happy/limit/empty); Doorkeeper authorization-code + PKCE integration spec.
- **MCP server:** token-guard unit tests (401 paths, introspection cache); tool schema/dispatch
  test against a stubbed readitsoon.
