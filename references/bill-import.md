# Statement import workflow

You parse the statement file (CSV, PDF, screenshot, OFX, ...) into structured rows yourself; the server only validates and imports them. The flow is plan → validate → execute:

1. Resolve the destination account with `list_accounts`. **All rows in one batch go into a single account**, in that account's currency. For a credit-card account, resolve the `card` too if the statement belongs to a specific card.
2. Optionally resolve categories with `get_category_list` and map statement lines to them. Leave `category` off rather than guessing wrong — the user can categorize later.
3. Call `preview_bill_import` with the rows (max **500 per call**; split larger statements into multiple batches). Nothing is written to transactions yet.
4. Inspect the per-row results: `READY`, `DUPLICATE`, or `INVALID` with field-level `errors[] {field, code, message}`. Fix your parsing for every INVALID row and re-preview. Loop until `readyCount` matches what you expect.
5. Call `commit_bill_import` with the returned `batchId`. DUPLICATE rows are skipped unless you pass `includeDuplicates: true` — only do that after the user explicitly confirms they are not duplicates.
6. Report the result and tell the user the batch can be reverted from the web app's import history page (there is no revert tool).

## Row schema

| Field | Required | Notes |
|---|---|---|
| `date` | yes | ISO 8601 with offset, or local `yyyy-MM-dd[ HH:mm[:ss]]` interpreted in the request `timezone` |
| `amount` | yes | Positive, major units. Direction comes from `type`, never the sign — flip negative statement amounts to positive and set `type` |
| `type` | yes | EXPENSE, INCOME, REFUND, or FEE |
| `payee` | no | ≤100 chars, auto-created |
| `description` | no | ≤500 chars |
| `category` | no | Name or id; must match the row's direction |
| `externalId` | no | ≤128 chars. **Strongly recommended** — the statement's own transaction id/reference number. Dedup keys on it; without it a weaker date+amount+payee fingerprint is used |
| `channel` | no | ≤100 chars, raw payment-channel text from the statement |

## Parsing tips

- Statements list refunds and fees as separate line types — map them to `REFUND` / `FEE`, don't fold them into EXPENSE/INCOME.
- Skip statement rows that are not transactions (balance lines, subtotals, repayment reminders).
- A credit-card repayment appearing in the card statement is the transfer's receiving leg; if the user also tracks the paying bank account, prefer recording it once as a transfer via `record_transfer` instead of importing it as INCOME on the card.
- Keep the original statement's transaction reference as `externalId` so re-importing the same file is a no-op (rows come back DUPLICATE).
