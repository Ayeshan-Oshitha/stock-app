# Backend Structure

## .NET services: vertical slices + rich domain + CQRS

Each .NET service has **two projects**: a pure **Domain** project and an **Api** project organised **by feature** (vertical slices). Example for portfolio:

```
services/portfolio/
├── src/
│   ├── Portfolio.Domain/              # Pure C#: no package or project references (except shared building blocks)
│   │   ├── Portfolios/                # Small aggregate: name, owner, fee settings
│   │   │   ├── Portfolio.cs
│   │   │   ├── PortfolioId.cs         # Strongly-typed ID (readonly record struct, Guid v7)
│   │   │   └── PortfolioErrors.cs
│   │   ├── Holdings/                  # Main aggregate: one portfolio + one stock
│   │   │   ├── Holding.cs             # Aggregate root: lots, sells, allocations, undo/restore, corporate actions
│   │   │   ├── Lot.cs                 # Child entity (changed only through the root)
│   │   │   ├── Sell.cs                # Child entity
│   │   │   ├── SellAllocation.cs      # Which lot(s) a sell came from
│   │   │   ├── HoldingId.cs
│   │   │   ├── SellStatus.cs          # Enum: Active, Undone
│   │   │   ├── HoldingErrors.cs       # Static Error definitions
│   │   │   └── Events/                # Domain events (sealed records, past tense)
│   │   └── Common/                    # Value objects specific to this service (Money, Quantity, FeeRate, …)
│   └── Portfolio.Api/
│       ├── Features/                  # One folder per use case (vertical slice)
│       │   ├── AddBuy/
│       │   │   ├── AddBuyEndpoint.cs      # Minimal API endpoint (thin)
│       │   │   ├── AddBuyRequest.cs       # HTTP request/response records
│       │   │   ├── AddBuyCommand.cs       # sealed record : ICommand<LotId>
│       │   │   ├── AddBuyHandler.cs       # internal sealed : ICommandHandler<…>
│       │   │   └── AddBuyValidator.cs     # FluentValidation, input shape only
│       │   ├── AddSell/
│       │   ├── UndoSell/
│       │   ├── RestoreSell/
│       │   └── GetHoldings/               # Query + handler + read model
│       ├── Common/                    # Result → HTTP mapping, decorators, paging
│       ├── Infrastructure/            # DbContext, EF configurations, repositories, messaging, clock
│       └── Program.cs
└── tests/
    ├── Portfolio.Domain.Tests/        # Every domain behaviour
    └── Portfolio.Api.Tests/           # Handlers, validators, integration tests (Testcontainers)
```

**Why this structure**

- **Everything for one feature is in one folder**, so it is easy to find, change and review.
- **The rich domain stays separate and pure**, so the core rules (lots, undo sells, holdings calculation, fees) are fully unit-testable.
- **Two projects per service instead of four**, a better size for microservices where each service is already small.

**Dependency rule**

```
Portfolio.Api ──▶ Portfolio.Domain ──▶ shared building blocks
Portfolio.Domain has no framework, EF Core or ASP.NET dependencies
```

### Gateway (exception: one project)

The gateway has no domain, so it is a **single project** (`src/gateway/Gateway/`): YARP routing configuration, JWT validation, CORS, rate limiting and health checks. It contains no business logic.

### Shared building blocks (`src/shared/dotnet`)

Written once, used by every .NET service. Infrastructure and patterns only, **never business logic**:

- `Result`, `Result<T>`, `Error`, `ErrorType`
- `Entity<TId>`, `AggregateRoot<TId>`, `IDomainEvent`
- CQRS interfaces: `ICommand`, `ICommand<TResponse>`, `IQuery<TResponse>`, `ICommandHandler<,>`, `IQueryHandler<,>`
- Handler decorators (validation, logging)
- `IClock`
- `Result` → HTTP ProblemDetails mapping
- Messaging helpers (RabbitMQ publish/consume, outbox)

## .NET rules

### Rich domain (non-negotiable)

1. Aggregates own their invariants. **No public setters.** State changes only through intention-revealing methods (`holding.UndoSell(sellId, clock.UtcNow)`, not `sell.Status = …`).
2. Create through static factories that return `Result<T>`. Constructors are private, plus a private parameterless one for EF.
3. Expected business failures return `Result` / `Error` (e.g. `HoldingErrors.InsufficientLotQuantity`). Throw only for bugs.
4. **Strongly-typed IDs** (`PortfolioId`, `HoldingId`, `LotId`), Guid v7 (`Guid.CreateVersion7()`). Reference other aggregates by ID only.
5. Statuses are enums owned by the aggregate. No magic ints.
6. `DateOnly` / `TimeOnly` for calendar values. **Never `DateTime.Now`**: inject `IClock` in handlers, pass time into domain methods.
7. Child entities change only through their aggregate root. Collections are exposed as `IReadOnlyCollection<T>`.
8. Only raise events that have a consumer.
9. Every domain behaviour change comes with unit tests.

### CQRS

- **Own interfaces, no MediatR** (commercial licence). **No AutoMapper** (commercial licence); map by hand.
- **Commands** load aggregates through repositories, call the domain, save, and return `Result` (an ID, never an entity).
- **Handler pattern:** load → gather facts the domain needs → call the domain method (no business `if`s in handlers) → `SaveChangesAsync` → return `Result`.
- **Queries** skip repositories and aggregates: `AsNoTracking` + `Select` projections straight into read models.
- **Validators** check input shape only (required, lengths, formats). Business rules stay in Domain.
- Cross-cutting concerns (validation, logging) are **decorators** around handlers.
- Pass `CancellationToken` through every async call.

### Result → HTTP

| `ErrorType` | HTTP |
|---|---|
| Validation | 400 |
| NotFound | 404 |
| Conflict | 409 |
| Failure (business rule) | 422 |

Returned as RFC 9457 ProblemDetails with the error `code` in extensions. Unhandled exceptions → 500 via a global exception handler.

### API conventions

- **Minimal APIs**, built-in OpenAPI (`Microsoft.AspNetCore.OpenApi`), no Swashbuckle.
- Endpoints are thin: map request → command/query, call the handler, map `Result` → HTTP.
- Routes: `/api/<resource>` with kebab-case plural nouns. Non-CRUD actions are `POST` sub-resources, e.g. `POST /api/portfolios/{portfolioId}/holdings/{holdingId}/sells/{sellId}/undo`.
- Request/response records stay separate from commands and domain types. **Never return entities.**
- Lists support paging (`?page=1&pageSize=20`, max 100).
- `.WithName()`, `.WithTags()`, `.Produces<T>()`, `.ProducesProblem()` on every endpoint.

### Persistence conventions

- EF Core + PostgreSQL (Npgsql), **snake_case** naming (`EFCore.NamingConventions`).
- Fluent API only; no data annotations on domain types.
- Enums stored as strings. Value objects as complex/owned types. Strongly-typed IDs via value converters; keys `ValueGeneratedNever()`.
- `xmin` concurrency token on every aggregate root. **Every change to an aggregate must update the root row** (e.g. a `version` / `updated_at` column), even when only child rows are added. Otherwise `xmin` does not change and concurrent changes (e.g. two sells from the same lot) are not detected. Concurrency and unique violations → 409.
- `DateTimeOffset` values are UTC.

### Time zone and trading dates

The CSE runs on **Sri Lanka time (`Asia/Colombo`, UTC+5:30)**.

| Kind of value | Type | Stored as |
|---|---|---|
| Timestamps (when something happened, e.g. created / undone at) | `DateTimeOffset` | **UTC** |
| Trading dates (trade date, price date, XD date, payment date) | `DateOnly` | **Colombo date** |
| "Today", market open/close, scheduled jobs (e.g. 14:30 daily run) | via `IClock` | Calculated in **`Asia/Colombo`** |

- `IClock` exposes both `UtcNow` and `TodayInColombo`; code never converts time zones by hand.
- The frontend displays all dates and times in Colombo time.
- Migrations are generated by EF Core. **Hand-written SQL in a migration is allowed only for** PostgreSQL extensions (`CREATE EXTENSION timescaledb` / `vector`), TimescaleDB hypertables, and seed data. Everything else is generated.
- Connection strings via user-secrets or environment variables, never in `appsettings*.json`.

### Events

| Kind | Scope | Mechanism |
|---|---|---|
| **Domain events** | Inside one service | Raised by aggregates; side effects inside the same service run in the same handler and the same save |
| **Integration events** | Between services | RabbitMQ, versioned contracts in `src/shared/contracts`, written to an **outbox in the same transaction** as the business change |

**Messaging implementation: `RabbitMQ.Client` + a hand-written outbox and inbox.** No MassTransit (commercial licence from v9). Wolverine (free) is the fallback if this ever becomes complex.

- **Outbox:** integration events are written to `outbox_messages` **inside `SaveChanges`**, in the **same database transaction** as the business change (an EF Core interceptor converts the aggregate's domain events into outbox rows before the commit). Never write outbox rows after `SaveChanges`: a crash in between would lose the message.
- **Publisher:** a background service reads pending outbox rows, publishes them to RabbitMQ and marks them sent. Failed publishes are retried.
- **Inbox:** consumers store each processed message ID in `inbox_messages` and skip duplicates.
- Event consumers are **idempotent** (safe to run twice) and log failures.

### Authentication

- **Keycloak** handles login, users, passwords and tokens (OpenID Connect / JWT). There is no custom identity service.
- The **gateway validates the JWT** and **forwards the JWT unchanged** to services.
- **Each service also validates the JWT** (ASP.NET Core JwtBearer against Keycloak; PyJWT or equivalent in Python). Services **never trust a user-ID header**: anyone who could reach a service directly could otherwise impersonate any user.
- Services are reachable only inside the Docker network; only the gateway (and Keycloak's login endpoints) are exposed publicly.
- Code reads the user through an `ICurrentUser` abstraction backed by the validated token's `sub` claim (the Keycloak user ID).
- User-owned data is filtered by `UserId` with EF Core **global query filters**; tests check that users cannot see each other's data. Row-Level Security is a later addition.

## Money rules

| Item | Rule |
|---|---|
| Type | `decimal` everywhere, never `double` or `float` |
| Currency | `Money` value object; LKR only for now |
| Share prices | 2 decimal places |
| Quantity | Whole shares (`long`) |
| Fees | Trade value × fee rate, **rounded to 2 dp, half away from zero, per transaction** |
| Average cost | Stored at higher precision (6 dp), displayed at 2 dp |
| Totals and P&L | Calculated from full-precision values; rounded only for display |

## Python services

Example for analytics:

```
services/analytics/
├── app/
│   ├── api/            # FastAPI routes
│   ├── core/           # Config, logging, settings
│   ├── features/       # Feature engineering (returns, volatility, indicators)
│   ├── models/         # Baselines, LightGBM quantile, conformal calibration
│   ├── pipelines/      # Daily forecast, calibration, weekly retraining
│   ├── evaluation/     # Backtesting, accuracy tracking, champion vs challenger
│   ├── db/             # Database access
│   └── messaging/      # Event consumers (e.g. PricesUpdated)
├── notebooks/          # Research only, not production code
├── tests/
├── pyproject.toml
└── Dockerfile
```

Knowledge follows the same pattern with `ingestion/` (download PDFs), `parsing/` (text, tables, OCR), `extraction/` (financial numbers) and `rag/` (embeddings, retrieval, answers).

## Rules between services

1. **Each service owns its database.** No cross-database queries.
2. **Shared code is minimal.** `src/shared/dotnet` holds building blocks only; `src/shared/contracts` holds versioned contracts. Never business logic.
3. **Communication:** HTTP via the gateway for the frontend; RabbitMQ integration events between services.
4. **Contracts are versioned** (`PricesUpdated.v1`).
5. **Each service has its own Dockerfile and tests** and can be deployed alone.
6. **Data from other services** is obtained by events (keeping a local copy where needed, e.g. latest prices in portfolio) or API calls, never by reading their tables.

## Portfolio domain rules

**Aggregates**

| Aggregate | Owns |
|---|---|
| `Portfolio` | Name, owner (`UserId`), default fee rate (1.12%), the user's tag list |
| `Holding` (one portfolio + one stock) | Lots, sells, sell-lot allocations, undo/restore, dividends received, corporate-action adjustments |

Two changes on different stocks never block each other, and every lot rule works within one stock.

**Storage: append-only transactions, not event sourcing.** Transactions are rows that are never deleted. The `Holding` is loaded from them and the domain calculates quantity, average cost, lot balances and P&L in memory.

**Rules**

- Every buy is stored as a lot; lots are never merged.
- **Every sell chooses its lot(s).** The sell window pre-selects the oldest lot (FIFO) as a convenience; the user can change it or split across lots (e.g. 30 from Lot A + 20 from Lot B). There is no "average" sell. The allocation is stored.
- **Realized P&L is always based on the chosen lots' cost.**
- **Holdings display:**
  - **Portfolio page:** one table, **one row per stock**, using average cost (quantity, average cost, price, value, unrealized P&L).
  - **Lot breakdown per stock only:** expanding a row (or opening the stock's detail view) shows that stock's lots (buy date, quantity, buy price, value, unrealized P&L per lot), next to the Buy and Sell actions.
  - The stock detail view also shows the stock's **realized P&L** (past sells) and **dividends received**.
  - Both levels show the same remaining shares and the **same total unrealized P&L**; the lot breakdown only splits it. Display never changes any number.
- Only sells can be undone, at any point in history. Undo is a status change plus an audit record; nothing is deleted. Undone sells can be restored.
- Editing or deleting a buy is blocked if it affects sold shares: "Undo the related sells first".

**Fees**

- Each buy and sell **stores its actual fee amount** and the rate used (e.g. fee 28.00 at 1.12%).
- The portfolio's fee rate (default 1.12%) is **only the default that pre-fills** the fee in the buy/sell window.
- The fee is **editable on every transaction** (e.g. to match the broker's contract note, or 0.00 for IPO subscriptions).
- Changing the portfolio's rate affects **future transactions only**; stored fees never change.
- Fees are added to buy cost and subtracted from sell proceeds.
- CSE rates are tiered (lower for large trades). For now the default is a flat rate and the user edits the fee for large trades; automatic tier calculation can come later.

**Tags**

- Each buy and sell can have **zero or more optional tags**.
- The tag list starts with **Investment**, **IPO** and **Trade**; users can add their own tags.
- Tags are **labels only**: they never affect P&L, lots or fees. They can be edited at any time.
- Used for filtering transaction history and, later, P&L by tag.

**Worked example**

| Lot | Qty | Buy price | Cost |
|---|---|---|---|
| A | 100 | 25.00 | 2,500 |
| B | 100 | 28.00 | 2,800 |

Sell 50 @ 27.50 (fees ignored for clarity):

| Lot chosen | Realized P&L | Remaining | Portfolio row (average) | Lot breakdown |
|---|---|---|---|---|
| A | 50 × (27.50 − 25.00) = **+125** | A 50, B 100 (cost 4,050) | 150 @ 27.00 | A 50 @ 25.00, B 100 @ 28.00 |
| B | 50 × (27.50 − 28.00) = **−25** | A 100, B 50 (cost 3,900) | 150 @ 26.00 | A 100 @ 25.00, B 50 @ 28.00 |

Unrealized P&L = current price × remaining quantity − remaining cost. At a price of 30.00:

| Lot chosen | Realized | Unrealized | **Total** |
|---|---|---|---|
| A | +125 | 150 × 30 − 4,050 = +450 | **+575** |
| B | −25 | 150 × 30 − 3,900 = +600 | **+575** |

**Total profit does not depend on the lot choice.** Total profit = sell proceeds (1,375) + value of held shares (4,500) − cost of all buys (5,300) = +575 in both cases. The lot choice only decides how the total cost is split between sold shares (realized) and held shares (unrealized). Per-transaction fee rounding can cause tiny differences.

**Corporate actions** (applied only after the user confirms)

- `ApplySplit(ratio)` and `ApplyBonus(ratio)` adjust **every open lot**: quantity changes, total cost stays the same, so cost per share falls.
- Fractional shares from bonus issues are **rounded down**; the fraction (and any cash paid for it) is recorded.
- Rights issues: subscribing creates a **new lot** at the rights price; selling or letting rights lapse is recorded separately.

## Forecasting rules

- Output: 70% range + bias + confidence for 1, 5 and 20 days.
- Models: baselines (random walk, historical volatility) → LightGBM quantile regression pooled across stocks → conformal calibration.
- Feedback loop: daily adaptive calibration, weekly retraining, champion vs challenger.
- Walk-forward backtesting; no look-ahead bias; liquidity filter; every forecast and model version stored.
