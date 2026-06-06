# MCP Kindle Send + OAuth 2.1 — Design

**Date:** 2026-06-06
**Status:** Approved design (re-scoped to Rails-only), pre-implementation

## Goal

Let a user say to Claude Code "send that markdown file to my Kindle." readitsoon (Rails)
exposes a remote Streamable-HTTP MCP endpoint with a `send_markdown_to_kindle` tool,
guarded by an OAuth 2.1 Authorization Server that readitsoon also hosts, reusing its
existing markdown→EPUB→Kindle pipeline.

## Re-scope note (supersedes earlier two-repo design)

The MCP server is **not** a separate TypeScript service. Everything lives inside readitsoon
on `readitsoon.chiq.me` as Rails resources (controllers, models, service classes). Because the
MCP Resource Server and the OAuth Authorization Server are the **same Rails app**, access tokens
are validated **locally** via `doorkeeper_authorize!` — no token introspection endpoint and no
second public host are needed. The `readitsoon-mcp` repo holds only the spec, plan, and the
E2E test fixtures.

## Decisions

- **Transport:** Remote HTTP, MCP Streamable HTTP transport, served by Rails.
- **MCP implementation:** Official `modelcontextprotocol/ruby-sdk` gem. Tool = `MCP::Tool` subclass;
  endpoint = `Mcp::ServerController` mounting `StreamableHTTPTransport` (`stateless: true`).
- **OAuth roles:** Claude Code = OAuth client. readitsoon = Authorization Server **and** Resource Server.
- **Token validation:** Local Doorkeeper `doorkeeper_authorize!` on the MCP endpoint. Opaque tokens.
- **Client registration:** RFC 7591 Dynamic Client Registration (hand-built; Doorkeeper lacks it).
- **Identity:** The kindle email = the `Email` record (`token.resource_owner_id`). Single-email identity.
- **OTP delivery:** OTP inside an `.epub` via the existing Kindle pipeline, with an in-flow
  step instructing the user to whitelist readitsoon's sender first.
- **Resend OTP:** Link on the OTP verify screen, enabled 30s after last send; server-enforced 30s gate.
- **AS implementation:** Doorkeeper gem for `/oauth/authorize`, `/oauth/token`, PKCE, token storage;
  custom controllers for DCR, AS + protected-resource metadata, and the OTP login.
- **Reuse:** Reuse readitsoon's EPUB/delivery/OTP services. New code lives under an `Mcp::` namespace.
- **Delivery proof:** `MailgunEmailClient` logs to STDOUT when it hands the EPUB to Mailgun.

## Components (all in readitsoon, `Mcp::` namespace)

- **Doorkeeper** — `/oauth/authorize`, `/oauth/token`, PKCE, public clients, token storage.
- **`Mcp::MetadataController`** — `/.well-known/oauth-authorization-server` (RFC 8414) and
  `/.well-known/oauth-protected-resource` (RFC 9728, advertising the `/mcp` resource + this AS).
- **`Mcp::RegistrationsController`** — RFC 7591 `POST /oauth/register` (DCR). Public client + PKCE.
- **`Mcp::AuthSessionsController`** (+ views) — the custom login Doorkeeper hands off to via
  `resource_owner_authenticator`:
  - `new` — enter kindle email + whitelist-the-sender instructions.
  - `create` — issue OTP, deliver OTP epub.
  - OTP verify screen with `resend` action (30s server-enforced gate).
  - `verify`/`confirm` — check OTP, mark session authenticated, return to Doorkeeper for consent + redirect.
- **`SendMarkdownToKindleTool`** (`MCP::Tool` subclass, lives in `app/mcp/`) — the tool logic:
  builds an `Article` and enqueues `DeliveryJob`, reusing `EpubCreator`/`DeliveryService`.
- **`Mcp::ServerController`** — `POST /mcp`, `doorkeeper_authorize!`-guarded. Builds an `MCP::Server`
  with the tool, passes `server_context: { email_id: token.resource_owner_id }`, hands the request
  to a stateless `StreamableHTTPTransport`.

## End-to-end flow

1. Claude Code connects to `/mcp` → no token → `401` + `WWW-Authenticate` → protected-resource metadata.
2. Claude Code reads `/.well-known/oauth-protected-resource` → discovers the AS (same host) →
   `POST /oauth/register` (DCR) → `client_id`.
3. Claude Code opens browser to `/oauth/authorize` (PKCE `code_challenge`).
4. Doorkeeper `resource_owner_authenticator` → no session → `Mcp::AuthSessions#new`:
   enter kindle email → whitelist instructions → send OTP epub → verify screen
   (resend enabled after 30s) → correct OTP → session authenticated → consent → redirect with `code`.
5. Claude Code `POST /oauth/token` (code + PKCE verifier) → opaque access token (+ refresh).
6. Claude Code calls the tool with the Bearer token → `Mcp::ServerController` runs `doorkeeper_authorize!`,
   resolves the `Email` from `resource_owner_id`, runs `SendMarkdownToKindleTool` →
   `EpubCreator.convert` + `DeliveryService.call` (+ `MailgunEmailClient` STDOUT log) → EPUB to kindle.

## The MCP tool: `send_markdown_to_kindle`

- `markdown` (required) — raw markdown.
- `filename` (required) — basename (minus extension) → EPUB title.
- `author` (optional) — caller may override; Claude infers from the markdown content
  (frontmatter/byline/title block) when present; falls back to the authed email's local-part if omitted.
- Kindle email is NOT a param — derived from the authenticated token's `resource_owner_id`.
  Cannot send to arbitrary addresses.

## Reuse map (readitsoon)

| Need | Reuse |
|------|-------|
| markdown→EPUB | `EpubCreator.convert` |
| EPUB→Kindle email | `DeliveryService.call` |
| OTP issue/verify | `Email#issue_sign_in_otp!` / `#valid_sign_in_otp?` / `#consume_sign_in_otp!` |
| OTP-as-epub delivery | `OtpEpubBuilder` + `OtpSignInDeliveryService` |
| abuse guard | `SignInIpGuard` on auth-session + register endpoints |

New code: Doorkeeper config + migrations, `Mcp::MetadataController` (AS + protected-resource),
`Mcp::RegistrationsController` (DCR), `Mcp::AuthSessionsController` (+ views, resend),
`SendMarkdownToKindleTool` (`MCP::Tool`), `Mcp::ServerController` (Streamable HTTP), routes,
`mcp` gem, and a STDOUT log in `MailgunEmailClient`.

## Error handling

- No/invalid token at `/mcp` → `401` + `WWW-Authenticate` (Doorkeeper) → triggers re-auth.
- OTP wrong/expired → re-prompt; `SignInIpGuard` ban on repeated failure.
- Resend before 30s → server rejects (`429`).
- Delivery failure / monthly limit → reuse existing `articles.send` limit logic, returned as an
  MCP tool error (`MCP::Tool::Response` with `error: true`).
- Empty markdown → MCP tool error response.

## Testing

- **Controllers:** request specs for metadata (AS + protected-resource), DCR, auth-session
  (OTP + resend + 30s gate), and `/mcp` (401 unauthenticated; `tools/call` happy/limit/empty
  with a valid token). Doorkeeper authorization-code + PKCE integration spec.
- **Tool:** unit spec for `SendMarkdownToKindleTool.call` (title from filename, author default,
  delivery enqueued, error responses).
