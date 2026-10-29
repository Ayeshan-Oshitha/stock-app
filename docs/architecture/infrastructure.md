# Infrastructure

## Hosting: one virtual machine

**Decision:** one VM / VPS running everything with Docker Compose, in the **Singapore or Mumbai** region (closest to Sri Lanka).

| Option | Verdict |
|---|---|
| **VM / VPS + Docker Compose** | ✅ Chosen. Cheapest, same setup as local development, supports TimescaleDB and long-running jobs |
| Azure App Service | ⚠️ Pay per service plus managed database; adds up quickly |
| AWS Lambda | ❌ 15-minute limit, no long-running WebSockets, cold starts. Supports .NET and Python, but wrong fit for the core app |
| Kubernetes | ❌ Overkill for now |

Candidate providers (choose at deployment time): Hetzner, DigitalOcean, AWS Lightsail, Azure VM. Target size: about 4 vCPU and 8–16 GB RAM.

**Containers on the VM**

```
gateway · keycloak · portfolio · market-data · analytics · knowledge · web
postgres (TimescaleDB + pgvector) · rabbitmq
```

Only the gateway and Keycloak's login endpoints are exposed publicly (behind HTTPS). All other containers are reachable only inside the Docker network.

**No cache server for now.** Nothing needs one yet. If caching becomes necessary, use **Valkey** (BSD-licensed Redis fork) rather than Redis (licence changed in 2024).

## File storage: S3-compatible

| Environment | Storage |
|---|---|
| Local development | MinIO in Docker (check the current state of MinIO's community edition; Garage or SeaweedFS are alternatives) |
| Live | Backblaze B2 |

- Code uses the S3 API only, so switching provider is a configuration change.
- Storage is kept separate from the VM so the server is sized for compute, not files, and files survive a server rebuild.
- Expected size: a few GB per year of CSE reports.

## Data sources

| Need | Source |
|---|---|
| Prices, indices, sectors, announcements, financial report links | Free unofficial `cse.lk` endpoints (behind a provider interface) |
| Older price history if needed | User-supplied CSV / text files, imported once |
| Fundamentals | Extracted from CSE financial report PDFs |
| LLM for extraction and RAG | Claude API |

**Usage rules for `cse.lk`:** call gently (low frequency, mainly after market close), store everything locally, and handle endpoint changes gracefully.

**Rejected sources**

| Source | Reason |
|---|---|
| ICE | Enterprise contracts, no published CSE price, far too expensive |
| Apify scraper | About $0.05 per record, no history, same legal position as the free source |
| stockanalysis.com | Terms forbid scraping, bulk collection and ML training |
| MyCSE packages | Personal web/app subscriptions, no API |

## Live updates (later)

During market hours (09:30–14:30), prices can be polled every 1–2 minutes and pushed to the browser with SignalR. The MVP uses end-of-day updates only.
