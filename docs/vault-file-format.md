# Vault File Format

Basalt stores your budget as a folder of plain YAML files — a **vault**. You own that folder; it works with git, Syncthing, Dropbox, or any file sync you like, and you can read or edit it by hand.

This document is the reference for the vault format: the layout, and the files created lazily at runtime. It's generated from Basalt's internal types and services (not included in this repo, which covers docs and community issues only), so file paths cited below (e.g. `packages/apple/BasaltKit/Sources/BasaltModels/Types`) are internal pointers for context, not links you can follow here.

## Overview

A Basalt vault is a plain directory containing YAML entity files and a small amount of JSON
metadata. Files can be read and edited without Basalt, but manual edits must preserve IDs,
references, data types, and YAML document separators.

The iOS and macOS apps open a folder the user picks in the system folder picker and save a
security-scoped bookmark to reopen it at the next launch. Creating a vault makes a new folder
inside the picked folder, named from the vault name (`My Vault` -> `my-vault`). Operations that
require vault data fail with `vault_unavailable` until a vault is open.

See `packages/apple/AppCore/Stores/VaultStore.swift`.

## Directory layout

```text
vault/
├── .basalt/
│   ├── vault.json
│   ├── migrations/
│   │   └── <provider>-<container-id>.json
│   └── bank-sync/
│       └── plaid-<item-id>-pending.json
├── accounts/
│   └── <sanitized-account-name>.yml
├── budgets/
│   └── YYYY-MM.yml
├── categories/
│   └── <sanitized-category-name>.yml
├── imports/
├── transactions/
│   └── YYYY-MM.yml
├── payees.yml
└── preferences.yml
```

The files and directories do not all exist in every vault:

- New-vault initialization creates `accounts/`, `budgets/`, `categories/`, `imports/`,
  `transactions/`, `.basalt/vault.json`, and `preferences.yml`.
- New-vault initialization seeds the default category files.
- Account, budget, and transaction files are created when their first entities are created.
- `payees.yml` is created lazily when the payee index is initialized or updated.
- `.basalt/migrations/` is created lazily when a migration or bank sync source map is written.
- `.basalt/bank-sync/` is created lazily when a bank sync has transactions waiting for review.
- `imports/` is currently an empty scaffold. Import jobs and uploaded documents are not stored
  there.

See `packages/apple/BasaltKit/Sources/BasaltBackend/Vault/VaultInitializer.swift`.

## YAML conventions

### Single-document files

Accounts, budgets, categories, `payees.yml`, and `preferences.yml` contain one YAML document
wrapped in `---` markers:

```yaml
---
key: value
---
```

### Multi-document transaction files

Each transaction is a separate YAML document. A monthly file begins and ends with `---`, with
documents separated by another `---`:

```yaml
---
id: tx_...
account_id: acc_...
# ...
---
id: tx_...
account_id: acc_...
# ...
---
```

The YAML writer uses:

- 2-space indentation
- no line wrapping
- no YAML aliases/references
- insertion order for keys; keys are not alphabetically sorted

Transactions are appended on creation. Updating a transaction in the same month preserves its
document position. Moving it to another month appends it to the destination file. Documents are
not sorted on disk; transaction queries sort in memory when needed.

See
`packages/apple/BasaltKit/Sources/BasaltBackend/Codec/YAMLWriter.swift`.

### Scalar types

Do not quote values merely to change their type:

- Numbers such as `49.43` are YAML numbers, not strings.
- Booleans are `true` or `false`.
- Empty arrays use `[]`.
- A missing parent category uses `null`.
- Empty text may be `''`.

### Date and time formats

Most entity timestamps use local date-time strings:

```text
YYYY-MM-DD HH:mm
```

Some fields use other formats:

- Month periods and monthly filenames: `YYYY-MM`
- Date-only values may appear as `YYYY-MM-DD`
- `preferences.updated_at`, `payees.yml` `updated_at`, and vault metadata use ISO 8601 strings
  such as `2026-07-10T18:58:09.306Z`
- Migration and bank sync source maps use the local `YYYY-MM-DD HH:mm` format, like entities

The parser accepts valid YAML scalars, but application code generally expects these exact string
formats. Quoting a timestamp is safe and prevents YAML libraries from interpreting it as a native
date value.

## IDs and filenames

### Entity IDs

Account, category, and transaction IDs use:

```text
<prefix><epoch-milliseconds>_<seven-base36-characters>
```

Prefixes:

- Account: `acc_`
- Category: `cat_`
- Transaction: `tx_`

Example:

```text
tx_1772577506616_jjjlrmy
```

Vault IDs use a similar `vault_<epoch-milliseconds>_<random>` format. IDs are opaque references;
code must not derive names, dates, or relationships from them.

See
`packages/apple/BasaltKit/Sources/BasaltBackend/Shared/SharedUtils.swift`.

### Name-based filenames

Account and category filenames are generated from the initial display name:

1. Convert to lowercase.
2. Trim leading and trailing whitespace.
3. Replace each whitespace run with `_`.
4. Remove every character except `a-z`, `0-9`, and `_`.
5. Append `.yml`.

Examples:

```text
Main Checking              -> main_checking.yml
Financial & Legal Services -> financial__legal_services.yml
```

Consequences:

- Names that sanitize to the same filename collide, and creation fails.
- Renaming an account or category changes its `name` field but does not rename its file.
- The filename is not an ID and does not need to match the current display name.
- Manually renaming a file is safe only if its contents and extension remain valid and no
  duplicate entity is introduced.

### Monthly filenames

- Budget filename: the budget's `period`, as `budgets/YYYY-MM.yml`.
- Transaction filename: the local calendar month derived from `transaction.date`, as
  `transactions/YYYY-MM.yml`.
- Changing a transaction date to another month moves the document between monthly files.
- When the last transaction leaves a month, the current writer leaves an empty `---` file rather
  than deleting it.

## Accounts

Path:

```text
accounts/<sanitized-initial-name>.yml
```

Example:

```yaml
---
id: acc_1770000000000_abc1234
name: Example Credit Card
type: credit_card
opening_balance: 49.43
current_balance: -2186.46
institution: Example Bank
credit_limit: 5000
opening_date: '2026-06-12'
last_synced: 2026-06-23 19:12
created_at: 2026-06-23 19:12
updated_at: 2026-07-09 11:20
most_recent_transaction_at: 2026-07-09 00:00
balance_last_set: 2026-07-06 17:56
notes: ''
exclude_from_budgets: false
exclude_from_net_worth: false
---
```

Fields:

- `id` — required string. Unique account ID with the `acc_` prefix.
- `name` — required string. Display name. Changing it does not rename the file.
- `type` — required enum:
  - `checking`
  - `savings`
  - `credit_card`
  - `investment`
  - `cash`
  - `loan`
  - `other`
- `institution` — optional string. Financial institution display name.
- `opening_balance` — required number. Historical balance anchor. It is rounded to two decimal
  places and normally locked after account creation.
- `opening_date` — required date/date-time string. Start of balance calculations. Account creation
  initially sets it to the creation time. Creating a transaction earlier than this date moves
  `opening_date` back to that transaction's date.
- `current_balance` — required number. User-verified balance anchor for the present. It is not
  automatically replaced whenever transactions change.
- `balance_last_set` — optional date-time string. Set when `current_balance` is manually changed.
  Balance calculations before this anchor use `opening_balance`; later calculations can use
  `current_balance`.
- `last_synced` — optional date-time string. Last account sync/update time. Account creation sets
  it to the creation time.
- `created_at` — required date-time string. Creation timestamp.
- `updated_at` — required date-time string. Last account update timestamp.
- `notes` — optional string. Free-form account notes.
- `credit_limit` — conditionally required positive number for `credit_card` and `loan` accounts.
  It is rounded to two decimal places.
- `exclude_from_budgets` — optional boolean. If `true`, transactions from this account do not
  contribute to budget actuals.
- `exclude_from_net_worth` — optional boolean. If `true`, the account is excluded from net-worth
  calculations and its transactions are excluded from spending widgets.
- `most_recent_transaction_at` — optional date-time string. Most recent transaction date used for
  “last used” sorting. Transaction create/update/delete operations maintain it, and account reads
  lazily backfill it when missing.

Balance math uses one sign convention for every account type: debits decrease a balance and
credits increase it. Negative balances represent debt, including for credit cards and loans.

Account balance values are anchors, not three interchangeable totals. For balance-calculation
behavior, see
`packages/apple/BasaltKit/Sources/BasaltBackend/Services/AccountService.swift`.

## Categories

Path:

```text
categories/<sanitized-initial-name>.yml
```

Example:

```yaml
---
id: cat_1770000000000_abc1234
name: Restaurant
icon: 🍔
type: expense
parent_id: null
created_at: 2026-02-04 23:32
updated_at: 2026-03-17 21:35
---
```

Fields:

- `id` — required string. Unique category ID with the `cat_` prefix.
- `name` — required string. Display name.
- `icon` — required string. Usually an emoji.
- `type` — required enum controlling budget behavior:
  - `income` — belongs in the income section. Credits add to actual income; debits subtract.
  - `expense` — belongs in the expense section. Debits add to spending; credits subtract as
    refunds.
  - `transfer` — excluded from budget calculations.
  - `debt_payment` — belongs in the expense section, but only debits contribute. Credits are
    ignored. A debit that carries `transfer_account_id` pointing at an on-budget
    `credit_card` account is excluded entirely — see *Transfer links* below.
- `parent_id` — required string or `null`. References another category ID for hierarchy;
  top-level categories use `null`.
- `created_at` — required date-time string.
- `updated_at` — required date-time string.

New vaults seed the categories listed in
`packages/apple/BasaltKit/Sources/BasaltBackend/Vault/VaultInitializer.swift`. Seeding
generates a new random `cat_` ID for every category.

The seed set is:

- Income: `Interests & Dividends`, `Investments`, `Other Income`, `Salary`
- Expenses: `Business`, `Check`, `Childcare`, `Electricity`, `Entertainment`,
  `Financial & Legal Services`, `Financial Fees`, `Restaurant`, `Gifts`, `Groceries`,
  `Healthcare`, `Home Improvement`, `Housing`, `Insurrance`, `Internet`, `Phone`,
  `Postage & Shipping`, `Shopping`, `Taxes`, `Taxi & Rides`, `Uncategorized`
- Transfer: `Transfers`
- Debt payment: `Credit Card Payment`

Seed filenames follow the same sanitizer as user-created categories. For example,
`Financial & Legal Services` becomes `financial__legal_services.yml`.

## Transactions

Path:

```text
transactions/YYYY-MM.yml
```

Each file contains zero or more transaction documents.

Example document:

```yaml
---
id: tx_1770000000000_abc1234
account_id: acc_1770000000000_def5678
category_id: cat_1770000000000_ghi9012
amount: 61.94
type: debit
date: 2026-03-03 00:00
description: ''
payee: Example Store
status: cleared
tags:
  - recurring
notes: ''
created_at: 2026-03-03 17:38
updated_at: 2026-03-03 17:38
---
```

Fields:

- `id` — required string. Unique transaction ID with the `tx_` prefix.
- `account_id` — required string. References the account that owns the transaction.
- `category_id` — required string. References its category.
- `amount` — required positive number. Direction is represented by `type`; do not store debits as
  negative amounts.
- `type` — required enum:
  - `debit` — money leaving the account
  - `credit` — money entering the account
- `date` — required date-time string. Determines the monthly filename. Invalid calendar dates may
  be corrected by the transaction service.
- `description` — optional string. Memo or description.
- `payee` — required string. Payee or payer name.
- `status` — required enum:
  - `pending` — not yet cleared or posted
  - `cleared` — cleared the account
  - `reconciled` — reconciled with a statement
- `transfer_account_id` — optional string or `null`. Counterpart account when this transaction
  moves money between the user's own accounts, such as the credit card settled by a payment
  made from a chequing account. Absent or `null` means the transaction is not a transfer. See
  *Transfer links* under Budgets for the effect on budget totals.
- `tags` — optional string array used for additional grouping and filtering.
- `notes` — optional string. Additional free-form notes.
- `created_at` — required date-time string.
- `updated_at` — required date-time string.

Creation appends a transaction to the month file, updates the account's
`most_recent_transaction_at`, may move the account's `opening_date` earlier, and queues a
`payees.yml` update. Changing `date` moves the document when its month changes.

See
`packages/apple/BasaltKit/Sources/BasaltBackend/Services/TransactionService.swift`.

## Budgets

Path:

```text
budgets/YYYY-MM.yml
```

Example:

```yaml
---
period: 2026-07
created_at: 2026-07-01 19:08
updated_at: 2026-07-09 11:20
income:
  - category_id: cat_1770000000000_income1
    budgeted: 1700
    actual: 0
    remaining: 1700
expenses:
  - category_id: cat_1770000000000_expense
    budgeted: 400
    actual: 225.5
    remaining: 174.5
---
```

Top-level fields:

- `period` — required `YYYY-MM` string. It is also the budget identifier and filename.
- `created_at` — required date-time string.
- `updated_at` — required date-time string.
- `income` — required array of income-category allocations.
- `expenses` — required array of expense and debt-payment-category allocations.

Allocation fields:

- `category_id` — required string. References a category.
- `budgeted` — required number. User-assigned amount.
- `actual` — required number. Cached transaction contribution for the period.
- `remaining` — required number. `budgeted - actual`, rounded to two decimal places.

Actual calculations:

- Transactions from accounts with `exclude_from_budgets: true` are ignored.
- Transfer categories are ignored.
- Transactions that are budget-neutral transfers are ignored — see *Transfer links* below.
- Income actual is credits minus debits.
- Expense actual is debits minus credits, allowing refunds to reduce spending.
- Debt-payment actual includes debits only.
- Categories with transactions but no allocation may be added with `budgeted: 0`.

### Transfer links

A payment to a credit card is not spending. The money left net worth when the purchase posted
to the card; the payment only moves it between two accounts the user owns. Counting both the
purchase and the payment reports the same money twice.

A transaction may carry an optional `transfer_account_id` naming the counterpart account. The
rule applied to it is:

> A transaction is budget-neutral **if and only if** `transfer_account_id` resolves to a known
> account, neither that account nor the transaction's own account has
> `exclude_from_budgets: true`, and one of the two is a `credit_card`.

Consequences of that rule, all deliberate:

- The rule is symmetric, so whichever leg carries the link is excluded.
- Absent or `null` means "not a transfer".
- A link pointing at a deleted or unknown account degrades to counting the transaction rather
  than silently zeroing a real expense. `deleteAccount` also clears links pointing at it.
- Loan payments keep counting as expenses. A loan records no per-transaction spending, so the
  payment is the only record of the money leaving.
- Setting `exclude_from_budgets: true` on a card flips it back to counting payments as
  expenses and ignoring purchases. That is the correct setting when a card's spending is not
  fully imported, since excluding its payments would otherwise make the budget under-report.

See `isBudgetNeutralTransfer` in
`packages/apple/BasaltKit/Sources/BasaltBackend/Shared/BudgetRules.swift` for the rule and
`packages/apple/BasaltKit/Sources/BasaltBackend/Shared/TransferDetection.swift`
for how unlinked payments are detected from the payee, previously confirmed links, and
amount/date pairing.

`actual` and `remaining` are persisted snapshots. They are recalculated during budget creation,
some budget updates, and explicit budget recalculation. Transaction changes do not automatically
rewrite every affected budget file, so values on disk can become stale. Budget view and summary
reads calculate fresh values from transactions; raw budget reads can return the stored values.

See
`packages/apple/BasaltKit/Sources/BasaltBackend/Services/BudgetService.swift`
and `packages/apple/BasaltKit/Sources/BasaltBackend/Shared/BudgetRules.swift`.

## Payee index

Path:

```text
payees.yml
```

Example:

```yaml
---
payees:
  - Example Store
  - Other Payee
updated_at: '2026-07-10T03:10:16.417Z'
version: 1
---
```

Fields:

- `payees` — required string array. Unique, case-sensitive payee names sorted with
  `localeCompare`. Differently cased names are distinct.
- `updated_at` — required ISO 8601 string. Last index update.
- `version` — required number. Current index schema version is `1`.

This is a derived index, not the authoritative transaction record. It can be rebuilt by scanning
all transaction files. Incremental transaction updates modify it in the background, so a process
failure can temporarily leave it out of sync. Removing a payee from the index does not remove or
edit transactions.

See
`packages/apple/BasaltKit/Sources/BasaltBackend/Services/PayeeIndexService.swift`.

## Preferences

Path:

```text
preferences.yml
```

Example with no credential:

```yaml
---
subtract_liabilities_from_net_worth: false
date_format: YYYY-MM-DD
number_format: 1,000.00
default_view: dashboard
show_pending: true
categorization: suggest
currency_symbol: $
theme: dark
net_worth_range: 6m
expenses_comparison_type: month_vs_last_month
account_chart_range: 3m
gemini_api_key: ''
assisted_imports_used: 0
budget_show_remaining_income: false
budget_show_remaining_expenses: false
account_order: last_used
import_reminder:
  enabled: true
  frequency: daily
  time: '21:00'
  weekday: 0
updated_at: '2026-07-10T18:58:09.306Z'
---
```

Fields:

- `subtract_liabilities_from_net_worth` — required boolean. Controls whether credit-card and loan
  balances are treated as liabilities in net worth.
- `date_format` — required string, for example `YYYY-MM-DD` or `MM/DD/YYYY`.
- `number_format` — required string, for example `1,000.00` or `1.000,00`.
- `default_view` — required enum: `dashboard`, `accounts`, `transactions`, `categories`, or
  `budgets`.
- `show_pending` — required boolean. Whether pending transactions are shown by default.
- `categorization` — required enum: `auto`, `manual`, or `suggest`.
- `currency_symbol` — required string such as `$`, `€`, or `£`.
- `theme` — required enum: `light`, `dark`, or `auto`.
- `net_worth_range` — required enum: `1m`, `3m`, `6m`, `1y`, or `all`.
- `expenses_comparison_type` — required enum:
  - `week_vs_last_week`
  - `month_vs_last_month`
  - `month_vs_last_year`
  - `month_vs_average`
- `account_chart_range` — required enum: `1m`, `3m`, `6m`, `1y`, or `all`.
- `gemini_api_key` — required string. Google Gemini API key used for assisted PDF/image imports.
  This is a secret stored in plaintext; do not publish or commit a populated value.
- `assisted_imports_used` — required number. Count of included assisted image imports used.
- `budget_show_remaining_income` — required boolean. Shows remaining instead of actual in the
  income budget column.
- `budget_show_remaining_expenses` — required boolean. Shows remaining instead of actual in the
  expense budget column.
- `account_order` — required enum: `alphabetical`, `type`, or `last_used`.
- `import_reminder` — required object containing:
  - `enabled` — boolean
  - `frequency` — `daily` or `weekly`
  - `time` — local `HH:mm` string
  - `weekday` — integer `0` through `6`, where `0` is Sunday
- `updated_at` — required ISO 8601 string.

When the file is missing, the preferences service creates it from current runtime defaults. When
reading an existing file, it shallow-merges missing top-level fields from those defaults. The
new-vault initializer contains a smaller default template; the service fills the missing
top-level fields in the returned object on read, but does not deep-merge nested objects or
immediately rewrite the file.

Current runtime defaults:

```yaml
subtract_liabilities_from_net_worth: true
date_format: YYYY-MM-DD
number_format: 1,000.00
default_view: dashboard
show_pending: false
categorization: suggest
currency_symbol: $
theme: auto
net_worth_range: 1m
expenses_comparison_type: month_vs_last_month
account_chart_range: 1m
gemini_api_key: ''
assisted_imports_used: 0
budget_show_remaining_income: false
budget_show_remaining_expenses: false
account_order: type
import_reminder:
  enabled: false
  frequency: daily
  time: '21:00'
  weekday: 0
```

`updated_at` is generated at runtime. The new-vault initializer currently differs by using
`subtract_liabilities_from_net_worth: false`, `show_pending: true`, `categorization: auto`,
`net_worth_range: 3m`, and `account_chart_range: 3m`, and by omitting newer budget, ordering, and
reminder fields.

Current runtime defaults are defined in
`packages/apple/BasaltKit/Sources/BasaltBackend/Services/PreferencesService.swift`.

## Vault metadata

Path:

```text
.basalt/vault.json
```

This file is JSON, not YAML:

```json
{
  "id": "vault_1770000000000_abcdefgh",
  "name": "Example Vault",
  "createdAt": "2026-07-10T18:58:09.306Z",
  "version": "1.0.0"
}
```

Fields:

- `id` — required string. Generated vault identifier.
- `name` — required string. Vault display name supplied at creation.
- `createdAt` — required ISO 8601 string.
- `version` — required string. Metadata format/application version written at creation; currently
  `1.0.0`.

The app's vault watcher reacts only to `.yml` and `.yaml` files, so this metadata does not
participate in YAML change detection.

## Migration source maps

Path:

```text
.basalt/migrations/<provider>-<encoded-container-id>.json
```

Example:

```json
{
  "provider": "ynab",
  "container_id": "example-budget",
  "account_map": {
    "provider-account-id": "acc_1770000000000_abc1234"
  },
  "transaction_map": {
    "provider-transaction-id": "tx_1770000000000_def5678"
  },
  "transaction_skip_map": {
    "skipped-provider-transaction-id": "2026-07-10 18:58"
  },
  "imported_at": "2026-07-10 18:58",
  "schema_version": 1
}
```

Fields:

- `provider` — required enum: `ynab` or `monarch` for migrations, `plaid` for bank sync (see
  *Bank sync* below).
- `container_id` — required string. Source provider's budget/container ID.
- `account_map` — required object mapping provider account IDs to Basalt account IDs.
- `transaction_map` — required object mapping provider transaction IDs to Basalt transaction IDs.
- `transaction_skip_map` — optional object mapping provider transaction IDs to the local
  `YYYY-MM-DD HH:mm` time at which the user chose to skip them.
- `imported_at` — required local `YYYY-MM-DD HH:mm` string. Last migration/import time.
- `schema_version` — required number. Current value is `1`.

The container ID in the filename is encoded with `encodeURIComponent`, with each `%` replaced by
`_`.

See
`packages/apple/BasaltKit/Sources/BasaltBackend/Services/MigrationService.swift`.

## Bank sync

Bank sync connects a bank through Plaid. Each linked bank (a Plaid *item*) has a source map and,
while transactions wait for review, a pending file.

### Bank source map

Path:

```text
.basalt/migrations/plaid-<encoded-item-id>.json
```

Example:

```json
{
  "provider": "plaid",
  "container_id": "example-item-id",
  "institution_name": "Example Bank",
  "account_map": {
    "plaid-account-id": "acc_1770000000000_abc1234"
  },
  "transaction_map": {
    "plaid-transaction-id": "tx_1770000000000_def5678"
  },
  "transaction_skip_map": {
    "skipped-plaid-transaction-id": true
  },
  "imported_at": "2026-11-14 10:12",
  "schema_version": 1,
  "sync_from": "2026-11-01",
  "cursor": "opaque-plaid-cursor"
}
```

It uses the migration source map format above, with these differences:

- `container_id` — the Plaid item ID.
- `institution_name` — string or `null`. Bank display name.
- `account_map` — Plaid account IDs to Basalt account IDs. Empty until the user maps the
  accounts; Plaid accounts that are not in the map do not sync.
- `transaction_skip_map` — values are `true`, not timestamps.
- `imported_at` — set when the bank is linked.
- `sync_from` — optional `YYYY-MM-DD` string. Transactions dated before it are ignored.
- `cursor` — optional string. Plaid's sync cursor; the next sync fetches changes after it.
- `last_error_code` — optional string. Plaid error of the last failed sync, for example
  `ITEM_LOGIN_REQUIRED`. A successful sync removes it.

### Pending transactions

Path:

```text
.basalt/bank-sync/plaid-<encoded-item-id>-pending.json
```

Transactions fetched from the bank wait here until the user reviews them. The file is in the
vault, so every device that opens the vault sees them.

Example:

```json
{
  "schema_version": 3,
  "rows": [
    {
      "plaid_id": "plaid-transaction-id",
      "plaid_account_id": "plaid-account-id",
      "account_id": "acc_1770000000000_abc1234",
      "date": "2026-11-20 00:00",
      "amount": 12.5,
      "type": "debit",
      "payee": "Example Store",
      "description": "EXAMPLE STORE #123",
      "status": "pending"
    }
  ]
}
```

Row fields:

- `plaid_id` — Plaid transaction ID.
- `plaid_account_id` — Plaid account ID.
- `account_id` — Basalt account ID from `account_map`.
- `date` — `YYYY-MM-DD 00:00`, from Plaid's authorized date, else its posted date.
- `amount` — positive number rounded to two decimal places.
- `type` — `debit` when money leaves the account (credit cards included), else `credit`.
- `payee` — the merchant name, else the bank's text. Empty when the bank sends only a generic
  label, so the review asks the user for a payee.
- `description` — the bank's own text when it differs from `payee`, else `''`.
- `status` — `pending` or `cleared`.

Imported rows move to `transaction_map` and skipped rows to `transaction_skip_map`; both leave
the pending file. The file is deleted when no row is left. Removing a bank deletes its source map and
pending file; its imported transactions stay.

Plaid access tokens are never written to the vault. They are kept in the synced iCloud Keychain,
so the user's other devices can sync the same bank.

See
`packages/apple/BasaltKit/Sources/BasaltBackend/Services/BankSyncService.swift`.

## Referential integrity

The format uses string references rather than embedding related entities:

```text
transaction.account_id  -> account.id
transaction.category_id -> category.id
budget.*.category_id    -> category.id
category.parent_id      -> category.id or null
migration account_map   -> account.id
migration transaction_map -> transaction.id
bank-sync row.account_id  -> account.id
```

Basalt does not have a database constraint layer. Manual edits can therefore create dangling or
duplicate references.

Guidelines:

- Treat `id` as immutable.
- Keep account, category, and transaction IDs globally unique within their entity type.
- Do not delete an account file without handling its transactions. The application account-delete
  operation also deletes associated transactions.
- Do not delete or change a category ID while transactions or budgets reference it.
- Keep each transaction in the month matching its `date`; range queries choose files by filename.
- Keep each budget's `period` equal to its filename.
- Do not place multiple entities in account, category, budget, preference, or payee files;
  single-document readers use only the first document.

## Runtime-only and unused artifacts

- `imports/` — created for future/optional import storage, but currently remains empty. Import
  jobs are held in memory and expire; raw documents and redacted pages are processed in memory
  and are never written to a vault.
- Migration sessions, YNAB PATs, Monarch session material, Plaid access tokens, and
  assisted-import installation IDs are runtime secrets. They are never written to YAML, source
  maps, vault metadata, or logs. The apps keep them in the Keychain (Plaid tokens in the synced
  iCloud Keychain) or, for Monarch, in memory only.
- File sync — no sync index or conflict sidecar is stored in the vault. The sync service the user
  picked (iCloud Drive, Syncthing, git, ...) moves the files.
- `.<filename>.icloud` — iCloud Drive placeholder for a file that is not yet downloaded to the
  device. Basalt downloads the file before it reads it.
- `.DS_Store` — may be created by macOS. Basalt does not own it, and file watching ignores it.

## Maintaining this document

Update this file whenever a change modifies:

- an entity in `packages/apple/BasaltKit/Sources/BasaltModels/Types/`
- default categories, preferences, IDs, or vault initialization in
  `packages/apple/BasaltKit/Sources/BasaltBackend/` (`Vault/`, `Shared/SharedUtils.swift`,
  `Services/PreferencesService.swift`)
- a path, serializer, or write rule in `packages/apple/BasaltKit/Sources/BasaltBackend/Services/`
  or `Codec/`
- iOS or macOS vault creation behavior (`packages/apple/AppCore/Stores/VaultStore.swift`)

Verify examples against a real vault, but replace personal names, financial values where
necessary, provider identifiers, and all credentials before committing documentation.
