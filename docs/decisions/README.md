# Architecture Decisions

Key decisions made during planning (October 2026).

| # | Decision | Choice | Reason |
|---|---|---|---|
| 1 | Users | Private app for the owner and 2–3 friends | Non-commercial for now |
| 2 | Platform | Web first, mobile later | API-first so mobile can reuse the backend |
| 3 | Repository | One monorepo (backend + frontend) | Solo developer; one change touches several services; easy to split later |
| 4 | Architecture | Microservices: gateway, portfolio, market-data (.NET); analytics, knowledge (Python); Keycloak for identity | Clear boundaries; ready for a team later |
| 5 | .NET service structure | Vertical slices + CQRS + rich domain: two projects per service (`*.Domain` pure, `*.Api` organised by feature); own CQRS interfaces, Result pattern, no MediatR/AutoMapper | One folder per feature; pure, testable domain; fewer projects per microservice |
| 6 | Databases | One PostgreSQL server, one database per service | Isolation without extra cost; easy to split later |
| 7 | Tenancy | Multi-tenant, shared tables; `user_id` on private data; EF Core global query filters now, Row-Level Security later | Market data shared; portfolios private; RLS is extra work with pooling |
| 8 | Communication | HTTP via gateway; events via RabbitMQ; versioned contracts | Loose coupling between services |
| 9 | Market data source | Free unofficial `cse.lk` endpoints behind a provider interface, plus user-supplied historical files | ICE too expensive; Apify costly; stockanalysis.com terms forbid scraping and ML use; MyCSE has no API |
| 10 | Financial reports | CSE report PDFs → extracted numbers + RAG (pgvector + Claude API) | Provides fundamentals and Q&A from data we own |
| 11 | File storage | S3-compatible: MinIO locally, Backblaze B2 live | Cheap, separates storage from compute |
| 12 | Hosting | One VM/VPS (Singapore or Mumbai) with Docker Compose | Cheapest; fits TimescaleDB and long-running jobs |
| 13 | Forecasting | Probabilistic ranges (1/5/20 days), LightGBM quantile + conformal calibration, feedback loop | Honest, calibrated output; no RL for now |
| 14 | Charts | TradingView Lightweight Charts + ECharts/Recharts | Free and professional |
| 15 | Portfolio storage | Append-only transactions; `Holding` loaded from them and the domain calculates holdings and P&L in memory. Not event sourcing | Full history and safe undo without an event store |
| 16 | Portfolio aggregates | `Holding` (one portfolio + one stock) is the main aggregate; `Portfolio` is small (name, owner, fees) | Small aggregates; all lot rules are per stock; no locking across stocks |
| 17 | Analytics price data | Bulk price API on market-data; analytics keeps its own copy | Clean boundary; history is too large for events |
| 18 | Messaging | `RabbitMQ.Client` + hand-written outbox and inbox. Wolverine as fallback | Free (MassTransit v9 is commercial); fits own CQRS; small at this scale |
| 19 | Identity | Keycloak (OpenID Connect / JWT), validated at the gateway. No custom identity service | Don't write security-critical auth code; industry standard; one less service |
| 20 | Money | `decimal`; `Money` value object (LKR); prices 2 dp; whole-share quantities; fees rounded to 2 dp half away from zero per transaction; average cost stored at 6 dp | Avoid rounding drift and floating-point errors |
| 21 | Build order | Portfolio → Market Data → Gateway + Keycloak → Analytics → Knowledge | Something useful runs early; infrastructure added when needed |
| 22 | Service authentication | Gateway forwards the JWT; every service validates it again; no trusted user-ID headers; only the gateway is public | Prevents user impersonation if a service is reached directly |
| 23 | Outbox timing | Integration events written to the outbox inside `SaveChanges` (same transaction), via an EF Core interceptor | Writing after the save could lose messages on a crash |
| 24 | Concurrency with append-only rows | Every aggregate change updates the root row so `xmin` changes | Child-only inserts would not change `xmin`, allowing overselling |
| 25 | Hand-written migration SQL | Allowed only for extensions, TimescaleDB hypertables and seed data | EF Core cannot create these |
| 26 | Cache | No cache server for now; Valkey (not Redis) if needed later | Nothing needs a cache yet; Redis licence changed |
| 27 | Integration event scope | Only between services; steps inside a service use internal jobs | Keeps RabbitMQ for cross-service communication only |
| 28 | Sell method and P&L | Every sell chooses its lot(s), FIFO pre-selected, can split; no "average" sell. Realized P&L uses the chosen lots. Portfolio page shows one row per stock (average); lot breakdown only per stock | Keeps Lot view, Average view, realized P&L and undo consistent |
| 29 | Fees | Each transaction stores its actual fee amount and rate; portfolio rate only pre-fills; editable per transaction; rate changes affect future transactions only; tiered rates edited manually for now | Contract note is the truth; old P&L must never change silently |
| 30 | Transaction tags | Optional, zero or more per buy/sell; starts with Investment, IPO, Trade; user-defined tags allowed; labels only | Filtering and later P&L by category without affecting calculations |
| 31 | Time zone | Timestamps in UTC; trading dates as `DateOnly` in Colombo time; "today" and jobs use `Asia/Colombo` via `IClock`; UI shows Colombo time | CSE runs on UTC+5:30; avoids off-by-one dates |
| 32 | Trading calendar | Market Data owns a CSE trading calendar (weekends, Poya days, holidays); filled from CSE lists + manual fallback; daily job runs on trading days only; forecast horizons count trading days | Avoids wrong forecasts, false job errors and wrong return calculations |
