# Performance & Scalability Strategy

---

## 9.1 Expected Load (MVP)

| Phase | MAU | DAU (≈20%) | RPS peak (estimate) | WS concurrent |
| --- | --- | --- | --- | --- |
| Beta (Month 0–1) | 100 | 20 | 5–10 | ≤ 50 |
| Soft Launch (Month 1–3) | 1 K | 200 | 30–50 | ≤ 500 |
| Early MVP (Month 3–6) | 5 K | 1 K | 80–150 | ≤ 1 K |
| Growth (Month 6–12) | 10–25 K | 2–5 K | 150–250 | ≤ 3 K |

Daily data growth at 5K MAU: ~50 MB chat ciphertext + ~1 GB photos. Neon storage stays under Launch-plan quota.

---

## 9.2 Bottleneck Analysis

| Component | Likely degradation | Trigger | Action |
| --- | --- | --- | --- |
| Discovery GIN query | Feed p95 > 500 ms | > 50 K active profiles | Tune index / add Meilisearch / extract module + add Elasticsearch |
| DB connections | "too many connections" | concurrent users > 500 | Upgrade Neon plan (PgBouncer already in path) |
| WebSocket memory | RAM > 80% | concurrent WS > 3 K | Scale Railway to 2 GB or 2× replicas (sticky sessions via WS routing) |
| Upstash commands | Upstash 429 | > 5 M cmd/month | Upstash fixed plan ($10/mo) |
| S3 egress | bill > $50/mo | > 25 K active users | Add CloudFront for media |
| Single-process CPU | > 80% sustained | any spike | 2× replicas (Fastify is stateless except WS — see sticky-routing note) |

---

## 9.3 Horizontal vs Vertical Scaling

| Concern | Strategy |
| --- | --- |
| HTTP REST | Stateless — scale horizontally by replica count behind Railway router. |
| WebSocket | Each WS pinned to a process. Phase 1 keeps 1 replica → no sticky routing needed. Phase 1.5 (multi-replica) uses Upstash pub/sub to fan messages between replicas. |
| DB | Vertical (Neon plan upgrade) until 25 K MAU; then read replicas for discovery query if needed. |
| Redis | Upstash auto-scales; vertical via plan upgrade. |
| S3 | Inherently scalable. |
| `modules/chat` | First candidate for horizontal extraction at the microservices boundary. |

---

## 9.4 Caching Strategy

| Layer | What | TTL | Why |
| --- | --- | --- | --- |
| Upstash | `block:{userId}` Set | 1 h / invalidate on mutation | Hot-path discovery exclude. |
| Upstash | `geo:{prefix5}` Sorted Set of active userIds | 5 min | Avoid Postgres scan for adjacency. |
| Upstash | `q:like:{userId}:{date}` counter | 24 h | Daily quota. |
| Upstash | `presence:{userId}` boolean | 90 s (heartbeat) | WS presence + online indicator. |
| In-process | Prisma client query cache for identity options | 10 min | Static-ish reference data. |
| Mobile | `dio_cache_interceptor` on `/v1/users/identity/options` | 24 h | Reduce cold-start traffic. |
| HTTP | `Cache-Control: private, no-cache` on user-scoped responses | — | Prevents intermediate caching of personal data. |

---

## 9.5 Async Processing

- **Outbox-driven dispatcher**: an in-process worker polls `outbox WHERE status='pending' AND next_run_at <= NOW()` every 250 ms. Push notifications, webhook fan-outs, and deletion-pipeline jobs flow through it.
- **Retry policy:** exponential backoff (1s, 4s, 16s, 1m, 5m, 1h), max 8 attempts, then `status='failed'` + Sentry alert.
- **At-least-once delivery:** consumers idempotent on `outbox.id` (push has natural dedupe on `notification_log.payload_digest`).
- When `modules/chat` is extracted, the outbox graduates to a dedicated Postgres or NATS queue.

---

## 9.6 Read Replicas / Sharding

- **MVP:** none. Single Neon writer.
- **Trigger:** discovery feed > 80 ms p99 OR DB CPU > 70%.
- **Plan:** Neon read replicas with Prisma `previewFeatures = ["multipleConnections"]`. Discovery feed query becomes replica-routed. Writes still go to primary.
- **Sharding:** explicitly deferred until > 200K MAU; the geohash prefix gives us a natural shard key when that arrives.
