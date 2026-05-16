# System Overview

---

## Executive Summary

Spectrum is a mobile-first dating and community app built specifically for LGBTQ+ users. The MVP delivers Google OAuth registration, an inclusive identity model (50+ genders, 20+ orientations, custom pronouns), location-based discovery with privacy-preserving geohashes, mutual-match flow, end-to-end-encrypted chat over the Signal Protocol, a strict safety stack (block, report, panic button, CSAM blocking), a privacy stack (face blur, contact block, incognito), Spectrum+ subscription monetisation via IAP and Stripe, and a Trust & Safety admin console. The backend is a single Node.js/TypeScript Fastify process on Railway, internally divided into ten domain modules, backed by Neon PostgreSQL, Upstash Redis, and AWS S3. The architecture targets < $80/month operating cost up to ~5K MAU and is designed to extract cleanly to microservices once any module sustains > 200 req/sec or MAU exceeds ~50K.

---

## Goals

- Ship every PRD Must-Have feature at production quality on a single-service deployment.
- Hold p99 API latency under 500 ms and p99 WebSocket delivery under 500 ms at MVP load.
- Guarantee end-to-end encryption of message content — server holds ciphertext only.
- Comply with GDPR (EU/UK), DPDP (India), CCPA (California) from day one.
- Maintain monthly infra cost ≤ $80 through 5 K MAU; ≤ $160 through 25 K MAU.
- Keep extract-to-microservices path obvious via `EXTRACT_BOUNDARY` annotations.

---

## Non-Goals

- Social feed, stories, community groups, AI match recommendations beyond filters (Phase 2).
- In-app voice/video calls or live streaming (Phase 2/3).
- HIV/STI status sharing (Phase 2 — requires medical-grade consent framework).
- Travel mode (Phase 3).
- Multi-region active/active deployment at MVP.

---

## Quality Attribute Targets

| Attribute | Target | Measurement |
| --- | --- | --- |
| Availability | 99.9% (≤ 43 min/month downtime) | BetterUptime probes on `/health` |
| Throughput | ≥ 200 req/sec sustained per 0.5 vCPU | k6 load test pre-launch |
| Latency (p99) | Feed < 200 ms, like < 300 ms, WS delivery < 500 ms | Sentry + Railway metrics |
| Scalability | Vertical to 25 K MAU on one Railway instance; horizontal split after | Capacity test at 2× expected Day-1 load |
| Security posture | OWASP Top 10 controls implemented; external pentest pre-launch | SAST/DAST gates in CI; pentest report |
| Maintainability | New developer ships first PR within 3 days | Onboarding metric (manual) |
| Privacy | Zero plaintext PII beyond required auth; zero exact GPS server-side | Schema review + log inspection |

---

## Architectural Style Decision

**Decision:** Monolithic-Modular Node.js service.

**Rationale:**

- Pre-PMF stage — coupling is *cheaper* than premature service boundaries because we will refactor domain boundaries as we learn.
- The 10-service AWS variant costs ~$2,543/month; the monolith costs ~$64/month — a 40× cost factor before user revenue exists.
- Internal function calls (sub-microsecond) are dramatically cheaper than gRPC/HTTP hops, materially reducing tail latency for swipe/match flows.
- Extraction is mechanical: each cross-module boundary is already tagged `EXTRACT_BOUNDARY`, so any module can be promoted to its own Railway service in a single sprint when load justifies it.
- E2EE chat means very little server-side coupling exists anyway — the heaviest lifting is on-device.

See [14-ADRs.md — ADR-001](./14-ADRs.md) for the full decision record.
