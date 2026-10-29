# Stock App

Portfolio tracking, research, analytics and forecasting for the **Colombo Stock Exchange (CSE)**.

> Private project for a small group of users. Forecasts are educational analysis, not investment advice.

## Architecture

```
React + Tailwind (web) ──login──▶ Keycloak
        │
   API Gateway (.NET + YARP, validates JWT)
        │
 ┌──────────────┬──────────────┬──────────────┐
Portfolio     Market Data     Analytics      Knowledge
(.NET)        (.NET)          (Python)       (Python)
        │
PostgreSQL (one DB per service, TimescaleDB, pgvector) · RabbitMQ · S3-compatible storage
```

## Repository layout

| Path | Contents |
|---|---|
| `src/gateway/` | API Gateway |
| `src/services/` | Backend microservices |
| `src/shared/` | Shared helpers and contracts (no business logic) |
| `src/web/` | React frontend |
| `deploy/` | Docker Compose and environment templates |
| `docs/` | Architecture decisions and planning |
| `scripts/` | Setup and data import scripts |

## Prerequisites

- .NET SDK 10 (pinned in `global.json`)
- Python (managed with [uv](https://docs.astral.sh/uv/))
- Node.js 22+
- Docker

## Getting started

Setup steps will be added as services are built.
