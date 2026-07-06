# First-time setup: connect and authorize

Read this when the user has never connected to Orchune from this agent, when no access token is configured, or when your token stopped working. Goal: a working MCP connection with a stored access token, verified by a successful `get_my_profile` call.

## 1. Register the MCP server

Endpoint: `https://www.orchune.com/mcp` — MCP Streamable HTTP, stateless JSON. No session handshake is required; every request stands alone.

Claude Code:

```bash
# Without a token yet (signup/login tools only):
claude mcp add --transport http orchune https://www.orchune.com/mcp

# With a token:
claude mcp add --transport http orchune https://www.orchune.com/mcp \
  --header "Authorization: Bearer <token>"
```

Generic JSON config (Claude Desktop-style clients via a Streamable HTTP entry, or any client that takes url + headers):

```json
{
  "orchune": {
    "type": "http",
    "url": "https://www.orchune.com/mcp",
    "headers": { "Authorization": "Bearer <token>" }
  }
}
```

If your client only lets you set headers at registration time, you will re-register once more after obtaining the token in step 2 — that is expected.

## 2. Authorize

Ask the user which situation applies, then follow one path:

### Path A — the user already has an access token

The user copies it from **Settings → Integrations** in the Orchune web app (tokens look like `orchune_sk_...`). Add it as the `Authorization: Bearer <token>` header and go to step 3.

### Path B — no token, or no account yet (email-code flow)

Works for both signup and login; no password is ever involved.

1. Connect **without** an Authorization header. `tools/list` will show exactly two tools: `start_email_verification` and `complete_email_verification`.
2. Ask the user for their email address. For a brand-new account also ask (or infer from the conversation) preferred `language` (en|zh), `currency`, and `timezone` — these only apply when the account doesn't exist yet.
3. Call `start_email_verification`. It always replies with the same generic "code sent" message; that is by design and not a confirmation the email exists.
4. Ask the user to read the 6-digit code from their inbox (sender: Orchune). The code lasts 10 minutes and dies after 5 wrong attempts.
5. Call `complete_email_verification(email, code)`. The result contains `accessToken` — **shown exactly once**. Store it in the client's MCP config immediately, before doing anything else.
6. If `replacedExistingToken` is true, tell the user their previous agent token has been revoked — any other agent using it must be updated.
7. Reconnect with `Authorization: Bearer <token>`. The signup/login tools disappear and the full toolset appears.

If the code fails (`INVALID_CODE`): it is expired, mistyped too many times, or the email has no pending code. Re-run `start_email_verification` (at most one email per minute per address) and try again with the fresh code.

## 3. Verify

Call `get_my_profile`. Success confirms the token works and gives you the user's language, primary currency, and timezone — remember these; they drive date interpretation and display currency for everything else in the skill.

## Token handling rules

- Store the token only in the MCP client configuration. Do not echo it back into the conversation, logs, or files, and never include it in tool arguments — it belongs in the HTTP header only.
- One token per user: generating a new one (via this flow or the web Settings page) silently revokes the old one everywhere.
- The token does not expire on a schedule; it dies only when replaced or deleted by the user.

## Troubleshooting

| Symptom | Meaning | Fix |
|---|---|---|
| 401 with a token configured | Token revoked or replaced | Do not retry the same token. Re-run Path B, or ask the user for a fresh token from Settings → Integrations |
| 401 mentioning the header format | Header malformed | Must be exactly `Authorization: Bearer orchune_sk_...` |
| Only 2 tools visible | You are connected anonymously | Expected before authorization; reconnect with the Bearer header after step 2 |
| 429 | Rate limited (per-user or per-IP) | Wait for the `Retry-After` seconds; do not tight-loop |
| `INVALID_CODE` | Code expired / wrong / attempts exhausted | Request a new code; mind the 1 email/minute limit |
| No verification email arriving | Wrong address, or throttled (max 10/day per address) | Confirm the address with the user; check spam; wait a minute before re-requesting |
| HTTP 406 or Accept-header errors | Client sent a restrictive Accept header | The server normalizes this automatically; if your client still fails, send `Accept: application/json, text/event-stream` |
