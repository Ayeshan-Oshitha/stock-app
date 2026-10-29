# Database Design

## Decision: one PostgreSQL server, one database per service

| Option | Verdict |
|---|---|
| One database, all tables mixed | ❌ Couples services together |
| One database, one schema per service | ⚠️ Works, but cross-schema queries are too easy |
| **One server, one database per service** | ✅ **Chosen** |
| One server per service | ❌ Too costly for now |

```
PostgreSQL server (one container on the VM)
├── keycloak_db      # managed by Keycloak (users, credentials, sessions); never touched by our code
├── portfolio_db     # portfolios, holdings, append-only transactions, outbox/inbox
├── marketdata_db    # companies, sectors, daily prices (TimescaleDB), indices, corporate actions, announcements, dividends, outbox
├── analytics_db     # copy of prices, indicators, forecasts, actuals, accuracy, model versions (TimescaleDB)
└── knowledge_db     # documents, chunks + embeddings (pgvector), extracted financials
```

**Why**

- Real isolation: a service cannot accidentally query another service's data.
- Each service has its own migrations.
- Still cheap: one container, one backup job.
- Moving a database to its own server later is a connection-string change.
- Extensions only where needed: TimescaleDB (marketdata, analytics), pgvector (knowledge). Extensions and hypertables are created with hand-written SQL inside migrations (the only allowed exception besides seed data).

**Access:** each service connects with its own database user that can access only its own database.

## Portfolio storage: append-only transactions

The portfolio is **not** event-sourced. It uses a simpler model with the same benefits:

- Buys, sells, sell-lot allocations, undo and restore actions are stored as rows that are **never deleted**.
- The `Holding` aggregate is loaded from these rows, and the **domain calculates** quantity, average cost, lot balances and P&L in memory.
- Undo is a status change on the sell plus an audit record; restore is the reverse.
- Each aggregate root has an `xmin` concurrency token, and **every change updates the root row** (e.g. `version` / `updated_at` on `holdings`). Adding only child rows would not change `xmin`, so two concurrent sells could both pass the "enough shares" check.

This gives full history and safe undo at any point without an event store, projections or snapshots.

## Tenancy: multi-tenant, shared tables

- One app and one set of databases serve all users.
- Every user-owned row has a `user_id`: the Keycloak user ID (`sub` claim) from the JWT, which each service validates itself.

| Data type | Examples | `user_id`? |
|---|---|---|
| Shared market data | Prices, companies, announced dividends, forecasts, reports, knowledge base | No (stored once, used by all) |
| Private user data | Portfolios, holdings, transactions, alerts, watchlists, notes, questions asked | Yes, always filtered |

**Now:** EF Core **global query filters** on `UserId` apply the filter automatically, and tests check that one user can never read another user's data.

**Later:** PostgreSQL **Row-Level Security** as a second safety layer, if the app becomes public. It needs the user ID set on every connection, which is extra work with connection pooling.

Shared or family portfolios (later) will use a sharing table, not a new tenancy model.

## Getting data from other services

- Portfolio needs current prices → consumes `PricesUpdated` and keeps a local latest-price table.
- Analytics needs price history → calls the market-data **bulk price API** after `PricesUpdated` and keeps its own copy.

## Messaging tables

Each service that publishes or consumes integration events has:

| Table | Purpose |
|---|---|
| `outbox_messages` | Events saved in the same transaction as the business change; a background service publishes them to RabbitMQ and marks them sent |
| `inbox_messages` | IDs of processed messages, so duplicate deliveries are ignored |

## Backups

- One nightly backup of the PostgreSQL server (including `keycloak_db`).
- Extracted data and embeddings in `knowledge_db` are important to back up: they cost LLM calls to recreate. PDFs can be re-downloaded.

## Detailed table design

To be designed per service, starting with `portfolio_db`.
