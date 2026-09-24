# Installing The Read It Soon MCP Server

This guide shows how to connect MCP clients to the Read It Soon Kindle sender.

The MCP endpoint is:

```text
https://readitsoon.app/mcp
```

For local development with `../readitsoon`, use:

```text
http://localhost:4001/mcp
```

## Prerequisites

Start the Rails app before installing or authenticating the MCP server:

```bash
cd ../readitsoon
./serve-dev
```

Expected local services:

- Rails: `http://localhost:4001`
- MCP: `http://localhost:4001/mcp`
- OAuth issuer: `http://localhost:4001`

Before requesting the OTP, add this approved sender in Amazon:

```text
sending@readitsoon.app
```

## Claude Code

Install the local MCP server:

```bash
claude mcp add --transport http readitsoon https://readitsoon.app/mcp
```

Use a user-wide install instead of the default project-local install:

```bash
claude mcp add --scope user --transport http readitsoon https://readitsoon.app/mcp
```

Check the server:

```bash
claude mcp list
claude mcp get readitsoon
```

Authenticate:

1. Run `claude`.
2. Type `/mcp`.
3. Select `readitsoon`.
4. Choose the authenticate/connect action.
5. Complete the browser OAuth flow.
6. Enter your Kindle email.
7. Read the OTP from the Rails logs:

   ```text
   [MCP OTP] you@kindle.com: <code>
   ```

   In production the code arrives on your Kindle as `otp.epub`.

8. Submit the OTP and approve access.

Send a document:

```text
/mcp__readitsoon__send
/mcp__readitsoon__send notes/plan.md
```

The command name follows the name used in `claude mcp add`. If the server was added as
`shortcut`, the command is `/mcp__shortcut__send`. You can also just ask: "send this to my
Kindle". Markdown, HTML, and plain text are supported, and each send uses one of your monthly
sends. When you're out of sends, the tool returns an error with your usage and an upgrade link
(`/mcp/checkout/<token>`, valid for 7 days) that opens Stripe Checkout for your Kindle email.
Paying subscribers are sent to the Stripe billing portal instead.

Remove the server:

```bash
claude mcp remove readitsoon
```

## Codex

Install the local MCP server:

```bash
codex mcp add readitsoon --url https://readitsoon.app/mcp
```

Authenticate:

```bash
codex mcp login readitsoon
```

Check the server:

```bash
codex mcp list
codex mcp get readitsoon
```

You can also inspect the active MCP servers from inside the Codex TUI:

```text
/mcp
```

Remove the server:

```bash
codex mcp logout readitsoon
codex mcp remove readitsoon
```

### Codex Config File Option

Codex stores MCP servers in `config.toml`. Add this to `~/.codex/config.toml` for a user-wide
install:

```toml
[mcp_servers.readitsoon]
url = "https://readitsoon.app/mcp"
```

Then authenticate:

```bash
codex mcp login readitsoon
```

For a project-scoped install, add the same table to `.codex/config.toml` in a trusted project.
Do not commit a localhost-only project config unless everyone using the repo should point at the
same local MCP URL.

If OAuth login fails because the client used an unexpected redirect URI, force a loopback
`/callback` URL in `~/.codex/config.toml`:

```toml
mcp_oauth_callback_port = 4321
mcp_oauth_callback_url = "http://127.0.0.1:4321/callback"

[mcp_servers.readitsoon]
url = "https://readitsoon.app/mcp"
```

Then retry:

```bash
codex mcp login readitsoon
```

## Smoke Test

After authentication, ask either client to call the tool with the sample fixture:

```text
Read fixtures/sample-article.md, then use the readitsoon MCP tool
send_to_kindle. Pass content as the full file contents, filename as
"The Quiet Hour Before Dawn.md", and author as "Guillermo Siliceo".
```

Expected tool response:

```text
Sending 'The Quiet Hour Before Dawn' to you@kindle.com. 9 sends left this period.
```

Expected worker log:

```text
[MailgunEmailClient] delivering 'The Quiet Hour Before Dawn' to you@kindle.com (The Quiet Hour Before Dawn.epub)
```

## References

- Claude Code MCP docs: https://docs.anthropic.com/en/docs/claude-code/mcp
- Codex MCP docs: https://developers.openai.com/codex/mcp
