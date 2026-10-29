# Costs and Licences

Quick reference for running costs and licence risks. Prices are approximate (October 2026); check before deciding.

## Running costs (monthly)

| Item | Estimated cost | Notes |
|---|---|---|
| VM / VPS | ~$10–100/month | Main cost. Hetzner cheapest; Azure / AWS higher |
| **Claude API** | Variable, **main cost to watch** | Report extraction (mostly one-time backlog), summaries, RAG questions. Set a monthly spending limit |
| Embeddings for RAG | $0 or small | Claude has no embeddings API. Free local model (e.g. sentence-transformers) or paid API (e.g. Voyage AI) |
| Backblaze B2 | ~$0–1/month | First 10 GB free |
| Domain name | ~$10–15/year | `.lk` domains usually cost more |
| SSL certificate | Free | Let's Encrypt |
| Email sending | Free tier | Keycloak password resets and invites (e.g. Brevo, Resend) |
| Off-server backups | ~$0–1/month | Copy nightly database backup to B2 |
| GitHub | Free | Private repos free; Actions has 2,000 free minutes/month for private repos |
| Monitoring | Free | Self-hosted Grafana + Prometheus, or free tiers (e.g. Sentry) |
| Keycloak | Free | Apache 2.0; ~0.5–1 GB RAM on the VM. Paid Red Hat support is optional and not needed |

**Expected total for a private app with 3–4 users: about $15–40/month**, mostly the VM and Claude API usage.

## Licence watch list

| Package / tool | Status | Action |
|---|---|---|
| MediatR, AutoMapper, MassTransit v9 | Commercial | ✅ Avoided |
| Redis | Licence changed in 2024 (source-available, later with an AGPL option) | ⚠️ Use **Valkey** (BSD fork, drop-in) instead |
| MinIO | AGPL; community edition reduced in 2025 | ⚠️ Development only. Alternatives: Garage, SeaweedFS |
| PyMuPDF | **AGPL** (commercial licence sold separately) | ⚠️ Prefer **pdfplumber** or **pypdf** (MIT/BSD). AGPL can force source release if the app becomes public |
| TimescaleDB | Apache core + Timescale License for some features | ✅ Free for self-hosting; restricted only when selling it as a database service |
| Docker Desktop | Free for personal use and small companies; paid for 250+ employees or $10M+ revenue | ✅ Free for this project. Docker Engine on Linux is always free |
| TradingView Lightweight Charts | Apache 2.0 | ✅ Free; **show the TradingView attribution** |
| JetBrains Rider | Free for non-commercial use | ✅ Fine now; paid for commercial use. Visual Studio Community and VS Code are free |
| Keycloak, RabbitMQ, PostgreSQL, pgvector, YARP, EF Core, Npgsql, FluentValidation, xUnit, Testcontainers, LightGBM, scikit-learn, pandas, FastAPI, MLflow, React, Tailwind, ECharts, Recharts | Open source (MIT / Apache / BSD / PostgreSQL / MPL) | ✅ Free |

**Rule for new packages:** check the licence, and whether it changed recently (MediatR, AutoMapper, MassTransit, Redis and MinIO all did), before adding it.

## Costs only if the app becomes public or paid

| Item | Notes |
|---|---|
| **CSE data licence** | Commercial use of CSE data likely needs a licence from CSE. Biggest unknown cost |
| Legal / SEC Sri Lanka review | Forecasts and analytics may count as investment advice |
| Payment gateway | e.g. PayHere: percentage fee per transaction |
| Bigger servers, managed databases | As users grow |
| AGPL libraries | Must be replaced or licensed commercially |
| Support and monitoring tools | Paid tiers once there are real customers |

## Recommended changes (not yet applied to the architecture docs)

1. ~~Valkey instead of Redis~~ **Applied:** no cache server for now; Valkey if one is needed later
2. pdfplumber / pypdf instead of PyMuPDF in the knowledge service
3. A free local embedding model for RAG
4. A monthly spending limit on the Claude API key
5. A licence check rule for new packages in `CLAUDE.md`
