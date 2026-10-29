# System Overview

## Purpose

A web app for Colombo Stock Exchange (CSE) investors that combines:

- **Portfolio tracking**: lots, sells with lot selection, undo of sells, dividends, fees, corporate actions
- **Research and analysis**: charts, indicators, screener, market overview, risk stats
- **Forecasting**: probabilistic price ranges (1 / 5 / 20 days) with a feedback loop
- **Knowledge base** (phase 2): CSE financial reports with extracted numbers and RAG Q&A

Used privately by a small group. Web first; a mobile app can be added later and reuse the same APIs.

## High-level diagram

```
                    React + Tailwind (web)
                     │ login (OIDC)      │ HTTPS + JWT
                     ▼                   ▼
                 Keycloak ◀──validates── API Gateway (.NET + YARP)
                 (identity)   tokens     routing · JWT validation
                                              │
             ┌────────────────┬───────────────┴──┬────────────────┐
             ▼                ▼                  ▼                ▼
         Portfolio        Market Data        Analytics        Knowledge
         (.NET)           (.NET)             (Python)         (Python)
         holdings, lots,  CSE ingestion,     indicators,      report PDFs,
         sells, undo,     prices, corp.      risk, forecasts, extraction,
         dividends        actions, ann.      feedback loop    RAG
             │                │                  │                │
             ▼                ▼                  ▼                ▼
        portfolio_db     marketdata_db       analytics_db     knowledge_db
                        (one PostgreSQL server; Keycloak has its own keycloak_db)

Shared infrastructure: RabbitMQ (events) · S3-compatible storage (files)
External: cse.lk endpoints (market data, announcements, reports) · Claude API (extraction, RAG)
```

## Services

| Service | Language | Responsibility | MVP |
|---|---|---|---|
| Gateway | .NET (YARP), single project | Single public entry point, routing, JWT validation, forwards the JWT to services (which validate it again) | ✅ |
| Keycloak | Off-the-shelf | Login, users, passwords, roles, tokens (OpenID Connect / JWT). Replaces a custom identity service | ✅ |
| Portfolio | .NET | Portfolios, holdings, lots, sells, undo/restore, dividends received, fees, corporate-action adjustments | ✅ |
| Market Data | .NET | CSE ingestion, companies, sectors, prices, indices, announcements, dividends, corporate actions; bulk price API | ✅ |
| Analytics | Python | Indicators, risk metrics, forecasting, backtesting, accuracy tracking | ✅ |
| Knowledge | Python | Report download, parsing, financial extraction, RAG | Phase 2 |
| Notifications | — | Price and dividend alerts | Later |

## Communication

- **Frontend → backend:** HTTP through the gateway only, with a Keycloak-issued JWT. The gateway forwards the JWT and each service validates it; services are not exposed outside the Docker network.
- **Service → service (asynchronous, preferred):** integration events through RabbitMQ, published via an outbox.
- **Service → service (synchronous):** HTTP, only when an immediate answer or a bulk read is needed.

Example events (versioned contracts in `src/shared/contracts`):

| Event | Published by | Consumed by |
|---|---|---|
| `PricesUpdated.v1` | Market Data | Portfolio (revalue), Analytics (sync prices, recompute forecasts) |
| `DividendAnnounced.v1` | Market Data | Portfolio (expected dividends) |
| `CorporateActionDetected.v1` | Market Data | Portfolio (ask user to confirm) |

Integration events are only for communication **between** services. Steps inside one service (e.g. Knowledge: download → parse → extract → embed) use an internal job queue, not RabbitMQ integration events.

**Bulk price data:** Analytics does not receive price history through events. After `PricesUpdated`, it calls the market-data **bulk price API** (`GET /api/prices?symbols=…&from=…&to=…`) and keeps its own copy in `analytics_db`.

## Trading calendar

The CSE is closed on weekends, **Poya days** (about 12 per year, often weekdays), public and bank holidays, and occasional special closures.

- **Market Data owns the trading calendar:** a table of trading days and holidays with a reason (Poya, New Year, …).
- **Filled** from CSE's published holiday list where possible, with manual entry as a fallback (once a year). If the daily job finds no trading on an expected trading day, it flags the day for review.
- The **daily job runs only on trading days**.
- **Forecast horizons count trading days** (1, 5 and 20 trading days), not calendar days.
- Analytics gets the calendar from Market Data through its API, like prices.
- Dividend dates (XD, payment) are checked against the calendar.

## Daily flow

```
14:30  CSE closes
  → Market Data fetches the day's prices, announcements, dividends, corporate actions
  → publishes PricesUpdated (via outbox)
      → Portfolio updates its latest-price table and revalues holdings
      → Analytics syncs new prices through the bulk API
                   records actuals vs previous forecasts
                   updates calibration (feedback loop)
                   generates new 1/5/20-day forecasts
  → Knowledge downloads any new financial reports (phase 2)
Weekly: Analytics retrains models; champion vs challenger decides which model is live
```

## Build order

Build so that something useful runs early, and add infrastructure only when it is needed.

| Step | Build | Result |
|---|---|---|
| 1 | Shared building blocks + **Portfolio service** (prices entered manually) | Core portfolio rules working and tested |
| 2 | **Market Data** (CSE ingestion) + `PricesUpdated` + outbox/inbox | Real prices; portfolio revalues automatically |
| 3 | **Gateway + Keycloak** | Secure multi-user access |
| 4 | **Analytics** | Indicators, risk, forecasts with feedback loop |
| 5 | **Knowledge** | Financial reports, fundamentals, RAG |

The web frontend grows alongside each step.

## Key principles

1. **Business logic lives in the backend**, never in the frontend.
2. **Each service owns its data.** No service reads another service's database.
3. **Market data is behind a provider interface**, so the source can be replaced.
4. **Forecasts are probabilistic and calibrated**, labelled as educational, not advice.
5. **Don't build what can be reused safely:** authentication comes from Keycloak.
6. **Keep it simple until scale demands otherwise.** One VM, Docker Compose, one PostgreSQL server.
