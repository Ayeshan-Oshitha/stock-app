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

## Recommendations not yet applied

From `docs/costs-and-licences.md`:

- pdfplumber / pypdf instead of PyMuPDF in the knowledge service
- Free local embedding model for RAG
- Monthly spending limit on the Claude API key
- Licence check rule for new packages in `CLAUDE.md`

## Minor

- `.editorconfig` forces LF for all files while `.gitattributes` forces CRLF for `*.csv` (harmless; CSVs are not committed).
