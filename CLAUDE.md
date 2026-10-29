# CLAUDE.md

Guidance for Claude (and developers) working in this repository.

## Code ownership: the most important rule

**The developer writes all code. Claude guides and reviews.**

- Never create, edit, rename or delete code or build files unless the developer explicitly asks for that specific change in the current message. This covers `*.cs`, `*.csproj`, `*.props`, `*.slnx`, `global.json`, `appsettings*.json`, migrations, `*.py`, `pyproject.toml`, `*.ts`, `*.tsx`, `package.json` and Docker files.
- Claude's role: explain concepts, propose designs, point to best practices, and review.
- When asked to review, list each problem with file and line, explain why it is a problem, and show the fix as a snippet **in the reply**. Do not apply it.
- Claude may create and update documentation when asked: everything under `docs/`, `README.md`, and all `CLAUDE.md` files.
- Running read-only commands (`dotnet build`, `dotnet test`, `git status`, `git diff`) to check the developer's work is allowed and encouraged.
- When the developer runs a new command, add it to `docs/commands.md` (when asked).
- **Commit messages:** when asked, reply with a single imperative line under 72 characters. Never run `git commit` or `git push`.

## Project

Portfolio tracker + research + analytics + forecasting for the Colombo Stock Exchange (CSE).
Private app for a small group of users.

- Architecture: `docs/architecture/` (start with `overview.md`; .NET and Python rules in `backend.md`)
- Key decisions: `docs/decisions/README.md`
- Planned features: `docs/future-work.md`
- Command log: `docs/commands.md`
- Costs and licence risks: `docs/costs-and-licences.md`
- **Open items (start here when resuming planning):** `docs/open-items.md`

## Architecture rules

- **Monorepo.** Services in `src/services/`, gateway in `src/gateway/`, frontend in `src/web/`.
- **Microservices with strict boundaries.**
  - Each service owns its own database and never reads another service's database.
  - Frontend → gateway over HTTP. Service ↔ service via RabbitMQ integration events (`RabbitMQ.Client` + hand-written outbox/inbox, no MassTransit), HTTP only for immediate answers or bulk reads (e.g. analytics → market-data bulk price API).
  - Shared code only in `src/shared/dotnet` (building blocks) and `src/shared/contracts` (versioned contracts). **No business logic in `shared/`.**
- **.NET services: two projects each.** `<Service>.Domain` (pure rich domain) and `<Service>.Api` (vertical slices in `Features/<UseCase>/`, plus `Common/` and `Infrastructure/`). **Exception:** the gateway is a single project (routing, JWT validation; no business logic).
- **Integration events only between services**, written to the outbox **inside `SaveChanges`** (same transaction). Steps inside one service use internal jobs, not RabbitMQ.
- **No cache server for now** (Valkey later if needed, not Redis).
- **Business logic lives in the backend**, never in the frontend.
- **Market data access goes through a provider interface.**
- **Identity: Keycloak** (OpenID Connect / JWT). The gateway validates and forwards the JWT; **each service validates it again** and never trusts a user-ID header. Code reads the user via `ICurrentUser`. Only the gateway is exposed publicly. No custom identity service.
- **Multi-tenant:** user-owned rows carry `user_id`, filtered by EF Core global query filters (Row-Level Security later); market data is shared.
- **Build order:** Portfolio → Market Data → Gateway + Keycloak → Analytics → Knowledge.

## .NET rules (summary; full rules in `docs/architecture/backend.md`)

- **Rich domain:** no public setters; intention-revealing methods; static factories returning `Result<T>`; strongly-typed Guid v7 IDs; enums for statuses; `IReadOnlyCollection` for children; time passed in, never `DateTime.Now`.
- **Result pattern:** expected failures return `Result`/`Error`; exceptions only for bugs. `ErrorType` → 400 / 404 / 409 / 422 ProblemDetails.
- **CQRS:** own `ICommand`/`IQuery`/handler interfaces. **No MediatR, no AutoMapper.** Handlers contain no business rules. Queries use `AsNoTracking` + `Select` projections. Validators check input shape only.
- **API:** Minimal APIs, built-in OpenAPI, thin endpoints, request/response records separate from domain types, never return entities.
- **Persistence:** EF Core + PostgreSQL, snake_case, Fluent API only, enums as strings, `xmin` concurrency tokens (every change updates the aggregate root row), generated migrations (hand-written SQL only for extensions, hypertables and seed data), secrets via user-secrets / environment variables.
- **Time:** timestamps stored in UTC; trading dates are `DateOnly` in Colombo time; "today" and scheduled jobs use `Asia/Colombo` via `IClock`.
- **Tests:** every domain behaviour has unit tests; handlers and endpoints have tests (xUnit, Testcontainers for integration).

## Portfolio rules

- Aggregates: `Holding` (one portfolio + one stock: lots, sells, allocations, undo/restore, corporate actions) and a small `Portfolio` (name, owner, fees).
- Storage: append-only transactions (never deleted); the domain calculates holdings and P&L in memory. Not event sourcing.
- Every buy is a lot. **Every sell chooses its lot(s)** (FIFO pre-selected, can split); there is no "average" sell. Realized P&L uses the chosen lots' cost.
- Display: portfolio page has one row per stock (average cost); the lot breakdown is shown only per stock (expand / detail view). Both show the same total unrealized P&L. Lot choice changes the realized/unrealized split, never the total profit.
- Only **sells** can be undone, at any point in history. Undone sells are kept and can be restored.
- Editing or deleting a buy is blocked if it affects sold shares ("Undo the related sells first").
- Fees: each transaction stores its **actual fee amount** (editable); the portfolio rate (default 1.12%) only pre-fills it and changes affect future transactions only.
- Optional tags on buys/sells (Investment, IPO, Trade + user-defined): labels only, never affect calculations.
- Money: `decimal`, prices 2 dp, whole shares, fees rounded to 2 dp (half away from zero) per transaction, average cost stored at 6 dp.
- Corporate actions adjust every open lot after user confirmation; bonus fractions are rounded down and recorded.

## Forecasting rules

- Output probabilistic ranges with calibrated probabilities, never single exact prices.
- Horizons (1 / 5 / 20) are **trading days**, using Market Data's CSE trading calendar (weekends, Poya days, holidays).
- Every model must beat baselines in walk-forward backtests. No look-ahead bias.
- Forecasts are labelled as educational analysis, not investment advice.

## Conventions

- .NET SDK pinned in `global.json`; shared build settings in `Directory.Build.props`; NuGet versions only in `Directory.Packages.props`.
- Python services use `uv` and a `pyproject.toml` per service.
- Never commit secrets, data files, PDFs or trained models. Only `.env.example` templates are committed.
- Files used inside Linux containers (Dockerfiles, compose files, shell scripts, Python) use LF line endings.
- Explanations: short and in plain sentences. Show a snippet when it's clearer than prose.
