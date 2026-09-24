# Read It Soon MCP

Send Markdown, HTML, or plain text from Claude to your Kindle. [Read It Soon](https://readitsoon.app)
converts the document to an EPUB and emails it to your Kindle address.

## Before you start

Amazon only delivers documents from approved senders. Add this address to your
Approved Personal Document E-mail List in [Manage Your Content and Devices](https://www.amazon.com/mycd)
(Preferences → Personal Document Settings):

```text
sending@readitsoon.app
```

You'll also need your Kindle email address (the `@kindle.com` one, listed on the same page).

## Install

```bash
claude mcp add --transport http readitsoon https://readitsoon.app/mcp
```

Add `--scope user` to make it available in every project, not only the current one.

## Sign in

1. Start `claude` and run `/mcp`.
2. Select `readitsoon` and choose **Authenticate**. A browser window opens.
3. Enter your Kindle email address.
4. A one-time code arrives on your Kindle as a document named `otp`. Type it into the browser.
5. Approve access and return to Claude.

The code expires after 1 hour. You can request a new one after 30 seconds.

## Send a document

```text
/mcp__readitsoon__send
/mcp__readitsoon__send notes/plan.md
```

Without an argument, Claude sends the most relevant document in the conversation. You can also just
ask: "send this to my Kindle".

- **Formats:** Markdown, HTML, and plain text. The format comes from the file extension
  (`.md`, `.html`, `.txt`) unless Claude says otherwise.
- **Title:** the filename without its extension.
- **Usage:** each send uses one of your monthly sends, and the reply tells you how many are left.
  Sending the same document twice within a minute counts once.

The command name comes from the name you used in `claude mcp add`. If you named the server
`kindle`, the command is `/mcp__kindle__send`.

## Running out of sends

When you hit your monthly limit, the send fails with your usage and an upgrade link:

```text
Monthly sending limit reached (10/10 used). Upgrade to keep sending: https://readitsoon.app/mcp/checkout/…
```

The link opens Stripe Checkout for your Kindle email and stays valid for 7 days. After paying, go
back to Claude and send again. If you already subscribe, the link opens the billing portal instead.

## Other clients

**Codex**

```bash
codex mcp add readitsoon --url https://readitsoon.app/mcp
codex mcp login readitsoon
```

Any MCP client that supports streamable HTTP and OAuth with a loopback (`http://localhost/…/callback`)
redirect works the same way. Claude.ai and Claude Desktop connectors aren't supported yet.

## Troubleshooting

- **No code on the Kindle:** check that `sending@readitsoon.app` is on your approved sender list,
  then request a new code.
- **"Too many attempts":** 10 wrong codes block your IP for 24 hours.
- **Signed out:** run `/mcp`, select `readitsoon`, and authenticate again.
- **Remove it:** `claude mcp remove readitsoon`

## Development

To run against a local copy of the server (`http://localhost:4001/mcp`), see
[MCP_INSTALL.md](MCP_INSTALL.md).
