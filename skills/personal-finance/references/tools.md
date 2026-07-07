# Orchune MCP tool reference

Complete catalog of the tools exposed at `https://www.orchune.com/mcp`. Required fields are marked `*`. Shared conventions (major-unit inputs, minor-unit string outputs, date/timezone rules, name-or-id resolution) are in SKILL.md and not repeated per tool.

## Auth (anonymous server only)

These two tools are exposed only when connecting without an Authorization header; the authenticated server never shows them.

### start_email_verification
Signup and login in one flow: creates the account if the email is unknown, then emails a 6-digit code either way.
- `email`* — the user's email address.
- `name`, `language` (en|zh, default zh), `currency` (ISO 4217, default CNY), `timezone` (IANA) — used only when creating a new account.

### complete_email_verification
Exchanges the code for an access token. Token is returned exactly once; any previous token is revoked (`replacedExistingToken`). Code: 10-minute expiry, 5 attempts.
- `email`*, `code`* (exactly 6 digits).

## Read

### get_my_profile
Profile (id, email, name, created) + settings (language, primaryCurrency, timezone, commonCurrencies). No input.

### list_accounts
Accounts with type, currency, balances (minor-unit strings).
- `includeArchived` (bool, default false).

### get_account
One account's detail incl. `cards[]` for credit cards (card numbers masked).
- `account`* — name or id.

### get_category_list
Categories for one transaction type, with ids for use in write tools.
- `type`* — EXPENSE | INCOME.

### search_payees
Partial-name search over existing payees; reuse the returned full name.
- `query`*, `limit` (1–50, default 10).

### list_transactions
Rows plus totals (income, expense, count). Supports fuzzy search and amount-range filters for locating specific transactions.
- `startDate`/`endDate` (YYYY-MM-DD, user timezone, endDate inclusive; default current month), `type` (array of EXPENSE|INCOME|TRANSFER, default expense+income), `account`, `category`, `limit` (1–100, default 50), `offset`.
- `payee` — case-insensitive substring match on payee names.
- `keyword` — fuzzy search across notes AND payee names (OR); combine with a date range for speed.
- `minAmount`/`maxAmount` — major units, compared against the absolute amount ("expenses over 500").
- A `category`/`payee` filter that matches nothing returns an empty result (not an unfiltered one).

### list_budgets
Current-period budget progress per line: limit/actual/remaining (major units), usedPct, status normal|warning|over, plus a total line. Each line carries `id` (the itemId for set_budget_override) and `categoryId`. No input.

### list_goals
Savings goals vs current linked-account balances: target/current/remaining, progressPct, achieved, optional targetDate/daysLeft. No input.

## Settings

### update_my_settings
- `primaryCurrency` (ISO 4217) and/or `timezone` (IANA); at least one. Timezone change recalculates monthly statistics.

## Accounts

### create_account
- `name`* (unique, case-insensitive), `type`* (GENERAL|BANK_CARD|CREDIT_CARD|DEBT), `currency`* (immutable afterwards), `openingBalance` (major units, may be negative, default 0), `paymentProvider`, `notes`.

### update_account
Name/provider/notes/archive only — type, currency, balances are immutable here.
- `account`* (may reference an archived account), `name`, `paymentProvider`, `notes` ("" clears), `archived` (bool). At least one change.

### calibrate_account
Sets the balance to the real-world figure the user reports; the difference is booked as an ADJUSTMENT transaction excluded from statistics. No-op when already matching. Use this instead of a fake expense/income to fix a drifted balance.
- `account`*, `actualBalance`* (major units, may be negative), `note`.

## Categories

### create_category
- `name`* (unique among siblings; matching an archived name reactivates it), `type`*, `parent` (main category of same type), `icon`, `color`.

### update_category
Rename / icon / color. System categories protected; type and parent immutable.
- `category`*, `type`*, plus at least one of `name`/`icon`/`color`.

### delete_category (destructive)
Archives the category; existing transactions keep referencing it. Fails on system categories or categories with active subcategories.
- `category`*, `type`*. Output includes `transactionCount`.

### merge_categories (destructive, irreversible)
Moves all transactions, splits, recurring rules, merchant preferences and subcategories from source into target, then archives source.
- `source`*, `target`* (active, same type, not a subcategory of source), `type`*.

## Budgets

Amounts in budget tools are major units of the budget's currency. Lines: at most one TOTAL (overall monthly cap) plus distinct CATEGORY lines (EXPENSE categories, name or id).

### create_budget
- `currency`*, `items`* (array of `{kind: TOTAL|CATEGORY, category (CATEGORY only), amount, rollover, alertThresholdPct, notes}`), `name`, `includePending`.

### update_budget
`items`, when provided, FULLY REPLACES the line set — send the complete list, not just the changed line. For a one-month change use set_budget_override instead.
- `budgetId`*, `name` (null clears), `status` (ACTIVE|PAUSED|ARCHIVED), `includePending`, `items`.

### set_budget_override
One-month override of a single line; the recurring template is untouched. Omit both `amount` and `skip` to clear an existing override.
- `budgetId`*, `itemId`* (line `id` from list_budgets), `periodKey`* (YYYY-MM), `amount`, `skip` (bool), `notes`.

### delete_budget (destructive)
Permanently removes the plan, its lines and overrides; transactions untouched. Prefer archiving via update_budget.
- `budgetId`*.

## Goals

Amounts in goal tools are major units of the goal's currency. A goal is linked 1:1 to an account; SAVINGS tracks its balance toward a target, DEBT pays a liability account down to zero.

### create_goal
- `name`*, `currency`* (must match the account), `target`*, `account`* (each account backs at most one goal; immutable for DEBT), `kind` (SAVINGS default | DEBT), `monthlyPlan`, `targetDate` (YYYY-MM-DD).

### update_goal
Archive with `status: ARCHIVED` (funds untouched), restore with `status: ACTIVE`.
- `goalId`*, plus any of `name`, `status`, `target`, `monthlyPlan`, `targetDate` (null clears), `account` (same currency; not for DEBT).

### fund_goal
Moves real money: a transfer from `sourceAccount` into the goal's account (SAVINGS deposit / DEBT repayment; `interest` books as an expense on top).
- `goalId`*, `amount`*, `sourceAccount`* (same currency, differs from the goal's account), `interest`, `note`.

### delete_goal (destructive)
Removes the goal definition only; accounts and transactions untouched. Prefer archiving.
- `goalId`*.

## Transactions

Shared optional fields on the record/update tools: `card` (credit-card accounts), `date`, `timezone`, `notes` (≤500), `status` (PENDING|CLEARED|RECONCILED, default CLEARED). The three record tools also take `clientRequestId` (≤64 chars) — an idempotency key; see SKILL.md conventions.

### record_expense / record_income
- `account`*, `amount`* (major units, positive), `category` (type must match), `payee` (auto-created; search first), `clientRequestId`.

### record_transfer
Same-currency transfer between two of the user's accounts.
- `fromAccount`*, `toAccount`*, `amount`*, `fee` (optional, charged from source; requires a fee category to exist), `clientRequestId`.

### update_transaction
Cannot edit transfers (either leg) — web app only.
- `transactionId`* (UUID), plus any of `account`, `amount`, `category`, `payee`, `card`, `date`, `timezone`, `notes`.

### delete_transaction
Deleting one leg of a transfer deletes the whole transfer.
- `transactionId`* (UUID).

### split_transaction
Replaces one EXPENSE transaction with 2–20 category lines (e.g. a supermarket receipt into groceries + household). Amounts must sum exactly to the original; balance unchanged; the lines inherit date/payee/notes/etc.
- `transactionId`* (UUID), `splits`* (array of `{category, amount}` in major units).

## Bill import

See [bill-import.md](bill-import.md) for the full workflow and row schema.

### preview_bill_import
Validates agent-parsed statement rows into a pending batch; writes nothing to transactions. Returns `batchId` and per-row READY/DUPLICATE/INVALID with field-level errors.
- `account`*, `transactions`* (1–500 rows), `card`, `timezone`, `fileName`.

### commit_bill_import
Imports the READY rows of a previewed batch; DUPLICATE rows skipped unless `includeDuplicates: true` (only after the user confirms).
- `batchId`*, `includeDuplicates` (default false).

### list_import_batches
Paged import history (newest first) with per-batch status and counts (total/imported/skipped).
- `page` (default 1), `pageSize` (1–50, default 10).

### revert_import_batch (destructive)
Undoes a COMPLETED import: permanently deletes the transactions it created (balances/stats restored) and clears dedup references so the same file can be re-imported. Confirm with the user first.
- `batchId`*.

## Investments — stock

### open_stock_position
New holding with an opening buy lot; fails if the symbol already has a position.
- `symbol`*, `quantity`*, `price`* (per share), `currency`, `name`, `market`, `exchange`, `sector`, `fee`, `date`, `timezone`, `notes`.

### record_stock_trade
- `position`* (id or symbol), `action`* (BUY|SELL|DIVIDEND), `quantity`+`price` (BUY/SELL), `amount` (DIVIDEND), `fee`, `date`, `timezone`, `notes`.

### list_stock_positions
Holdings with quantity, avg cost, price, market value, P&L. No input.

## Investments — options

### open_option_position
OCC contract code is generated from underlying/expiry/type/strike; multiplier defaults to 100. LONG = buy-to-open, SHORT = sell-to-open.
- `underlyingSymbol`*, `optionType`* (CALL|PUT), `side`* (LONG|SHORT), `expiryDate`* (YYYY-MM-DD), `strikePrice`*, `quantity`* (contracts), `premium`* (per share), `currency`, `contractMultiplier`, `fee`, `date`, `timezone`, `notes`.

### record_option_trade
- `position`* (id or OCC code, e.g. AAPL240119C00150000), `action`* (BUY_OPEN|SELL_OPEN|BUY_CLOSE|SELL_CLOSE|EXERCISE|ASSIGN|EXPIRE), `quantity`+`premium` (open/close actions), `amount` (settlement cash for EXERCISE/ASSIGN when not booking the stock leg), `settleIntoUnderlying` (EXERCISE/ASSIGN, default true — also books the underlying stock trade at strike), `fee`, `date`, `timezone`, `notes`.

### list_option_positions
Contract details, premium, market value, P&L. No input.

## Investments — funds

### open_fund_position
- `code`* (e.g. 000001), `quantity`*, `price`* (NAV per share), `valuationType` (NAV_FLUCTUATION default | SHARE_ACCRUAL for money-market funds), `currency`, `name`, `fee`, `date`, `timezone`, `notes`.

### record_fund_trade
- `position`* (id or code), `action`* (BUY|SELL|DIVIDEND|INCOME), `quantity`+`price` (BUY/SELL/INCOME), `amount` (DIVIDEND), `fee`, `date`, `timezone`, `notes`. Do not use INCOME for money-market earnings — use record_fund_return.

### record_fund_return
Books accrued earnings into the fund's return calendar (SHARE_ACCRUAL: reinvests as shares; NAV funds: journal-only). Upserts by date.
- `position`*, `amount`* (major units, may be negative), `periodType` (DAILY default | MONTHLY), `date`, `timezone`, `note`.

### delete_fund_return (destructive)
- `position`*, `date`* (`YYYY-MM-DD` | `YYYY-MM` | `YYYY`), `periodType` (omit = both).

### get_fund_returns
- All optional: `position` (omit = all funds), `date` (day/month/year), `periodType`. Returns entries + per-fund totals + overall total.

### list_fund_positions
Holdings with quantity, avg cost, NAV, market value, P&L, valuationType. No input.

## Error codes worth knowing

- `*_NOT_FOUND` / `*_AMBIGUOUS` — resolution failures; ambiguous errors list candidates, retry with the id.
- `CATEGORY_TYPE_MISMATCH` — category type vs transaction type.
- `CROSS_CURRENCY_TRANSFER_UNSUPPORTED`, `SAME_ACCOUNT_TRANSFER`, `FEE_CATEGORY_REQUIRED` — transfer constraints.
- `CATEGORY_SYSTEM_PROTECTED`, `CATEGORY_HAS_SUBCATEGORIES` — category management constraints.
- `CARD_ON_NON_CREDIT_ACCOUNT`, `CARD_NOT_FOUND` — card field misuse (the latter lists available cards).
- `FUND_POSITION_EXISTS` — use record_fund_trade instead of open_fund_position.
- `INVALID_DATE`, `INVALID_TIMEZONE` — fix the format and retry.
- `INVALID_CODE`, `RATE_LIMITED` — auth flow; request a fresh code / back off.
- `BUDGET_NOT_FOUND`, `BUDGET_ITEM_NOT_FOUND` — use ids from list_budgets (line `id` = itemId).
- `GOAL_NOT_FOUND`, `GOAL_ACCOUNT_TAKEN`, `GOAL_DEBT_ACCOUNT_IMMUTABLE`, `ACCOUNT_CURRENCY_MISMATCH` — goal/account pairing constraints.
- `TRANSACTION_NOT_SPLITTABLE`, `SPLIT_FX_NOT_SUPPORTED`, `SPLIT_AMOUNT_MISMATCH` — split constraints (plain EXPENSE only; amounts must sum exactly).
- `IMPORT_BATCH_NOT_FOUND`, `IMPORT_BATCH_NOT_REVERTIBLE` — only COMPLETED batches revert.
- `IDEMPOTENCY_KEY_CONFLICT` — the clientRequestId was already used by a different tool; generate a fresh id per logical write.

Error results also carry `structuredContent.errorCode` with the machine-readable code, so you don't need to parse the message text.
