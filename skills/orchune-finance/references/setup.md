# First-time setup: connect and authorize

Read this when the user has never connected to Orchune from this agent, when no access token is configured, or when your token stopped working. Goal: a working MCP connection with the token stored in an environment variable, verified by a successful `get_my_profile` call.

## 1. Register the MCP server

Key configuration — apply it however your host environment registers MCP servers (CLI command, JSON config file, UI form, ...). You know your own environment best; only these facts matter:

| Setting | Value |
|---|---|
| Server name | `orchune` (any name works) |
| URL | `https://www.orchune.com/mcp` |
| Transport | MCP Streamable HTTP (stateless JSON; no session handshake, no SSE required) |
| Auth header | `Authorization: Bearer ${ORCHUNE_ACCESS_TOKEN}` |

Prefer referencing the token through the `ORCHUNE_ACCESS_TOKEN` environment variable if your client supports variable expansion in config; otherwise place the literal token in the header field of the client's config — never anywhere else. Registering first without the auth header is fine: the server serves the signup/login tools to anonymous connections.

## 2. Authorize

Ask the user which situation applies, then follow one path:

### Path A — the user already has an access token

The user copies it from **Settings → Integrations** in the Orchune web app (tokens look like `orchune_sk_...`). Go to step 3 to store it.

### Path B — no token, or no account yet (email-code flow)

Works for both signup and login; no password is ever involved.

1. Connect **without** the Authorization header. `tools/list` will show exactly two tools: `start_email_verification` and `complete_email_verification`.
2. Ask the user for their email address. For a brand-new account also ask (or infer from the conversation) preferred `language` (en|zh), `currency`, and `timezone` — these only apply when the account doesn't exist yet.
3. Call `start_email_verification`. It always replies with the same generic "code sent" message; that is by design and not a confirmation the email exists.
4. Ask the user to read the 6-digit code from their inbox (sender: Orchune). The code lasts 10 minutes and dies after 5 wrong attempts.
5. Call `complete_email_verification(email, code)`. The result contains `accessToken` — **shown exactly once**. Store it immediately (step 3), before doing anything else.
6. If `replacedExistingToken` is true, tell the user their previous agent token has been revoked — any other agent using it must be updated.

## 3. Store the token and reload the connection

1. Save the token into the `ORCHUNE_ACCESS_TOKEN` environment variable so it persists and stays out of conversation logs — e.g. the user's shell profile, a `.env` file the client loads, or the client's own secrets/env mechanism. If the client can't read env vars in its MCP config, put the literal token in the config's header field instead.
2. Update the Orchune server entry so it sends `Authorization: Bearer <token>` (via the env var where possible).
3. **Ask the user to restart or refresh the MCP connection** — most clients only apply header/env changes on reconnect (restart the agent, reload its MCP servers, or toggle the server off and on). The signup/login tools disappearing and the full toolset (~34 tools) appearing confirms the reload took effect.

## 4. Verify

Call `get_my_profile`. Success confirms the token works and gives you the user's language, primary currency, and timezone — remember these; they drive date interpretation and display currency for everything else in the skill.

## Token handling rules

- The token lives in the environment variable / client config only. Do not echo it back into the conversation, logs, or files, and never include it in tool arguments — it belongs in the HTTP header only.
- One token per user: generating a new one (via this flow or the web Settings page) silently revokes the old one everywhere.
- The token does not expire on a schedule; it dies only when replaced or deleted by the user.

## Troubleshooting

| Symptom | Meaning | Fix |
|---|---|---|
| 401 with a token configured | Token revoked or replaced | Do not retry the same token. Re-run Path B, or ask the user for a fresh token from Settings → Integrations |
| 401 mentioning the header format | Header malformed | Must be exactly `Authorization: Bearer orchune_sk_...` |
| Only 2 tools visible | Connected anonymously | Expected before authorization; after storing the token, reload the MCP connection |
| Full toolset still missing after adding the token | Client hasn't reloaded | Restart the agent or refresh/re-register the MCP server — header changes rarely apply live |
| 429 | Rate limited (per-user or per-IP) | Wait for the `Retry-After` seconds; do not tight-loop |
| `INVALID_CODE` | Code expired / wrong / attempts exhausted | Request a new code; mind the 1 email/minute limit |
| No verification email arriving | Wrong address, or throttled (max 10/day per address) | Confirm the address with the user; check spam; wait a minute before re-requesting |
| HTTP 406 or Accept-header errors | Client sent a restrictive Accept header | The server normalizes this automatically; if your client still fails, send `Accept: application/json, text/event-stream` |
