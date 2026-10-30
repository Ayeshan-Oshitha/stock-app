# portfolio_db Design

Status: **draft, step 2 (columns, types and constraints)**.

Scope (version 1): portfolios, buys/lots, sells with lot choice, undo/restore of sells, holdings, history, fees, tags, manual prices, and the read-only company copy.

*Italic rows and tables are added later* (corporate actions, dividends, events and other later features). They are shown now so the version 1 design leaves room for them.

## Diagram

See [diagrams/portfolio-db-erd.md](diagrams/portfolio-db-erd.md). Later tables are shaded grey; later columns are marked `later`.

## Overview

| Table | Holds | Notes |
|---|---|---|
| `companies` | Read-only copy of CSE securities (symbol, name, sector, status) | Same `id` as Market Data. Filled from the CSV now, from Market Data events later. Portfolio never edits it |
| `latest_prices` | Current price per company (global, one row per stock, no `user_id`) | Shared by all users. Entered by hand in version 1; later filled from Market Data's `PricesUpdated` after each trading day |
| `portfolios` | A user's portfolio: name, default fee rate | Small aggregate. Archived, not deleted, once it has transactions |
| `holdings` | One portfolio + one company | The `Holding` aggregate root. Updated on every change so `xmin` changes (decision #24) |
| `transactions` | Every buy and sell | Append-only. A buy **is** a lot. Stores actual price, quantity, fee amount and fee rate |
| `sell_allocations` | Which lot(s) a sell took shares from, and how many | Several rows when a sell is split across lots. Realized P&L comes from these |
| `sell_status_history` | Each undo and restore of a sell, with time and user | Audit trail; the sell's current status is on `transactions` |
| `tags` | Built-in (Investment, IPO, Trade) and user-defined tags | Labels only |
| `transaction_tags` | Links buys/sells to tags | Zero or more tags per transaction |
| *`lot_adjustments`* | *Bonus/split changes to a lot's quantity* | *Corporate actions* |
| *`dividend_entitlements`* | *Dividends a holding is entitled to, and whether received* | *Dividends* |
| *`outbox_messages`* | *Events waiting to be published* | *Events (decision #23)* |
| *`inbox_messages`* | *IDs of events already processed* | *Events* |

## Common rules

- Primary keys are `uuid` (GUID v7, created in the application, so they sort by time).
- Timestamps are `timestamptz` in UTC; trading dates are `date` in Colombo time (decision #31).
- Money: prices and amounts `numeric(18,2)`; fee rates `numeric(7,4)` as a percentage (1.1200 = 1.12%); quantities `bigint`, whole shares (decision #20).
- Enum-like values are stored as text (`varchar`) with a check constraint, not as numbers.
- `user_id` is the Keycloak `sub` (`uuid`) and is filtered by EF Core global query filters (decision #7).
- Concurrency uses PostgreSQL's hidden `xmin` column on aggregate roots (`portfolios`, `holdings`); it is not a real column.

## Columns

### companies

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK. Same id as Market Data, never generated here |
| `symbol` | varchar(20) | no | Unique. Can change; links always use `id` |
| `name` | varchar(200) | no | |
| `sector` | varchar(100) | yes | |
| `status` | varchar(20) | no | `Active` / `Suspended` / `Delisted`. New buys only when `Active` |
| `updated_at` | timestamptz | no | |
| *`share_type`* | *varchar(20)* | *yes* | *`Voting` / `NonVoting`, if the UI needs it* |
| *`source_updated_at`* | *timestamptz* | *yes* | *Time of the Market Data change; older events are ignored* |

### latest_prices

| Column | Type | Null | Notes |
|---|---|---|---|
| `company_id` | uuid | no | PK, FK → `companies.id` |
| `price` | numeric(18,2) | no | Check `> 0` |
| `price_date` | date | no | Trading date the price belongs to |
| `source` | varchar(20) | no | `Manual` / `MarketData` |
| `updated_at` | timestamptz | no | |
| `updated_by` | uuid | yes | User who entered it; empty when from Market Data |
| *`previous_close`* | *numeric(18,2)* | *yes* | *For day change on the holdings page* |

### portfolios

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `user_id` | uuid | no | Owner. Indexed |
| `name` | varchar(100) | no | Unique per user (`user_id`, `name`) |
| `default_fee_rate` | numeric(7,4) | no | Default 1.1200. Check `>= 0` and `< 100` |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |
| `archived_at` | timestamptz | yes | Set when archived; archived portfolios are read-only |
| *`description`* | *varchar(500)* | *yes* | |
| *`display_order`* | *int* | *yes* | *User's order on the portfolio list* |

### holdings

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `portfolio_id` | uuid | no | FK → `portfolios.id` |
| `company_id` | uuid | no | FK → `companies.id` |
| `user_id` | uuid | no | Copied from the portfolio for filtering |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | Changed on **every** change to the holding, so `xmin` changes |

Unique: (`portfolio_id`, `company_id`). Indexes: `user_id`, `company_id`.

### transactions

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK. For a buy, this is the lot id |
| `holding_id` | uuid | no | FK → `holdings.id` |
| `user_id` | uuid | no | For filtering |
| `type` | varchar(10) | no | `Buy` / `Sell` |
| `status` | varchar(10) | no | Buy: `Active` / `Voided`. Sell: `Active` / `Undone` |
| `trade_date` | date | no | |
| `quantity` | bigint | no | Check `> 0` |
| `price` | numeric(18,2) | no | Check `> 0` |
| `fee_rate` | numeric(7,4) | no | Rate used (pre-filled from the portfolio) |
| `fee_amount` | numeric(18,2) | no | Actual fee, editable. Check `>= 0` |
| `note` | varchar(500) | yes | |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |
| `voided_at` | timestamptz | yes | Buys only, when deleted |
| *`source`* | *varchar(20)* | *no* | *`Manual` / `Import` / `Rights`; default `Manual`* |
| *`corporate_action_id`* | *uuid* | *yes* | *Rights take-up buys: the confirmed action in Market Data* |
| *`settlement_date`* | *date* | *yes* | *T+2 settlement* |
| *`contract_note_no`* | *varchar(50)* | *yes* | *Broker contract note reference* |

Index: (`holding_id`, `trade_date`). History filters use `user_id`, `type`, `trade_date`.

### sell_allocations

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `sell_id` | uuid | no | FK → `transactions.id` (type `Sell`) |
| `buy_id` | uuid | no | FK → `transactions.id` (type `Buy`, the lot) |
| `quantity` | bigint | no | Check `> 0` |

Unique: (`sell_id`, `buy_id`). Index: `buy_id` (to find sells from a lot). The sum of a sell's allocations must equal its quantity; the domain checks this.

### sell_status_history

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `sell_id` | uuid | no | FK → `transactions.id` |
| `action` | varchar(10) | no | `Undo` / `Restore` |
| `occurred_at` | timestamptz | no | |
| `user_id` | uuid | no | Who did it |
| *`reason`* | *varchar(500)* | *yes* | *Optional note from the user* |

### tags

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | PK |
| `user_id` | uuid | yes | Empty = built-in tag (Investment, IPO, Trade) |
| `name` | varchar(50) | no | Unique per user (`user_id`, `name`); built-in names reserved |
| `created_at` | timestamptz | no | |
| *`color`* | *varchar(7)* | *yes* | *Hex colour for the UI* |

### transaction_tags

| Column | Type | Null | Notes |
|---|---|---|---|
| `transaction_id` | uuid | no | PK part, FK → `transactions.id` |
| `tag_id` | uuid | no | PK part, FK → `tags.id` |

### *lot_adjustments (later)*

| Column | Type | Null | Notes |
|---|---|---|---|
| *`id`* | *uuid* | *no* | *PK* |
| *`buy_id`* | *uuid* | *no* | *FK → `transactions.id` (the lot)* |
| *`corporate_action_id`* | *uuid* | *no* | *Confirmed action in Market Data* |
| *`type`* | *varchar(20)* | *no* | *`Bonus` / `Split` / `Consolidation`* |
| *`quantity_change`* | *bigint* | *no* | *Shares added (or removed for consolidation)* |
| *`fraction_dropped`* | *numeric(18,6)* | *no* | *Fractional shares rounded down and recorded* |
| *`effective_date`* | *date* | *no* | |
| *`confirmed_at`* | *timestamptz* | *no* | |
| *`confirmed_by`* | *uuid* | *no* | |

### *dividend_entitlements (later)*

| Column | Type | Null | Notes |
|---|---|---|---|
| *`id`* | *uuid* | *no* | *PK* |
| *`holding_id`* | *uuid* | *no* | *FK → `holdings.id`* |
| *`user_id`* | *uuid* | *no* | |
| *`dividend_id`* | *uuid* | *no* | *Dividend announcement in Market Data* |
| *`xd_date`* | *date* | *no* | |
| *`payment_date`* | *date* | *yes* | |
| *`quantity_entitled`* | *bigint* | *no* | *Shares held before the XD date* |
| *`amount_per_share`* | *numeric(18,4)* | *no* | |
| *`gross_amount`* | *numeric(18,2)* | *no* | |
| *`tax_amount`* | *numeric(18,2)* | *no* | *Withholding tax* |
| *`net_amount`* | *numeric(18,2)* | *no* | |
| *`status`* | *varchar(20)* | *no* | *`Expected` / `Received`* |
| *`received_on`* | *date* | *yes* | |

### *outbox_messages (later)*

| Column | Type | Null | Notes |
|---|---|---|---|
| *`id`* | *uuid* | *no* | *PK, also the message id* |
| *`type`* | *varchar(200)* | *no* | *Event name and version* |
| *`payload`* | *jsonb* | *no* | |
| *`occurred_at`* | *timestamptz* | *no* | |
| *`processed_at`* | *timestamptz* | *yes* | *Empty = not yet published. Partial index on unprocessed rows* |
| *`attempts`* | *int* | *no* | |
| *`last_error`* | *text* | *yes* | |

### *inbox_messages (later)*

| Column | Type | Null | Notes |
|---|---|---|---|
| *`message_id`* | *uuid* | *no* | *PK; duplicates are ignored* |
| *`type`* | *varchar(200)* | *no* | |
| *`processed_at`* | *timestamptz* | *no* | |

## How the rules map to the tables

- **Lot balance** = buy quantity (+ *lot adjustments*) − allocations from active (not undone) sells. Calculated in the domain, not stored.
- **Holding quantity, average cost, P&L** are calculated in memory from `transactions` and `sell_allocations` (decision #15).
- **Undo a sell:** its status becomes Undone and a row is added to `sell_status_history`. Its allocations stay but are ignored; restore makes them count again.
- **Edit or delete a buy:** blocked if any active sell has an allocation from it. A deleted buy is marked Voided, never removed.
- **Fees:** each transaction stores its own fee amount and rate; the portfolio rate only pre-fills (decision #29).

Because lot balance is always calculated, adding `lot_adjustments` later needs no change to existing tables.

## Open questions

1. Deleting a portfolio: allow it only when it has no transactions, otherwise archive?
