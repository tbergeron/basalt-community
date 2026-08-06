# Vault File Format

Basalt stores your budget as a folder of plain YAML files — a **vault**. You own that folder; it works with git, Syncthing, Dropbox, or any file sync you like, and you can read or edit it by hand.

This document is the reference for the vault format: the current layout, files created lazily at runtime, and legacy data found in older vaults. It's generated from Basalt's internal types and services (not included in this repo, which covers docs and community issues only), so file paths cited below (e.g. `packages/shared/src/types`) are internal pointers for context, not links you can follow here.

## Overview

A Basalt vault is a plain directory containing YAML entity files and a small amount of JSON
metadata. Files can be read and edited without Basalt, but manual edits must preserve IDs,
references, data types, and YAML document separators.

Node runtime resolves its default active vault in this order:

1. `BASALT_VAULT_PATH`
2. `BASALT_DATA_DIR`
3. `<repository>/data/my-budget`

Electron sets the active Node storage root from its vault host bridge. Mobile injects its active
root through the React Native storage port after opening a security-scoped bookmark; it does not
read Node environment variables. Operations that require vault data fail with `vault_unavailable`
until a platform has configured an active root.

See `packages/backend/src/runtime`.

## Directory layout

```text
vault/
├── .basalt/
│   ├── vault.json
│   └── migrations/
│       └── <provider>-<container-id>.json
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
├── preferences.yml
└── meta.yml
```

The files and directories do not all exist in every vault:

- New-vault initialization creates `accounts/`, `budgets/`, `categories/`, `imports/`,
  `transactions/`, `.basalt/vault.json`, and `preferences.yml`.
- New-vault initialization seeds the default category files.
- Account, budget, and transaction files are created when their first entities are created.
- `payees.yml` is created lazily when the payee index is initialized or updated.
- `.basalt/migrations/` is created lazily when a migration source map is written.
- `imports/` is currently an empty scaffold. Import jobs and uploaded documents are not stored
  there.
- `meta.yml` is legacy/demo data. The current runtime does not read or write it.
- Older vaults may have no `.basalt/` directory and can still contain usable financial data.

See `packages/shared/src/utils/vaultInit.ts`.

## YAML conventions

### Single-document files

Accounts, budgets, categories, `payees.yml`, `preferences.yml`, and legacy `meta.yml` contain one
YAML document wrapped in `---` markers:

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
`packages/backend/src/services/yamlService.ts`.

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
- `preferences.updated_at`, migration timestamps, and vault metadata use ISO 8601 strings such as
  `2026-07-10T18:58:09.306Z`

The parser accepts valid YAML scalars, but application code generally expects these exact string
formats. Quoting a timestamp is safe and prevents YAML libraries from interpreting it as a native
date value.

## IDs and filenames

### Entity IDs

Current account, category, and transaction IDs use:

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
`packages/shared/src/utils/validation.ts`.

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

Current example:

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
`packages/backend/src/services/accountService.ts`.

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
`packages/shared/src/defaultCategories.ts`. Current
seeding generates a new random `cat_` ID for every category.

The current seed set is:

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
`packages/backend/src/services/transactionService.ts`.

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
- Absent or `null` means "not a transfer", which is the behaviour of every vault written
  before this field existed. Upgrading changes nothing on its own.
- A link pointing at a deleted or unknown account degrades to counting the transaction rather
  than silently zeroing a real expense. `deleteAccount` also clears links pointing at it.
- Loan payments keep counting as expenses. A loan records no per-transaction spending, so the
  payment is the only record of the money leaving.
- Setting `exclude_from_budgets: true` on a card flips it back to counting payments as
  expenses and ignoring purchases. That is the correct setting when a card's spending is not
  fully imported, since excluding its payments would otherwise make the budget under-report.

See `packages/shared/src/utils/transfer.ts` for the
rule and
`packages/shared/src/services/transferDetection.ts`
for how unlinked payments are detected from the payee, previously confirmed links, and
amount/date pairing.

`actual` and `remaining` are persisted snapshots. They are recalculated during budget creation,
some budget updates, and explicit budget recalculation. Transaction changes do not automatically
rewrite every affected budget file, so values on disk can become stale. Budget view and summary
reads calculate fresh values from transactions; raw budget reads can return the stored values.

See
`packages/backend/src/services/budgetService.ts`
and `packages/shared/src/utils/category.ts`.

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
`packages/backend/src/services/payeeIndexService.ts`.

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
new-vault initializer contains an older, smaller default template; the service fills newer
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
`packages/shared/src/types/preferences.ts`.

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

The standard backend YAML file watcher ignores dotfiles and JSON, so this metadata does not
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
    "skipped-provider-transaction-id": "2026-07-10T18:58:09.306Z"
  },
  "imported_at": "2026-07-10T18:58:09.306Z",
  "schema_version": 1
}
```

Fields:

- `provider` — required enum. Currently `ynab` or `monarch`.
- `container_id` — required string. Source provider's budget/container ID.
- `account_map` — required object mapping provider account IDs to Basalt account IDs.
- `transaction_map` — required object mapping provider transaction IDs to Basalt transaction IDs.
- `transaction_skip_map` — optional object mapping provider transaction IDs to the ISO timestamp
  at which the user chose to skip them.
- `imported_at` — required ISO 8601 string. Last migration/import timestamp.
- `schema_version` — required number. Current value is `1`.

The container ID in the filename is encoded with `encodeURIComponent`, with each `%` replaced by
`_`. The migration service also reads the legacy `.basalt/migrations/<provider>.json` path as a
fallback. Account merges rewrite `account_map` values from the source account ID to the target
account ID.

See
`packages/backend/src/services/migrationService.ts`
and
`packages/backend/src/services/accountMergeService.ts`.

## Legacy `meta.yml`

Path:

```text
meta.yml
```

Example:

```yaml
---
app_version: 1.0.0
created_at: 2024-01-01 00:00
last_opened: 2026-01-29 12:00
---
```

Fields:

- `app_version` — string. Application version associated with the vault.
- `created_at` — date-time string. Legacy vault creation timestamp.
- `last_opened` — date-time string. Legacy last-opened timestamp.

The `Meta` shared type still exists, and the demo-vault generator can create this file, but the
current backend has no meta service or route and does not read or update `meta.yml`. Current vault
identity metadata belongs in `.basalt/vault.json`.

## Referential integrity

The format uses string references rather than embedding related entities:

```text
transaction.account_id  -> account.id
transaction.category_id -> category.id
budget.*.category_id    -> category.id
category.parent_id      -> category.id or null
migration account_map   -> account.id
migration transaction_map -> transaction.id
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
- Migration sessions, YNAB PATs, Monarch session material, and assisted-import installation IDs
  are runtime secrets. They are never written to YAML, migration maps, vault metadata, or logs.
  Mobile stores permitted transient secrets through its Keychain port; Electron keeps migration
  session secrets in process memory.
- Sync state and merge conflicts — held in memory; no sync index or conflict sidecar is stored in
  the vault.
- `.temp` — some mobile code configures this path, but current import code does not write to it.
- `.DS_Store` — may be created by macOS. Basalt does not own it, and file watching ignores it.
- `accounts-backup-<timestamp>/` — may be created by the maintenance script
  `scripts/fix-account-balances.ts`; it is not part of the runtime vault format.

## Legacy and compatibility notes

### Legacy account `balance`

Old account files used one `balance` field instead of `opening_balance` and `current_balance`.
When an account has `balance` but no `opening_balance`, account loading migrates it in memory:

- `opening_balance` and `current_balance` both receive the old value.
- `opening_date` is set approximately ten years before migration so historical transactions are
  included.
- A later account write serializes the migrated current fields.

`balance` is not part of the current `Account` type. Some transitional vaults contain both
`balance` and the current fields. Because current reads and updates spread parsed objects, that
unknown key can be preserved on later writes; consumers must not rely on it. The authoritative
fields are `opening_balance` and `current_balance`.

### Fixed category IDs

Older and demo vaults may use readable IDs such as `cat_food`, `cat_housing`, or `cat_salary`.
These remain valid string IDs when references are consistent. Current new-vault seeding generates
timestamp/random `cat_` IDs instead, so code must never assume a category's purpose from its ID.

### Missing `.basalt`

Older vaults may have valid YAML data but no `.basalt/vault.json`. This indicates that the vault
predates current metadata initialization; it does not invalidate the YAML entities.

### Initializer defaults versus runtime defaults

The new-vault YAML template and the current `defaultPreferences` object are not identical. Runtime
reads merge current top-level defaults into the stored file. Documentation and migrations should
use the shared `Preferences` type and `defaultPreferences` object as the current schema.

## Maintaining this document

Update this file whenever a change modifies:

- an entity in `packages/shared/src/types/`
- default categories, preferences, IDs, or vault initialization in `packages/shared/src/`
- a path, serializer, or write rule in `packages/backend/src/services/`
- Electron or mobile vault creation behavior

Verify examples against a real vault, but replace personal names, financial values where
necessary, provider identifiers, and all credentials before committing documentation.
