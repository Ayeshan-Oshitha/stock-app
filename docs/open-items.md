# Open Items

Topics still to discuss and document. Pick up from here in the next session.

## Decisions still to make

None at the moment.

## Documentation still to write

| Topic | Why it matters |
|---|---|
| Event contract format | Contracts must work for both .NET and Python: language-neutral JSON with an envelope (id, type, version, timestamp) |
| Python rules | Equivalent of the .NET rules: Ruff, typing, tests, FastAPI conventions, outbox/inbox and JWT validation in Python |
| Frontend architecture | Routing, state, OIDC login flow (PKCE) with Keycloak, API client generated from OpenAPI |
| Market Data domain | Aggregates for companies, prices, corporate actions, dividends, trading calendar |
| Ownership of watchlists and alerts | Which service owns them (likely Portfolio for watchlists, Notifications for alerts) |
| Operations | Logging, health checks, tracing (OpenTelemetry), environments, production secrets, backup restore tests, CI/CD |
| `portfolio_db` table design | First service in the build order |

## Agreed, not yet in decisions

- **Company list (CSV first):** Market Data starts with a CSV of CSE securities, same shape as the future `marketdata_db.companies` table, read behind an interface; switched to EF Core + PostgreSQL later. Columns: `id` (GUID), `symbol`, `name`, `sector`, `status` (Active/Suspended/Delisted), `isin`, `share_type` (Voting/NonVoting), `board` (Main/DiriSavi/Empower), `listed_on`, `delisted_on`.
- **One row per security** (e.g. `JKH.N0000` and `JKH.X0000` are separate rows). Later split into `companies` and `securities` tables.
- **Portfolio keeps a read-only copy** of companies in `portfolio_db` (id, symbol, name, sector, status), filled from the CSV for now and from Market Data events later. Two tables in production.
- **Local development database:** PostgreSQL in Docker (`timescale/timescaledb-ha`, includes pgvector), one container with one database per service. Not Neon.

## Recommendations not yet applied

From `docs/costs-and-licences.md`:

- pdfplumber / pypdf instead of PyMuPDF in the knowledge service
- Free local embedding model for RAG
- Monthly spending limit on the Claude API key
- Licence check rule for new packages in `CLAUDE.md`

## Minor

- `.editorconfig` forces LF for all files while `.gitattributes` forces CRLF for `*.csv` (harmless; CSVs are not committed).
