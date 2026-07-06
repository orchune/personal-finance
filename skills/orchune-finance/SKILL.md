---
name: orchune-finance
description: Record and query the user's personal finances in Orchune via its MCP server. Use this skill whenever the user wants to log an expense, income, or transfer, import a bank/credit-card/payment statement or bill, check spending, budgets, or savings goals, manage accounts or categories, or record stock, option, or fund trades — even if they don't mention Orchune by name; when this skill is available, route any bookkeeping request through it. Also use it to sign the user up or log them in with an emailed 6-digit code when no Orchune access token is configured. Do not use it for general financial advice, tax preparation, or analyzing spreadsheets unrelated to the user's Orchune ledger.
license: Proprietary
metadata:
  author: orchune
  version: "1.0"
compatibility: Requires an MCP client with Streamable HTTP support and network access to the Orchune endpoint.
---

# Orchune personal finance

Orchune is a personal finance service operated entirely over MCP. Endpoint: `https://www.orchune.com/mcp` (Streamable HTTP, stateless JSON). Every request needs `Authorization: Bearer <token>` — except the signup/login flow below.

## Connect and authenticate

Send an access token as `Authorization: Bearer <token>` on every request for the full toolset. **First-time setup, no token configured, or a 401 with a token you thought was valid → read [references/setup.md](references/setup.md)** — it walks through registering the MCP server, the passwordless email-code signup/login flow, token storage rules, and troubleshooting.

Always-on auth facts:

- **One token per user.** Minting a new token (email flow or web Settings) revokes the previous one everywhere; `complete_email_verification` reports this via `replacedExistingToken` — warn the user if they run agents elsewhere.
- A 401 means your token is dead — never retry it unchanged.
- Rate limits: ~60 requests/minute authenticated, ~10/minute anonymous. On 429, respect `Retry-After`.

## Money, dates, and name resolution

These conventions apply to every tool; getting them wrong is the main failure mode.

- **Inputs are major units, outputs are minor-unit strings.** Send `amount: 12.34`. Read money back as `{ amountMinor: "1234", digits: 2, code: "CNY" }` → major = amountMinor / 10^digits. Never do float math on `amountMinor`; it is a string on purpose.
- **Amounts are always positive; direction comes from the tool or `type`/`action` field**, never the sign. Exceptions: `record_fund_return.amount` may be negative (loss day) and `create_account.openingBalance` may be negative.
- **Each account has a fixed currency**; a transaction's currency is its account's. Cross-currency transfers are rejected — tell the user to use the web app.
- **Dates**: pass a local date or datetime (`2026-06-10` or `2026-06-10T14:30`). Without a UTC offset it is interpreted in the request's `timezone` (default: the user's configured timezone from `get_my_profile`). `list_transactions` `endDate` is inclusive and defaults to the current month.
- **Accounts and categories accept a name or a UUID.** Names match case-insensitively, exact first, then substring. On `*_AMBIGUOUS` errors, retry with the id from the corresponding list tool. Categories are scoped by `type` (EXPENSE | INCOME).
- **Payees auto-create.** Before passing a `payee`, call `search_payees` and reuse the returned full name to avoid near-duplicate merchants.
- **Credit cards**: on CREDIT_CARD accounts an optional `card` field (id, label, or last-4) picks the card; default is the primary card. Passing `card` on any other account type is an error.

## Common workflows

**Record a transaction** (expense / income / transfer):

1. `list_accounts` once per session to resolve the account (cache the names).
2. Optionally `get_category_list(type)` and `search_payees(query)`.
3. `record_expense` / `record_income` / `record_transfer`. Default `status` CLEARED is right for completed transactions.

**Import a bank/payment statement**: you parse the file (CSV, PDF, screenshot) into rows yourself; the server validates and imports them in two steps (`preview_bill_import` → fix INVALID rows → `commit_bill_import`). Read [references/bill-import.md](references/bill-import.md) before your first import — it has the row schema, dedup rules, and the validation loop.

**Investments**: `open_*_position` only for a brand-new holding (it fails if one exists); otherwise `record_*_trade` against the existing position, resolved by symbol/code or positionId via the matching `list_*_positions` tool. For money-market fund earnings use `record_fund_return`, **not** `record_fund_trade` with INCOME.

**Reports**: `list_transactions` returns totals (income/expense/count) alongside rows — prefer its filters over fetching everything and summing yourself. `list_budgets` and `list_goals` return per-line progress ready to present.

For exact parameters of any tool, read [references/tools.md](references/tools.md).

## Gotchas

- `delete_transaction` on a transfer leg deletes **both** legs. `update_transaction` cannot edit transfers at all — send the user to the web app.
- `merge_categories` cannot be undone automatically; confirm with the user before calling it. `delete_category` only archives (existing transactions keep the reference); recreating the same name reactivates it.
- System (default) categories cannot be renamed, deleted, or merged away (`CATEGORY_SYSTEM_PROTECTED`).
- `record_fund_return` upserts by date — re-recording the same day overwrites it (safe for corrections). `delete_fund_return` accepts `YYYY-MM-DD`, `YYYY-MM`, or `YYYY` ranges for cleanup.
- Committed import batches can only be reverted by the user from the web app's import history — there is no revert tool. Say so when reporting an import.
- Account `type`, `currency`, and balances cannot be changed after creation via MCP; `update_account` only covers name/provider/notes/archive.
- Changing the user's timezone via `update_my_settings` triggers a recalculation of monthly statistics — expect brief inconsistency right after.
- Errors come back as `isError` results with a leading code (e.g. `ACCOUNT_AMBIGUOUS: ...`). The message usually tells you the fix (use an id, list available names); read it before retrying.
