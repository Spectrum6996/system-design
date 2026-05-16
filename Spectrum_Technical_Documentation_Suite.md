# Spectrum — Technical Documentation Suite

**Product:** Spectrum — LGBTQ+ Dating & Community App
**Scope:** Phase 1 / MVP
**Architecture:** Monolithic-Modular (Node.js / TypeScript / Fastify on Railway)
**Document Version:** 1.0 (derived from TRD v2.0 / Architecture v3.0)
**Authors:** Principal Software Architect (synthesised from Engineering Lead, Backend Architect, Mobile Lead, Security Engineer)
**Date:** May 2026
**Status:** Engineering-ready

---

## Phase 1 — TRD Analysis

### Extracted Project Profile

| Attribute | Value |
| --- | --- |
| Product purpose | Safety-first, identity-inclusive dating & community app for LGBTQ+ users in India, US, UK |
| MVP goal | Ship the PRD's Must-Have features as a single monolithic Node.js service on Railway, at < $80/month for the first 5K users |
| Primary user roles | End user (free), End user (Spectrum+ premium), Moderator, Trust & Safety Lead, Support Agent, Read-only Analyst, Super Admin |
| Functional pillars | Auth (OTP+JWT), Inclusive identity, Geohash discovery & matching, E2EE chat (Signal Protocol), Safety (block/report/panic/CSAM), Privacy (incognito, contact block, face blur), Billing (IAP+Stripe), Push notifications, Admin/T&S dashboard |
| Non-functional requirements | p99 latency targets per endpoint (Section 15), 99.9% uptime, GDPR + DPDP + CCPA compliance, zero plaintext PII on server beyond what auth needs, mandatory E2EE, server-side premium gating |
| Core technology stack | Node 20 LTS, TypeScript, Fastify 4, Prisma 5, Neon PostgreSQL 15, Upstash Redis, AWS S3, Railway.app, Flutter 3 mobile, Signal Protocol (libsignal-node), Twilio Verify, FCM + APNs, Stripe + IAP |
| Third-party integrations | Twilio Verify (OTP), AWS S3 (media), Stripe + Apple App Store Server API + Google Play Developer API (billing), FCM + APNs (push), Sentry (errors), BetterUptime (uptime), Resend (transactional email), Cloudflare (DNS+TLS), PhotoDNA/NCMEC (CSAM hashes) |
| Data entities (13) | users, identity_profiles, user_photos, user_locations, discovery_filters, swipes, matches, conversations, messages, blocks, reports, subscriptions, devices (+ derived: auth_sessions, consent_log, notification_log, admin_users, moderation_actions) |
| Constraints | Server must never see plaintext chat or exact GPS; client-only Signal private keys; India data resident in ap-south-1 equivalent; under-18 hard blocked at DB CHECK; moderator MUST NOT see chat content; OWASP Top 10 baseline; Railway env vars for all secrets |
| Deployment target | Railway.app (single service), Neon (PostgreSQL), Upstash (Redis), AWS S3 (ap-south-1 / us-east-1). 1 replica baseline, scales horizontally |
| Migration trigger | Any module > 200 req/sec sustained, team > 8 engineers, or 50K+ MAU → extract modules to microservices using `EXTRACT_BOUNDARY` markers |

### Open Questions (carried forward — not blocking)

1. **Voice/video call roadmap.** Voice calling is implied (voice notes are MVP) but real voice calling is Phase 2. Confirm no MVP requirement for in-app calling so we can skip TURN/STUN infra in v1.
2. **Boost economics.** "Profile Boost" exists in premium gates but is not priced or scoped (duration, multiplier window, cool-down). Assumed: 30-minute window, 2× scoring multiplier, 1 free boost per week for premium users.
3. **Photo verification automation.** TRD says "AWS Rekognition (Phase 2) or manual review (Phase 1)". Assumed: manual review backed by an admin queue at MVP; Rekognition deferred.
4. **Face-blur enforcement.** Blur default is opt-out in discovery feed for photo 1 only; behaviour on photos 2–9 unclear. Assumed: photos 2–9 unblurred once user explicitly opts in to show face; otherwise consistent blur.
5. **Refresh token reuse-detection.** "Single-use rotation" stated but reuse-detection / family invalidation policy not specified. Assumed: detect reuse → invalidate entire token family + force re-login (industry standard).
6. **Soft delete window for blocks/conversations.** TRD says blocked conversations are "hidden, not deleted — retained as moderation evidence" but no retention window. Assumed: 180 days retention, then hard purge unless under legal hold.
7. **Multi-region failover.** Cost-model implies single-region per market. Cross-region DR strategy is not defined. Assumed: Neon point-in-time recovery is the DR; RTO 4h / RPO 5min for MVP.
8. **Admin VPN.** Section 12 says "VPN IP allowlist". No specific VPN technology named. Assumed: Tailscale ACL (free for < 100 users) feeding a Cloudflare WAF allowlist.

### Architectural Style Decision (Mandated)

Per stakeholder direction and reaffirmed by the TRD: **Modular Monolith** — a single deployable Fastify process subdivided into ten domain modules (`auth`, `user`, `discovery`, `match`, `chat`, `safety`, `notification`, `billing`, `media`, `admin`) with `EXTRACT_BOUNDARY`-annotated cross-module function calls. Microservice extraction is deferred until concrete triggers (Section 18) are met.

### Beyond-TRD Recommendations Folded In

These additions are inspired by Tinder, Bumble, Grindr, HER, Hinge, and Feeld but kept within MVP scope cost:

- **OpenTelemetry** wired from day 1 (Sentry already ingests), so distributed tracing is free if we extract modules later.
- **Feature flags** via Unleash OSS or LaunchDarkly free tier — gate Boost, premium filters, and risky migrations.
- **Outbox table** for cross-module side effects (push, billing webhooks) so internal calls can be replayed if Notification fails — critical when extracting to microservices.
- **HMAC-SHA256 anti-bot on `/auth/otp/request`** — Cloudflare Turnstile widget on mobile to suppress SMS pumping fraud (a hard-learned lesson from every dating app).
- **Tombstoned soft delete with cryptographic shredding** for GDPR/DPDP: instead of `DELETE`, rotate per-user encryption envelope key in KMS so plaintext is unrecoverable, then schedule physical delete.
- **Shadow-ban primitive** in `users.status` (`shadow_banned`) so T&S can degrade abusers' reach without alerting them.

---

# Phase 2 — Documentation Suite

## 1. System Overview

### Executive Summary
Spectrum is a mobile-first dating and community app built specifically for LGBTQ+ users. The MVP delivers OTP-authenticated registration, an inclusive identity model (50+ genders, 20+ orientations, custom pronouns), location-based discovery with privacy-preserving geohashes, mutual-match flow, end-to-end-encrypted chat over the Signal Protocol, a strict safety stack (block, report, panic button, CSAM blocking), a privacy stack (face blur, contact block, incognito), Spectrum+ subscription monetisation via IAP and Stripe, and a Trust & Safety admin console. The backend is a single Node.js/TypeScript Fastify process on Railway, internally divided into ten domain modules, backed by Neon PostgreSQL, Upstash Redis, and AWS S3. The architecture targets < $80/month operating cost up to ~5K MAU and is designed to extract cleanly to microservices once any module sustains > 200 req/sec or MAU exceeds ~50K.

### Goals
- Ship every PRD Must-Have feature at production quality on a single-service deployment.
- Hold p99 API latency under 500 ms and p99 WebSocket delivery under 500 ms at MVP load.
- Guarantee end-to-end encryption of message content — server holds ciphertext only.
- Comply with GDPR (EU/UK), DPDP (India), CCPA (California) from day one.
- Maintain monthly infra cost ≤ $80 through 5 K MAU; ≤ $160 through 25 K MAU.
- Keep extract-to-microservices path obvious via `EXTRACT_BOUNDARY` annotations.

### Non-Goals
- Social feed, stories, community groups, AI match recommendations beyond filters (Phase 2).
- In-app voice/video calls or live streaming (Phase 2/3).
- HIV/STI status sharing (Phase 2 — requires medical-grade consent framework).
- Travel mode (Phase 3).
- Multi-region active/active deployment at MVP.

### Quality Attribute Targets

| Attribute | Target | Measurement |
| --- | --- | --- |
| Availability | 99.9% (≤ 43 min/month downtime) | BetterUptime probes on `/health` |
| Throughput | ≥ 200 req/sec sustained per 0.5 vCPU | k6 load test pre-launch |
| Latency (p99) | Feed < 200 ms, like < 300 ms, WS delivery < 500 ms | Sentry + Railway metrics |
| Scalability | Vertical to 25 K MAU on one Railway instance; horizontal split after | Capacity test at 2× expected Day-1 load |
| Security posture | OWASP Top 10 controls implemented; external pentest pre-launch | SAST/DAST gates in CI; pentest report |
| Maintainability | New developer ships first PR within 3 days | Onboarding metric (manual) |
| Privacy | Zero plaintext PII beyond required auth; zero exact GPS server-side | Schema review + log inspection |

### Architectural Style Decision

**Decision:** Monolithic-Modular Node.js service.
**Rationale:** (a) Pre-PMF stage — coupling is *cheaper* than premature service boundaries because we will refactor domain boundaries as we learn. (b) The 10-service AWS variant costs ~$2,543/month; the monolith costs ~$64/month — a 40× cost factor before user revenue exists. (c) Internal function calls (sub-microsecond) are dramatically cheaper than gRPC/HTTP hops, materially reducing tail latency for swipe/match flows. (d) Extraction is mechanical: each cross-module boundary is already tagged `EXTRACT_BOUNDARY`, so any module can be promoted to its own Railway service in a single sprint when load justifies it. (e) E2EE chat means very little server-side coupling exists anyway — the heaviest lifting is on-device.

---

## 2. High-Level Architecture

### Component Layer Diagram (text)

```
┌──────────────────────────────────────────────────────────────────────┐
│                         MOBILE CLIENTS                               │
│   Flutter 3.x (iOS 15+ / Android API 29+) — Riverpod, GoRouter      │
│   Local: sqflite (msg cache), flutter_secure_storage (JWT + Signal   │
│   private keys), Signal libraries (libsignal-client-flutter)         │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ HTTPS/TLS 1.3   +    WSS
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                Cloudflare (DNS + TLS termination + WAF)              │
│      → IP allowlist for /admin/* (VPN egress only)                   │
└───────────────────────────┬──────────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         RAILWAY.APP                                  │
│           Single Node 20 / Fastify 4 process — 1 replica             │
│   ┌──────────────────────────────────────────────────────────────┐   │
│   │                    Cross-cutting middleware                  │   │
│   │  JWT verify (RS256) │ Rate limit (Upstash) │ Request log     │   │
│   │  Zod validation     │ Error envelope       │ OpenTelemetry   │   │
│   └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   ┌──────────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐ ┌─────────┐  │
│   │ modules/ │ │ modules/ │ │  modules/  │ │ modules/ │ │modules/ │  │
│   │   auth   │ │   user   │ │ discovery  │ │  match   │ │  chat   │  │
│   │OTP+JWT   │ │profile+  │ │geohash feed│ │like/pass │ │WSS+E2EE │  │
│   │devices   │ │identity  │ │filters     │ │match SM  │ │media key│  │
│   └────┬─────┘ └────┬─────┘ └─────┬──────┘ └────┬─────┘ └────┬────┘  │
│        │            │             │              │            │      │
│   ┌────┴─────┐ ┌────┴─────┐ ┌─────┴──────┐ ┌────┴─────┐ ┌────┴────┐  │
│   │ modules/ │ │ modules/ │ │  modules/  │ │ modules/ │ │modules/ │  │
│   │ safety   │ │  notif   │ │  billing   │ │  media   │ │ admin   │  │
│   │block/rep │ │FCM/APNs  │ │Stripe+IAP  │ │S3+pHash  │ │T&S UI   │  │
│   └──────────┘ └──────────┘ └────────────┘ └──────────┘ └─────────┘  │
│                                                                      │
│   Shared: prisma client │ Upstash client │ S3 client │ logger        │
└──────────┬───────────────────┬──────────────────┬────────────────────┘
           │                   │                  │
           ▼                   ▼                  ▼
   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐
   │ Neon Postgres│    │Upstash Redis │    │  AWS S3     │
   │ 15 + GIN idx │    │rate-limit,   │    │ media,      │
   │ PgBouncer    │    │geohash sets, │    │ encrypted   │
   │              │    │block sets    │    │ ciphertext  │
   └──────────────┘    └──────────────┘    └─────────────┘

   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐
   │ Twilio Verify│    │FCM v1 + APNs │    │Stripe + IAP │
   │   (OTP)      │    │(push)        │    │(billing)    │
   └──────────────┘    └──────────────┘    └─────────────┘

   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐
   │ Sentry       │    │BetterUptime  │    │ NCMEC/      │
   │ (errors)     │    │ (uptime)     │    │ PhotoDNA    │
   └──────────────┘    └──────────────┘    └─────────────┘
```

### Module Responsibilities (summary)

| Module | Responsibility |
| --- | --- |
| `auth` | OTP issuance/verify, JWT mint, refresh rotation, device registry, logout, session lockout |
| `user` | Profile CRUD, identity options, visibility settings, account export, deletion pipeline |
| `discovery` | Geohash feed query, filters, scoring, contact-block enforcement |
| `match` | Swipe state machine, mutual-match creation, daily-like quotas, unmatch, undo (premium) |
| `chat` | WSS gateway, Signal pre-key store, ciphertext message persistence, media-key envelope, typing indicators |
| `safety` | Block graph, report intake, severity routing, moderator action log, panic-mode hooks |
| `notification` | Device-token registry, FCM/APNs dispatch, privacy-safe payloads, outbox replay |
| `billing` | Stripe subscriptions, Apple/Google IAP verification, webhook handling, premium flag mutation |
| `media` | S3 presigned URLs, face-blur queue, CSAM hash check, pHash duplicate prevention |
| `admin` | T&S dashboard, RBAC, audit log, moderator action surface, system metrics |

### Data Flow Narratives

**Sign-up + first feed load (cold start)**
1. Mobile client posts `/auth/otp/request` → middleware: Turnstile + per-phone Upstash rate limit → `auth.requestOtp()` → Twilio Verify.
2. Twilio delivers SMS → user enters code → `/auth/otp/verify` → `auth.verifyOtp()` validates via Twilio → if first time, `user.createUser()` (DB write, `is_verified=false`); JWT pair signed RS256.
3. Client uploads geohash via `/users/location` → `discovery.upsertLocation()` writes `user_locations` + updates Upstash geohash set.
4. Client requests `/discovery/feed?limit=50` → JWT middleware → `discovery.feed(userId)` → reads geohash, fetches 9 adjacent prefixes from Upstash, runs PostgreSQL GIN query with `open_to @> caller_gender`, filters swipes+blocks (loaded into Redis Set at session start), scores, returns 50 cards.

**Like → mutual match → push notification (hot path)**
1. `POST /matches/like/:targetId` → JWT middleware extracts `userId` → `match.like(userId, targetId)`.
2. `match.like` opens a transaction: `INSERT INTO swipes ... liked=true`, checks reverse swipe — if reverse row exists and `liked=true`, inserts `matches` row, inserts `conversations` row.
3. On match success, `// EXTRACT_BOUNDARY: notification ← match` → `notificationModule.dispatch({ userId: otherUser, type: 'match_new', matchId })`. Notification module enqueues to outbox table, returns; Fastify response unblocked.
4. Async worker (same process) reads outbox → posts FCM/APNs → marks delivered.

**Send an E2EE message**
1. Client opens WSS `/chat?token=<jwt>` (Fastify @fastify/websocket).
2. Client computes Signal session (or initiates X3DH on first message), encrypts payload, sends `{type:'message', conversation_id, ciphertext, ...}`.
3. `chat.ws.onMessage` persists ciphertext in `messages` (server cannot decrypt), updates `conversations.last_message_at`, pushes envelope down recipient's open WSS if connected, otherwise dispatches a "you have a new message" push via Notification.
4. Recipient client decrypts locally using Double Ratchet state, displays.

**Report a user (CRITICAL severity)**
1. `POST /safety/report { reportedUserId, category, description }` → `safety.createReport()`.
2. Severity classifier maps category → if CRITICAL, immediate `users.status='suspended'` on target inside the same transaction, write `moderation_actions` row, push to PagerDuty via webhook.
3. Reporter receives 202; CRITICAL row pinned in T&S queue with red badge.

### External Integration Map

```
┌──────────────┐  OTP SMS        ┌────────────┐
│ modules/auth │ ───────────────▶│  Twilio    │
└──────────────┘                  │  Verify    │
                                  └────────────┘

┌──────────────┐  presigned URL  ┌────────────┐
│modules/media │ ───────────────▶│  AWS S3    │
└──────────────┘ ◀───────────────│            │
                                  └────────────┘

┌──────────────┐  outbound push  ┌────────────┐
│ modules/     │ ───────────────▶│ FCM v1 /   │
│ notification │                  │ APNs HTTP2 │
└──────────────┘                  └────────────┘

┌──────────────┐  receipt verify ┌────────────┐
│modules/      │ ◀──────────────▶│ Apple App  │
│ billing      │                  │ Store SS / │
│              │                  │ Play Dev API│
└──────────────┘                  └────────────┘
       │  webhook
       ▼
┌──────────────┐
│   Stripe     │
└──────────────┘

┌──────────────┐  CSAM hash check┌────────────┐
│modules/media │ ───────────────▶│ PhotoDNA / │
└──────────────┘                  │ NCMEC      │
                                  └────────────┘

┌──────────────┐  error events   ┌────────────┐
│  All modules │ ───────────────▶│  Sentry    │
└──────────────┘                  └────────────┘
```

---

## 3. Technology Stack

| Layer | Technology | Version | Justification |
| --- | --- | --- | --- |
| Mobile app | Flutter / Dart | 3.x | One codebase for iOS+Android keeps a 2-engineer mobile team viable; mature Signal Protocol bindings. |
| State mgmt | Riverpod | 2.x | Compile-time safety vs. Provider; testable; widely adopted. |
| Routing | GoRouter | 14.x | Deep links + nested routes required for panic-mode decoy screen + push deep links. |
| Mobile HTTP | Dio | 5.x | Interceptors for JWT refresh, retry, request signing. |
| Mobile secure store | flutter_secure_storage | 9.x | Keychain (iOS) / Android Keystore for JWTs and Signal private keys. |
| Mobile local DB | sqflite | 2.x | Encrypted message cache; large message history without re-fetch. |
| Geo | geolocator | 12.x | Approximate accuracy supported; battery-efficient. |
| Push (mobile) | firebase_messaging | 15.x | One SDK handles APNs + FCM. |
| Crypto (mobile) | libsignal-client (Dart bindings) | latest | Official Signal implementation; battle-tested. |
| Backend lang | Node.js | 20 LTS | Strong async I/O fit for WebSocket + REST; same language across team. |
| Type system | TypeScript | 5.x | Compile-time guarantees critical for cross-module function calls. |
| HTTP + WS framework | Fastify | 4.x | Fastest Node framework benchmark; first-class plugin model; @fastify/websocket. |
| ORM | Prisma | 5.x | Parameterized queries by default; great migration tooling; type-safe. |
| Validation | Zod | 3.x | Inferred TS types from schemas; uniform validation across REST + WS. |
| Primary DB | Neon (PostgreSQL) | 15 | Serverless Postgres with PgBouncer; scale-to-zero saves cost; per-branch databases ideal for previews. |
| Cache + queues | Upstash Redis | — | HTTP API, no TCP connections to manage; serverless free tier covers MVP. |
| Object storage | AWS S3 | — | Presigned URLs let client upload directly; SSE-S3 at rest; region selection for data residency. |
| Discovery search | PostgreSQL GIN + pg_trgm | 15 | Replaces Elasticsearch (~$773/mo) at < 50K profiles. Plan B: Meilisearch self-host. |
| Auth | JWT RS256 + Twilio Verify | — | Asymmetric so all modules verify with public key only; Twilio offloads SMS reliability. |
| E2EE | Signal Protocol (libsignal-node) | latest | Forward + future secrecy; industry standard; non-negotiable per TRD. |
| Push (server) | firebase-admin + node-apn | latest | FCM v1 (Android) + APNs HTTP/2 (iOS). |
| Payments | Stripe + StoreKit 2 + Play Billing 6 | latest | Web (Stripe), iOS, Android — all three required. |
| Email | Resend | — | Free 3K/month; clean DX. |
| Deploy | Railway.app | — | Dockerfile auto-build, WSS native, env-var secrets, ~2-min deploys. |
| IaC | Terraform (S3, IAM, KMS) | 1.x | Only AWS resources are Terraformed; Railway uses railway.toml. |
| CI/CD | GitHub Actions + Railway auto-deploy | — | Free tier covers PR checks + main-branch deploys. |
| Errors | Sentry | — | Free Developer tier; OpenTelemetry-compatible. |
| Uptime | BetterUptime | — | 3-min probes on `/health`. |
| Tracing/metrics | OpenTelemetry → Sentry Performance | — | Future-proofs distributed tracing for microservice split. |
| Feature flags | Unleash OSS or LaunchDarkly free | — | Gate Boost, premium filters, risky migrations. |
| WAF + DNS | Cloudflare | — | DDoS, TLS, IP allowlists for admin. |
| Anti-abuse | Cloudflare Turnstile | — | Suppresses SMS pumping attacks on `/auth/otp/request`. |

---

## 4. Data Architecture

### 4.1 Entity Relationship Overview

| Entity | Key Fields | Cardinality |
| --- | --- | --- |
| users | id (PK), phone (uniq), email (uniq), birth_date, status, is_premium | 1—1 → identity_profiles, 1—* → user_photos, 1—1 → user_locations, 1—* → swipes, 1—* → blocks (as blocker), 1—* → reports |
| identity_profiles | user_id (PK/FK), gender_identity, open_to, relationship_intent | 1—1 with users |
| user_photos | id (PK), user_id (FK), s3_key, ord, is_blurred, phash | *—1 users |
| user_locations | user_id (PK/FK), geohash (precision 7), city, country | 1—1 users |
| discovery_filters | user_id (PK/FK), age_min, age_max, intents[], structures[] | 1—1 users |
| swipes | id (PK), swiper_id (FK), swiped_id (FK), liked, created_at | *—1 users (both sides) |
| matches | id (PK), user_a (FK), user_b (FK), status, matched_at | *—1 users (twice); 1—1 → conversations |
| conversations | id (PK), match_id (FK), last_message_at | 1—1 matches; 1—* → messages |
| messages | id (PK), conversation_id (FK), sender_id (FK), ciphertext, media_key | *—1 conversations |
| blocks | id (PK), blocker_id (FK), blocked_id (FK), created_at | *—* directed graph |
| reports | id (PK), reporter_id (FK), reported_id (FK), severity, category, status | *—* directed |
| subscriptions | id (PK), user_id (FK), channel, plan, expires_at, store_receipt | *—1 users |
| devices | id (PK), user_id (FK), push_token, platform, refresh_token_hash, last_seen | *—1 users |
| auth_sessions | id (PK), user_id (FK), device_id (FK), issued_at, revoked_at | *—1 users |
| consent_log | id (PK), user_id (FK), policy_version, consent_type, granted_at | *—1 users |
| moderation_actions | id (PK), admin_id (FK), target_user_id (FK), action, rationale, taken_at | *—1 users |
| notification_log | id (PK), user_id (FK), kind, payload_digest, sent_at, status | *—1 users |
| outbox | id (PK), aggregate_type, aggregate_id, kind, payload, status, attempts | append-only queue |
| admin_users | id (PK), email, role, mfa_secret_hash, last_login_at | independent |

### 4.2 Primary Schema (DDL — abridged; production schema lives in `prisma/schema.prisma`)

```sql
-- ============ USERS ============
CREATE TABLE users (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone           VARCHAR(20) UNIQUE,
  email           VARCHAR(255) UNIQUE,
  display_name    VARCHAR(50) NOT NULL,
  birth_date      DATE NOT NULL,
  status          VARCHAR(20) DEFAULT 'active',  -- active|suspended|shadow_banned|deleted
  is_verified     BOOLEAN DEFAULT false,
  is_premium      BOOLEAN DEFAULT false,
  incognito_since TIMESTAMPTZ,
  locale          VARCHAR(10) DEFAULT 'en',
  last_active     TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW(),
  CONSTRAINT age_check CHECK (birth_date <= NOW() - INTERVAL '18 years')
);

-- ============ IDENTITY ============
CREATE TABLE identity_profiles (
  user_id                 UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  gender_identity         VARCHAR(100) NOT NULL,
  gender_custom           VARCHAR(100),
  sexual_orientation      VARCHAR(100) NOT NULL,
  orientation_custom      VARCHAR(100),
  pronouns                VARCHAR(50),
  pronouns_custom         VARCHAR(50),
  relationship_intent     VARCHAR(50) NOT NULL,
  relationship_structure  VARCHAR(50),
  open_to                 JSONB NOT NULL DEFAULT '[]',
  bio                     TEXT,
  show_orientation        BOOLEAN DEFAULT true,
  show_structure          BOOLEAN DEFAULT false,
  profile_completion_score SMALLINT DEFAULT 0,
  updated_at              TIMESTAMPTZ DEFAULT NOW()
);

-- ============ LOCATION (never exact GPS) ============
CREATE TABLE user_locations (
  user_id    UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  geohash    VARCHAR(12) NOT NULL,
  city       VARCHAR(100),
  country    VARCHAR(100),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============ MATCHING ============
CREATE TABLE swipes (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  swiper_id  UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  swiped_id  UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  liked      BOOLEAN NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(swiper_id, swiped_id)
);
CREATE TABLE matches (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_a     UUID REFERENCES users(id),
  user_b     UUID REFERENCES users(id),
  status     VARCHAR(20) DEFAULT 'active',
  matched_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_a, user_b),
  CONSTRAINT ordered CHECK (user_a < user_b)
);

-- ============ CHAT (ciphertext only) ============
CREATE TABLE conversations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  match_id        UUID UNIQUE REFERENCES matches(id) ON DELETE CASCADE,
  last_message_at TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE TABLE messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
  sender_id       UUID REFERENCES users(id),
  msg_type        VARCHAR(20),                  -- text|image|voice|system
  ciphertext      TEXT NOT NULL,
  media_key       TEXT,
  sent_at         TIMESTAMPTZ DEFAULT NOW(),
  delivered_at    TIMESTAMPTZ,
  read_at         TIMESTAMPTZ,
  deleted_at      TIMESTAMPTZ
);

-- ============ SAFETY ============
CREATE TABLE blocks (
  blocker_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  blocked_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (blocker_id, blocked_id)
);
CREATE TABLE reports (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  reporter_id  UUID REFERENCES users(id) ON DELETE SET NULL,
  reported_id  UUID REFERENCES users(id) ON DELETE CASCADE,
  category     VARCHAR(40) NOT NULL,
  severity     VARCHAR(10) NOT NULL,            -- CRITICAL|HIGH|STANDARD
  description  TEXT,
  status       VARCHAR(20) DEFAULT 'open',
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- ============ BILLING ============
CREATE TABLE subscriptions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,
  channel       VARCHAR(20) NOT NULL,           -- apple|google|stripe
  plan          VARCHAR(20) NOT NULL,           -- monthly|quarterly|annual
  status        VARCHAR(20) NOT NULL,           -- active|grace|expired|cancelled
  store_receipt TEXT,
  started_at    TIMESTAMPTZ,
  expires_at    TIMESTAMPTZ,
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

-- ============ DEVICES / AUTH ============
CREATE TABLE devices (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id             UUID REFERENCES users(id) ON DELETE CASCADE,
  push_token          TEXT,
  platform            VARCHAR(10),              -- ios|android
  app_version         VARCHAR(20),
  refresh_token_hash  TEXT NOT NULL,            -- sha256(refresh)
  refresh_family_id   UUID NOT NULL,
  last_seen           TIMESTAMPTZ
);

-- ============ OUTBOX (new — for guaranteed cross-module side effects) ============
CREATE TABLE outbox (
  id            BIGSERIAL PRIMARY KEY,
  aggregate     VARCHAR(40) NOT NULL,
  aggregate_id  UUID NOT NULL,
  kind          VARCHAR(60) NOT NULL,
  payload       JSONB NOT NULL,
  status        VARCHAR(20) DEFAULT 'pending',  -- pending|sent|failed
  attempts      SMALLINT DEFAULT 0,
  next_run_at   TIMESTAMPTZ DEFAULT NOW(),
  created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

### 4.3 Indexing Strategy

| Index | Purpose | Reasoning |
| --- | --- | --- |
| `idx_users_active (status, incognito_since, last_active) WHERE status='active'` | Discovery candidate set | Partial index keeps it small; the most selective predicate is `status='active'`. |
| `idx_ul_geohash_prefix (geohash text_pattern_ops)` | Prefix range scans across 9 cells | `text_pattern_ops` lets `LEFT(geohash,5) = ANY(...)` use a B-tree range. |
| `idx_ip_open_to_gin USING GIN (open_to jsonb_path_ops)` | `open_to @> $callerGender` | GIN with `jsonb_path_ops` is 2-3× faster than the default GIN for containment. |
| `idx_ip_intent (relationship_intent)` | Intent filter | Low cardinality but combined with geohash narrows candidate set efficiently. |
| `idx_swipes_swiper (swiper_id)` | Exclude already-swiped from feed | High-write table; this index is essential to the discovery `NOT IN` join. |
| `idx_swipes_pair (swiped_id, swiper_id)` | Mutual-like lookup in `match.like` | Composite supports the symmetric check `WHERE swiper=$other AND swiped=$me`. |
| `idx_blocks_blocker (blocker_id)`, `idx_blocks_blocked (blocked_id)` | Bidirectional exclusion | Discovery loads `blocks` into Redis Set; queries also exist for fallback. |
| `idx_messages_conv_sent (conversation_id, sent_at DESC)` | Paginated message history | Always sorted DESC; covers the most common query. |
| `idx_reports_severity_status (severity, status, created_at DESC)` | T&S queue | Pinned CRITICAL at top relies on this ordering. |
| `idx_outbox_status_next_run (status, next_run_at) WHERE status='pending'` | Async worker dequeue | Partial index, small. |
| `idx_devices_user_lastseen (user_id, last_seen DESC)` | Active device lookup for push | Most-recent device first. |

### 4.4 Data Access Patterns

- **Read-heavy:** `/discovery/feed` (60+ rps/user), `/chat/conversations` (every app open), `/users/:id` (profile inspection). All cacheable.
- **Write-heavy:** `swipes` (every card view that triggers like/pass), `messages` (chat), `outbox`.
- **Cache layers:**
  - Upstash Redis **per-user block set** (`block:{userId}` Set) — loaded on session start, invalidated on `/safety/block` mutation.
  - Upstash **geohash neighbour cache** (`geo:{prefix5}` Sorted Set) of active userIds — refreshed every 5 minutes by a Fastify scheduled job; expired naturally.
  - Upstash **daily quotas** (`q:like:{userId}:{yyyymmdd}` counter with EXPIRE midnight UTC).
  - Sentry/Fastify HTTP cache headers (private, no-cache) on user-specific responses; CDN on static assets only.
- **Pagination:** Cursor-based for chronological lists (`messages`, `matches`, `reports`); offset-based for the discovery feed (acceptable because feeds are ephemeral and shuffled). Cursor format: opaque base64 of `(sortField, id)`.
- **Hot-path query latency budget:** feed query < 80 ms p50 / < 200 ms p99 at 50K profiles, verified on Neon Launch plan.

### 4.5 Migration & Seeding

- **Migrations:** Prisma Migrate (`prisma migrate dev` locally, `prisma migrate deploy` in CI). Every migration must be backward-compatible across one deploy (online migration discipline: add column → backfill → switch reads → drop later).
- **Schema branching:** Neon's branch-per-PR feature gives each PR its own database; CI runs `prisma migrate deploy` then integration tests.
- **Seeding:** `prisma/seed.ts` provisions: 3 admin roles, 50 synthetic users across genders/orientations for E2E tests, an outbox-clean state, and identity-options.json content.
- **Data residency:** Two Neon projects (`spectrum-prod-eu`, `spectrum-prod-ap`) for EU and India regions; users routed by registration country. (US users default to `us-east-1` Neon project.)

---

## 5. API Design

### 5.1 Style
- **Primary:** REST/JSON over HTTPS for all CRUD and stateless RPC-like actions (matches Flutter Dio ergonomics, simple to cache, no schema lock-in for mobile evolution).
- **Real-time:** WebSocket Secure (WSS) for chat — exactly one persistent socket per app session via `@fastify/websocket` on the same Fastify process.
- **No GraphQL/tRPC at MVP** — endpoint surface is small (42 endpoints), and GraphQL's introspection complicates rate limiting and abuse modelling for a safety-critical app.

### 5.2 Base URL & Versioning

| Concern | Convention |
| --- | --- |
| REST base | `https://api.spectrum.app/v1/` |
| WSS base | `wss://api.spectrum.app/chat` |
| Versioning | URL-prefix major version (`/v1/`). Breaking changes require a `/v2/` namespace; both run in the monolith via Fastify route plugins. Deprecation requires 6-month overlap and `Deprecation`/`Sunset` headers (RFC 8594). |
| Content type | `application/json; charset=utf-8` only (no XML, no form-encoded except IAP webhooks where required). |
| Compression | `Accept-Encoding: br, gzip` honoured. |

### 5.3 Authentication & Authorisation
- **Auth:** Bearer JWT (RS256), 15 min TTL, in `Authorization: Bearer <token>`. Asymmetric so all modules verify with public key only.
- **Refresh:** Opaque refresh token, 30-day TTL, single-use, family-tracked (reuse → invalidate entire family + force re-login).
- **Authorization:** RBAC via JWT `role` claim (`free | premium | admin:*`). Premium gates enforced server-side in module-layer code (Section 10 of TRD).
- **Admin:** Separate JWT issuance flow with hardware MFA (TOTP/FIDO2) + VPN-network-bound IP allowlist.

### 5.4 Endpoint Reference (Phase 1 — 42 endpoints)

| Method | Path | Auth | Request Body | Response (200/2xx) | Notes |
| --- | --- | --- | --- | --- | --- |
| POST | `/v1/auth/otp/request` | Public + Turnstile | `{ phone }` | `{ challenge_id, expires_at }` | 5/hour/phone. SMS via Twilio Verify. |
| POST | `/v1/auth/otp/verify` | Public | `{ phone, code }` | `{ access, refresh, user }` | 10 attempts → 60-min lockout. |
| POST | `/v1/auth/refresh` | Refresh | `{ refresh }` | `{ access, refresh }` | Single-use rotation; family invalidation on reuse. |
| POST | `/v1/auth/logout` | JWT | `{}` | `204` | Removes refresh + device push token. |
| GET | `/v1/users/:id` | JWT | — | `User` | 404 if blocked or non-existent. |
| PUT | `/v1/users/profile` | JWT | `Profile` | `Profile` | Validates display_name, bio length. |
| PUT | `/v1/users/identity` | JWT | `Identity` | `Identity` | Custom values length-checked. |
| GET | `/v1/users/identity/options` | Public | — | `{ genders, orientations, pronouns, intents, structures }` | Driven by `identity-options.json` so additions don't need a deploy. |
| PUT | `/v1/users/settings/visibility` | JWT | `Visibility` | `Visibility` | `show_orientation`, `show_structure`, incognito (premium). |
| DELETE | `/v1/account` | JWT | `{}` | `202` | Starts GDPR/DPDP delete pipeline; 30-day soft delete. |
| GET | `/v1/account/export` | JWT | — | `202 + Location header` | Async; SSE or signed S3 download URL emailed when ready. |
| POST | `/v1/users/photos` | JWT | multipart or presign | `Photo` | Either direct multipart (small) or `presign=true` returns S3 URL. |
| DELETE | `/v1/users/photos/:id` | JWT | — | `204` | |
| POST | `/v1/users/location` | JWT | `{ geohash, city?, country? }` | `204` | Server rejects if `geohash.length != 7`. |
| POST | `/v1/chat/media/presign` | JWT | `{ file_size, mime_type }` | `{ upload_url, s3_key }` | URL valid 5 min; `Content-Length` enforced. |
| GET | `/v1/discovery/feed` | JWT | query: `limit`, `cursor?` | `{ cards: Card[], next_cursor }` | 60/h free, 120/h premium. |
| PUT | `/v1/discovery/filters` | JWT | `Filters` | `Filters` | Premium fields ignored if `is_premium=false`. |
| GET | `/v1/discovery/filters` | JWT | — | `Filters` | |
| POST | `/v1/matches/like/:id` | JWT | `{}` | `{ matched: boolean, match_id?: uuid }` | 10/day free. |
| POST | `/v1/matches/pass/:id` | JWT | `{}` | `204` | |
| DELETE | `/v1/matches/undo` | JWT + Premium | `{}` | `204` | `403` if non-premium. |
| GET | `/v1/matches` | JWT | `cursor?` | `{ matches: Match[], next_cursor }` | |
| POST | `/v1/matches/unmatch/:id` | JWT | `{}` | `204` | Hides conversation both sides. |
| GET | `/v1/chat/conversations` | JWT | `cursor?` | `{ conversations: Conv[], next_cursor }` | |
| GET | `/v1/chat/conversations/:id/messages` | JWT | `cursor?, limit` | `{ messages: Msg[], next_cursor }` | Returns ciphertext only. |
| DELETE | `/v1/chat/messages/:id` | JWT | — | `204` | Soft-delete with `deleted_at`. |
| POST | `/v1/chat/keys` | JWT | `KeyBundle` | `204` | Uploads identity+signed+one-time pre-keys. |
| GET | `/v1/chat/keys/:userId` | JWT | — | `KeyBundle` | One-time pre-key consumed per fetch. |
| POST | `/v1/safety/block/:userId` | JWT | `{}` | `204` | Refreshes Redis `block:` set within 500 ms. |
| DELETE | `/v1/safety/block/:userId` | JWT | — | `204` | |
| GET | `/v1/safety/blocks` | JWT | — | `{ blocked_users: id[] }` | |
| POST | `/v1/safety/report` | JWT | `{ reported_id, category, description }` | `202` | CRITICAL paths PagerDuty. |
| POST | `/v1/privacy/contact-block` | JWT | `{ hashed_phones: string[] }` | `204` | Hashes are HMAC-SHA256 client-side. |
| GET | `/v1/billing/subscription` | JWT | — | `Subscription` | |
| POST | `/v1/billing/iap/apple/verify` | JWT | `{ receipt }` | `Subscription` | Calls Apple SS API server-side. |
| POST | `/v1/billing/iap/google/verify` | JWT | `{ purchase_token, product_id }` | `Subscription` | Calls Play Dev API. |
| POST | `/v1/billing/cancel` | JWT | `{}` | `Subscription` | Cancel-at-period-end. |
| POST | `/v1/billing/webhooks/apple` | Apple sig | (signed payload) | `204` | App Store Server Notifications v2. |
| POST | `/v1/billing/webhooks/google` | Google sig | (signed payload) | `204` | RTDN. |
| POST | `/v1/billing/webhooks/stripe` | Stripe sig | (signed payload) | `204` | `Stripe-Signature` check. |
| POST | `/v1/devices/register` | JWT | `{ push_token, platform, app_version }` | `204` | Called every cold start. |
| POST | `/v1/devices/unregister` | JWT | `{}` | `204` | On logout. |
| GET | `/v1/health` | Public | — | `200 OK` | Health probe (no payload secrets). |

### 5.5 Error Envelope (universal)

```jsonc
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",     // stable machine code
    "message": "birth_date must be ≥ 18 years ago",
    "field": "birth_date",           // optional
    "details": { "min_age": 18 }     // optional
  },
  "meta": { "request_id": "uuid", "timestamp": "2026-05-13T10:00:00Z" }
}
```

**Error codes (canonical set):** `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `RATE_LIMITED`, `PAYMENT_REQUIRED`, `UNDERAGE`, `BLOCKED`, `INTERNAL_ERROR`, `UPSTREAM_UNAVAILABLE`.

### 5.6 Rate Limits

| Endpoint Group | Free | Premium | Window |
| --- | --- | --- | --- |
| `POST /auth/otp/request` | 5 | 5 | per phone, per hour |
| `POST /matches/like` | 10 | unlimited | per user, per day |
| `GET /discovery/feed` | 60 | 120 | per user, per hour |
| `POST /chat/*` | 200 | 500 | per user, per hour |
| `POST /safety/report` | 10 | 10 | per user, per day |
| All other authed | 100 | 200 | per user, per minute |

Backed by an Upstash Lua-script token bucket. Responses include `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, `Retry-After` (RFC 9239 draft).

### 5.7 WebSocket Events

| Direction | Event | Payload | Notes |
| --- | --- | --- | --- |
| C→S | `message` | `{conversation_id, client_message_id, msg_type, ciphertext, media_key?, sent_at}` | Server stores ciphertext. |
| C→S | `typing` | `{conversation_id}` | TTL 5 s, throttled to 1/sec. |
| C→S | `read` | `{message_id}` | Read receipt; server only forwards if recipient is premium. |
| C→S | `ping` | `{}` | Keepalive every 30 s. |
| S→C | `message.new` | full envelope | |
| S→C | `message.delivered` | `{message_id, delivered_at}` | |
| S→C | `message.read` | `{message_id, read_at}` | Premium only. |
| S→C | `match.new` | `{match_id, user_preview}` | |
| S→C | `conversation.typing` | `{conversation_id, user_id}` | |
| S→C | `user.unmatched` | `{match_id}` | |
| S→C | `user.blocked` | `{}` | Silent — UI hides conversation. |
| S→C | `pong` | `{}` | |
| S→C | `error` | `{code, message}` | Authoritative server errors. |

---

## 6. System Components (Module Design)

### 6.1 modules/auth
- **Purpose:** OTP-based identity proof, JWT lifecycle, device-bound refresh tokens.
- **Sub-components:** `routes.ts`, `service.ts`, `repository.ts`, `otp.ts` (Twilio wrapper), `jwt.ts` (RS256 with kid header), `device.ts`.
- **Interfaces (internal):** `auth.verifyJWT(token)`, `auth.requireRole(roles[])`, `auth.invalidateFamily(userId)`.
- **Dependencies:** Twilio Verify, Upstash (rate limits + lockouts), Prisma.
- **State owned:** `users (auth fields)`, `devices`, `auth_sessions`.
- **Edge cases:** Phone re-registration (transfer of accounts disabled — duplicate phone returns 409); SIM swap (force re-OTP after JWT age > 24h on suspicious device fingerprint); refresh-token reuse (invalidate entire family, force re-login).

### 6.2 modules/user
- **Purpose:** Profile, identity, account lifecycle, consent log, deletion pipeline.
- **Sub-components:** `routes.ts`, `service.ts`, `identityOptions.ts`, `consent.ts`, `deletion.ts`.
- **Interfaces:** `user.get(id)`, `user.updateProfile(id, patch)`, `user.startDeletion(id)`.
- **Dependencies:** Prisma, S3 (export bucket), Notification (for deletion confirmation).
- **State owned:** `users`, `identity_profiles`, `consent_log`.
- **Edge cases:** Underage report → immediate suspension + audit; identity custom-text length overflow → 422; account deletion in flight → all reads return 404 to other users (caller sees grace screen).

### 6.3 modules/discovery
- **Purpose:** Build personalised candidate feed using geohash + identity filters.
- **Sub-components:** `routes.ts`, `feed.ts` (query builder), `geo.ts` (geohash neighbour math), `score.ts`.
- **Interfaces:** `discovery.feed(userId, filters)`, `discovery.refreshBlockSet(userId)`.
- **Dependencies:** Prisma (GIN query), Upstash (block sets + geohash cache).
- **State owned:** `user_locations`, `discovery_filters`.
- **Edge cases:** New user with no geohash → returns city-fallback feed; user travelling rapidly → geohash updated only on app foreground; contact-block matched users excluded silently (never appear); incognito mode → user removed from any other user's feed within 5 min (verified across all 9 cells).

### 6.4 modules/match
- **Purpose:** Like/pass/unmatch state machine; mutual match creation; daily quotas.
- **Sub-components:** `routes.ts`, `service.ts`, `quota.ts`.
- **Interfaces:** `match.like(swiper, target)`, `match.unmatch(matchId, userId)`.
- **Dependencies:** Prisma (transactional), Notification (EXTRACT_BOUNDARY), Upstash (quota counters).
- **State owned:** `swipes`, `matches`.
- **Edge cases:** Self-like (422), like blocked user (404), simultaneous reciprocal like (DB unique constraint resolves; idempotent on retry), undo on non-premium (403), undo when more than the last swipe is involved (only the most recent pass is undoable).

### 6.5 modules/chat
- **Purpose:** WSS gateway + ciphertext storage + Signal pre-key store + media keys.
- **Sub-components:** `ws.ts` (Fastify WS plugin), `routes.ts` (REST for history + keys), `service.ts`, `keys.ts`.
- **Interfaces:** `chat.deliver(envelope)`, `chat.getKeys(userId)`, `chat.consumeOTK(userId)`.
- **Dependencies:** Prisma, Upstash (presence — `presence:{userId}` with TTL), S3 (encrypted media), Notification (offline push).
- **State owned:** `conversations`, `messages`, pre-key bundles (in `chat_keys` table — sub-table of users).
- **Edge cases:** Recipient offline (push fallback with privacy-safe body), recipient blocked sender (silent drop, returns 200 to sender), invalid ciphertext (server doesn't validate semantics; size cap enforced 64KB), one-time pre-key exhaustion (client warned at < 10 remaining via `chat.replenish_otks` event).

### 6.6 modules/safety
- **Purpose:** Bidirectional block graph; report intake & severity classification; panic mode hooks.
- **Sub-components:** `routes.ts`, `service.ts`, `reportClassifier.ts`, `panic.ts`.
- **Interfaces:** `safety.isBlocked(a,b)`, `safety.createReport(...)`, `safety.escalateCritical(reportId)`.
- **Dependencies:** Prisma, Upstash (block sets), PagerDuty (webhook), Notification.
- **State owned:** `blocks`, `reports`, `moderation_actions` (write-only by admins).
- **Edge cases:** Block during open WSS (server force-closes the conversation socket frame to both sides); reporter blocks reported (block proceeds normally — report still queued for moderator); duplicate report (rate-limited at 10/day; duplicates merged in queue UI).

### 6.7 modules/notification
- **Purpose:** Outbox-driven dispatch of FCM/APNs notifications + in-app store.
- **Sub-components:** `service.ts` (internal API), `dispatcher.ts` (worker tick), `templates.ts`, `outbox.ts`.
- **Interfaces:** `notification.dispatch(userId, kind, payload)` — enqueues to outbox.
- **Dependencies:** Prisma (outbox + devices), FCM Admin SDK, node-apn.
- **State owned:** `outbox`, `devices`, `notification_log`.
- **Edge cases:** Panic mode active → suppress dispatch (read `users.incognito_since`? No — panic flag is in-app only; server suppression key is `users.panic_active` cached in Redis); stale token → APNs feedback parsed daily; user has > 3 devices → keep latest 3, drop oldest.

### 6.8 modules/billing
- **Purpose:** Subscription state-of-truth, IAP receipt validation, Stripe.
- **Sub-components:** `routes.ts`, `apple.ts`, `google.ts`, `stripe.ts`, `webhooks.ts`, `gates.ts` (used by other modules).
- **Interfaces:** `billing.isPremium(userId)`, `billing.activate(...)`, `billing.cancel(...)`.
- **Dependencies:** Apple App Store Server API, Google Play Developer API, Stripe, Prisma.
- **State owned:** `subscriptions`.
- **Edge cases:** Refund / chargeback → status→`revoked`, `is_premium=false`, premium features disabled mid-session via JWT short TTL; cross-platform downgrade (user buys on Android then logs in on iOS — single subscription per user enforced); grace period (Apple 16-day, Google 3-day) honoured.

### 6.9 modules/media
- **Purpose:** Photo upload pipeline + face-blur stub + CSAM hash check + pHash dedupe.
- **Sub-components:** `routes.ts`, `presign.ts`, `faceBlur.ts` (Sharp), `phash.ts`, `csam.ts` (PhotoDNA).
- **Interfaces:** `media.attachPhoto(userId, s3Key)`, `media.markBlurred(photoId)`.
- **Dependencies:** S3, Sharp, PhotoDNA/NCMEC, Prisma.
- **State owned:** `user_photos`.
- **Edge cases:** CSAM hash match → upload rejected, user suspended, NCMEC report triggered, action logged immutably; pHash duplicate of previously flagged image → reject; corrupt file → 422; over-size (> 10 MB) → 413.

### 6.10 modules/admin
- **Purpose:** T&S dashboard back-end, RBAC, immutable audit log, user-management actions.
- **Sub-components:** `routes.ts`, `rbac.ts`, `audit.ts`, `dashboard.ts`.
- **Interfaces:** Internal only — not exposed to mobile clients.
- **Dependencies:** All modules (read-only) + `moderation_actions` writes; VPN allowlist middleware; FIDO2/TOTP MFA.
- **State owned:** `admin_users`, `moderation_actions`.
- **Edge cases:** Moderator attempts to read chat content via admin API → API returns 403 unconditionally (system-enforced — chat tables have no admin-readable plaintext column); admin account compromised → all sessions revoked via `auth.invalidateFamily` + audit alert.

---

## 7. Security Design

### 7.1 Threat Model Summary

| Trust boundary | Attack surface | Top threats |
| --- | --- | --- |
| Mobile client ↔ Internet | TLS, JWT, refresh token | Token theft (root/jailbreak), MITM, replay |
| Internet ↔ Cloudflare | TLS termination, WAF | DDoS, scraping, OWASP Top 10 |
| Cloudflare ↔ Railway | TLS, IP allowlist (admin) | Origin bypass, unauthorised admin access |
| Railway ↔ Neon/Upstash/S3 | Connection strings in env vars | Credential theft, lateral movement |
| User ↔ User | Block, report, chat, profile views | Harassment, doxxing, outing, CSAM, catfishing, contact-stalking |
| Admin ↔ System | VPN + MFA + RBAC | Insider abuse, moderator over-reach into chat |

### 7.2 Authentication Flow (step-by-step)

```
1. App → /v1/auth/otp/request { phone }
   ├─ Cloudflare Turnstile token verified
   ├─ Upstash rate limit: 5/h/phone (else 429)
   └─ Twilio Verify sends SMS

2. User enters code → App → /v1/auth/otp/verify { phone, code }
   ├─ Upstash attempt counter: ≤10 (else 429 + 60-min lockout)
   ├─ Twilio Verify validates server-side
   ├─ If new user: insert users + identity (status=registering) + consent_log
   └─ Mint access JWT (RS256, 15min) + refresh (opaque, 30d) bound to a new family_id

3. App stores access in memory + refresh in flutter_secure_storage (Keychain / Keystore)

4. Subsequent API calls → Authorization: Bearer <access>
   ├─ JWT middleware verifies RS256 signature against rotating public key (kid header)
   ├─ Validates exp, nbf, iat, sub
   └─ Attaches { userId, role } to request.context

5. Access expires → App → /v1/auth/refresh { refresh }
   ├─ Look up devices.refresh_token_hash (sha256)
   ├─ If found: rotate (insert new hash, mark old as used)
   ├─ If old token reused: invalidate entire family_id (security incident)
   └─ Return new pair

6. Logout → /v1/auth/logout → device row + family invalidated; push token unregistered
```

### 7.3 Authorisation Matrix

| Role | Endpoints | Notes |
| --- | --- | --- |
| **anonymous** | `/auth/otp/*`, `/users/identity/options`, `/health`, billing webhooks | Public surfaces; rate-limited heavily. |
| **user (free)** | All `/v1/*` except `/admin/*`, `/matches/undo`, premium gates | Limits enforced (likes/day, filters). |
| **user (premium)** | Same + premium gated endpoints | Server checks `users.is_premium`. |
| **moderator** | `/admin/reports/*`, `/admin/users/search`, `/admin/users/:id`, `/admin/actions/*` | NEVER message content endpoints. |
| **trust_safety_lead** | All moderator perms + policy config + SLA dashboards | |
| **support_agent** | `/admin/users/:id` (no photos, no full PII) + `/admin/users/:id/reset-otp` | |
| **readonly_analyst** | `/admin/metrics/*` (anonymised) | Read-only. |
| **super_admin** | All admin endpoints + `/admin/team/*` | Hardware FIDO2 required. |

### 7.4 Data Protection
- **In transit:** TLS 1.3 minimum (Cloudflare enforces). HSTS preload list submission. mTLS between Railway and Stripe/Apple/Google not required (signature verification suffices).
- **At rest:** Neon Postgres AES-256 (managed). Upstash AES-256. S3 SSE-S3 (or SSE-KMS for India region if regulator audit demands). E2EE messages — only ciphertext stored.
- **Secrets:** Railway encrypted env vars. **Zero secrets in code or Git.** Pre-commit hook + `git-secrets` scan. KMS-backed envelope keys for cryptographic shredding on GDPR delete.
- **PII handling:**
  - Phone numbers stored as raw only for OTP path; contact-block phones are **HMAC-SHA256 client-side** before upload (server holds hashes only).
  - Birth date stored as `DATE` (not exact timestamp) — age is computed; year-only display option in UI.
  - Exact GPS coordinates **never** leave device — only precision-7 geohash.

### 7.5 Input Validation & Injection Prevention
- All routes use **Zod schemas** at handler entry. Reject unknown keys (`strict()` parse).
- **Prisma ORM** — all DB access parameterised; no raw SQL strings allowed (lint rule).
- **Length caps** at schema layer: bio 500, display_name 50, custom identity 100, ciphertext 64 KB.
- **JSON depth limits**: Fastify `bodyLimit: 1mb`; custom schema validation rejects nesting > 8.
- **SVG/HTML uploads:** rejected — only JPEG/PNG/HEIC photos.
- **URL fetching:** the only outbound URLs the server fetches are S3 presigned URLs it minted itself. No user-supplied URL is fetched (SSRF prevention).

### 7.6 OWASP Top 10 (2021) Mitigation Checklist

| Risk | Control |
| --- | --- |
| A01 Broken Access Control | JWT RS256 verified every request; userId from token only (never body/header); premium gates server-side only; admin VPN+MFA+RBAC; chat-content read denied to admin by API design. |
| A02 Cryptographic Failures | TLS 1.3, AES-256 at rest, Signal Protocol E2EE for chat, HMAC-SHA256 for contact hashes, RS256 JWTs. |
| A03 Injection | Prisma parameterised; Zod validation; output encoding via Fastify JSON serializer; no eval, no shell exec. |
| A04 Insecure Design | Threat model + STRIDE pass during architecture review; privacy-by-default; under-18 hard CHECK; geohash > exact coords. |
| A05 Security Misconfiguration | Terraform IaC for AWS; Railway env-vars; CSP, HSTS, X-Content-Type-Options, Referrer-Policy headers set globally; no defaults left. |
| A06 Vulnerable & Outdated Components | Renovate bot bumps deps weekly; Snyk SCA blocks high/critical CVEs in CI; SBOM generated by syft. |
| A07 Auth Failures | 15-min JWT, single-use refresh, family invalidation on reuse, lockout after 10 OTP fails, Turnstile on OTP request, optional biometric step-up on app open. |
| A08 Software & Data Integrity | Signed Docker images (cosign), GitHub branch protection, mandatory PR review, deploy approval gate. |
| A09 Logging & Monitoring Failures | Sentry + structured Pino logs + immutable audit trail in `moderation_actions` + PagerDuty escalations. |
| A10 SSRF | Server only fetches its own presigned S3 URLs; outbound URL allowlist enforced by `undici` interceptor. |

### 7.7 Compliance Plan

| Regime | Obligations | Implementation |
| --- | --- | --- |
| **GDPR** (EU/UK) | Right to access, delete, rectify, portability; DPO; lawful basis; DPA | `/account/export` returns JSON within 72h; `/account` DELETE with 30-day soft → hard purge across all modules + S3 + Neon WAL purge; consent_log with policy version; DPO appointed; DPA template under legal review. |
| **DPDP** (India) | Notice + granular consent; Grievance Officer; data localisation | Onboarding consent screen with per-category opt-in; Grievance Officer email in app settings; India users routed to `spectrum-prod-ap` Neon region. |
| **CCPA** (California) | Right to know, delete, opt-out of sale | We don't sell data — "Do Not Sell" link surfaced anyway; deletion via the unified pipeline. |
| **Apple/Google Store** | LGBTQ+ content 17+, content declaration | iOS App Tracking Transparency (we don't track), 17+ rating, content declaration in App Store Connect. |
| **NCMEC** | CSAM reporting | CyberTipline account active; PhotoDNA hash check on every photo upload before persistence. |

---

## 8. Infrastructure & Deployment

### 8.1 Target Environment
- **Primary:** Railway.app (US-East). Single service. WebSocket-native.
- **DB:** Neon Serverless PostgreSQL 15 — `spectrum-prod-us` (default), `spectrum-prod-eu` (EU/UK), `spectrum-prod-ap` (India / DPDP residency).
- **Cache:** Upstash Redis — region-pinned per Neon region.
- **Storage:** AWS S3 — `us-east-1`, `eu-west-1`, `ap-south-1` buckets; same KMS key policy.
- **Edge:** Cloudflare (DNS, TLS, WAF, Turnstile).

### 8.2 Infrastructure Diagram (text)

```
            ┌─────────────────────────────────────────┐
            │            Cloudflare (global)          │
            │   DNS · TLS 1.3 · WAF · Turnstile       │
            └─────────────────────┬───────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
      ┌────────────┐      ┌────────────┐      ┌────────────┐
      │  Railway   │      │  Railway   │      │  Railway   │
      │  US-East   │      │  EU-West   │      │  AP-South  │
      │ (default)  │      │            │      │  (DPDP)    │
      └─────┬──────┘      └─────┬──────┘      └─────┬──────┘
            │                   │                   │
   ┌────────┴────────┐ ┌────────┴────────┐ ┌────────┴────────┐
   │ Neon us-east-1  │ │ Neon eu-west-1  │ │ Neon ap-south-1 │
   │ Upstash us-east │ │ Upstash eu-west │ │ Upstash ap-south│
   │ S3 us-east-1    │ │ S3 eu-west-1    │ │ S3 ap-south-1   │
   └─────────────────┘ └─────────────────┘ └─────────────────┘

   Shared services (no PII): Sentry · BetterUptime · Resend · Stripe ·
   Apple SS · Google Play Dev API · FCM · APNs · Twilio Verify
```

### 8.3 Environment Matrix

| Environment | Purpose | Compute | Data | Trigger |
| --- | --- | --- | --- | --- |
| `local` | Dev laptop | Docker Compose (Postgres+Redis) | Synthetic seed | manual `npm run dev` |
| `preview` | Per-PR ephemeral | Railway preview env | Neon branch, anonymised | open PR |
| `staging` | Pre-prod integration | Railway staging service | Anonymised prod copy nightly | merge to `main` |
| `prod-us`, `prod-eu`, `prod-ap` | Live | Railway 1×0.5vCPU/512MB → autoscale | Real PII (residency-correct) | manual approval gate |

### 8.4 Containerisation
- **Single Dockerfile**, multi-stage (Node 20 Alpine). `railway.toml` declares the start command + health check.
- **No Kubernetes at MVP.** Railway supplies process supervision, auto-restart, and rolling deploys. The architecture supports moving to AWS ECS-Fargate when scaling triggers fire (Section 18 of TRD).
- **Image registry:** GHCR; images signed with cosign; SBOM via syft attached to release.

### 8.5 CI/CD Pipeline (GitHub Actions)

```
PR opened
 ├── lint (eslint, prettier --check)
 ├── typecheck (tsc --noEmit)
 ├── unit tests (Jest)
 ├── integration tests (testcontainers Postgres + Redis)
 ├── SAST (Semgrep w/ ts-node-security ruleset)
 ├── dep scan (Snyk — high/critical block merge)
 ├── secret scan (gitleaks)
 ├── docker build + cosign sign
 ├── deploy preview env on Railway (per-PR Neon branch)
 └── E2E smoke (Patrol against preview env)

merge to main
 ├── prisma migrate deploy (staging)
 ├── Railway auto-deploy to staging
 ├── DAST (OWASP ZAP baseline) against staging
 ├── E2E full suite (Patrol)
 ├── manual approval (Release Manager + CTO via GitHub Environments)
 ├── Railway deploy to prod (rolling)
 ├── post-deploy smoke (/health, /v1/users/identity/options, sample login)
 └── auto-rollback if error rate > 1% within 10 minutes
```

### 8.6 Secrets Management
- All credentials live in Railway encrypted env vars (per environment).
- JWT keys rotated every 90 days via overlap window: new `kid` introduced, both keys honoured for 24h, then old key retired.
- KMS-managed envelope key per Neon region for cryptographic delete.

### 8.7 Rollback Strategy
- **Tactical (instant):** Railway "Redeploy previous" — single click, rolls back to last green Docker image (≤ 2 min).
- **DB migrations:** Every migration must be backward-compatible across one deploy. If a forward migration ships with a code change, deploy migration **first**, code **second**. Down-migrations are written but only applied as last resort with DBA approval.
- **Feature flags:** Unleash flags wrap risky paths so we can disable a feature without a rollback (preferred over redeploying).
- **Automatic rollback:** GH Action triggers `railway redeploy --previous` when Sentry error rate > 1% sustained 10 min after deploy.

---

## 9. Performance & Scalability Strategy

### 9.1 Expected Load (MVP)

| Phase | MAU | DAU (≈20%) | RPS peak (estimate) | WS concurrent |
| --- | --- | --- | --- | --- |
| Beta (Month 0–1) | 100 | 20 | 5–10 | ≤ 50 |
| Soft Launch (Month 1–3) | 1 K | 200 | 30–50 | ≤ 500 |
| Early MVP (Month 3–6) | 5 K | 1 K | 80–150 | ≤ 1 K |
| Growth (Month 6–12) | 10–25 K | 2–5 K | 150–250 | ≤ 3 K |

Daily data growth at 5K MAU: ~50 MB chat ciphertext + ~1 GB photos. Neon storage stays under Launch-plan quota.

### 9.2 Bottleneck Analysis

| Component | Likely degradation | Trigger | Action |
| --- | --- | --- | --- |
| Discovery GIN query | Feed p95 > 500 ms | > 50 K active profiles | Tune index / add Meilisearch / extract module + add Elasticsearch |
| DB connections | "too many connections" | concurrent users > 500 | Upgrade Neon plan (PgBouncer already in path) |
| WebSocket memory | RAM > 80% | concurrent WS > 3 K | Scale Railway to 2 GB or 2× replicas (sticky sessions via WS routing) |
| Upstash commands | Upstash 429 | > 5 M cmd/month | Upstash fixed plan ($10/mo) |
| S3 egress | bill > $50/mo | > 25 K active users | Add CloudFront for media |
| Single-process CPU | > 80% sustained | any spike | 2× replicas (Fastify is stateless except WS — see sticky-routing note) |

### 9.3 Horizontal vs Vertical Scaling

| Concern | Strategy |
| --- | --- |
| HTTP REST | Stateless — scale horizontally by replica count behind Railway router. |
| WebSocket | Each WS pinned to a process. Phase 1 keeps 1 replica → no sticky routing needed. Phase 1.5 (multi-replica) uses Upstash pub/sub to fan messages between replicas. |
| DB | Vertical (Neon plan upgrade) until 25 K MAU; then read replicas for discovery query if needed. |
| Redis | Upstash auto-scales; vertical via plan upgrade. |
| S3 | Inherently scalable. |
| `modules/chat` | First candidate for horizontal extraction at the microservices boundary. |

### 9.4 Caching Strategy

| Layer | What | TTL | Why |
| --- | --- | --- | --- |
| Upstash | `block:{userId}` Set | 1 h / invalidate on mutation | Hot-path discovery exclude. |
| Upstash | `geo:{prefix5}` Sorted Set of active userIds | 5 min | Avoid Postgres scan for adjacency. |
| Upstash | `q:like:{userId}:{date}` counter | 24 h | Daily quota. |
| Upstash | `presence:{userId}` boolean | 90 s (heartbeat) | WS presence + online indicator. |
| In-process | Prisma client query cache for identity options | 10 min | Static-ish reference data. |
| Mobile | `dio_cache_interceptor` on `/v1/users/identity/options` | 24 h | Reduce cold-start traffic. |
| HTTP | `Cache-Control: private, no-cache` on user-scoped responses | — | Prevents intermediate caching of personal data. |

### 9.5 Async Processing
- **Outbox-driven dispatcher**: an in-process worker polls `outbox WHERE status='pending' AND next_run_at <= NOW()` every 250 ms. Push notifications, webhook fan-outs, and deletion-pipeline jobs flow through it.
- **Retry policy:** exponential backoff (1s, 4s, 16s, 1m, 5m, 1h), max 8 attempts, then `status='failed'` + Sentry alert.
- **At-least-once delivery:** consumers idempotent on `outbox.id` (push has natural dedupe on `notification_log.payload_digest`).
- When `modules/chat` is extracted (Section 18 of TRD), the outbox graduates to a dedicated Postgres or NATS queue.

### 9.6 Read Replicas / Sharding
- **MVP:** none. Single Neon writer.
- **Trigger:** discovery feed > 80 ms p99 OR DB CPU > 70%.
- **Plan:** Neon read replicas with Prisma `previewFeatures = ["multipleConnections"]`. Discovery feed query becomes replica-routed. Writes still go to primary.
- **Sharding:** explicitly deferred until > 200K MAU; the geohash prefix gives us a natural shard key when that arrives.

---

## 10. Observability Plan

### 10.1 Logging
- **Format:** JSON via Pino. Required fields: `ts`, `level`, `request_id`, `user_id` (when authenticated), `module`, `event`, `latency_ms`, `status_code`.
- **PII rules:** never log phone numbers, OTP codes, JWTs, ciphertext, S3 keys with user identifiers, or geohashes longer than 5 chars.
- **Levels:** `trace` (dev only), `debug` (preview only), `info` (default), `warn`, `error`, `fatal`.
- **Pipelines:** stdout → Railway log drain → Sentry Logs (free tier).

### 10.2 Metrics (instrumented via prom-client → OpenTelemetry exporter)

| Metric | Type | Labels |
| --- | --- | --- |
| `http_request_duration_ms` | histogram | method, route, status |
| `ws_message_dispatch_ms` | histogram | direction |
| `db_query_duration_ms` | histogram | module, operation |
| `outbox_pending` | gauge | kind |
| `auth_otp_attempts_total` | counter | result |
| `safety_reports_total` | counter | severity |
| `match_likes_total` | counter | premium |
| `notif_dispatch_total` | counter | platform, result |
| `chat_active_ws_connections` | gauge | — |

### 10.3 Tracing
- OpenTelemetry SDK in Fastify, exporting OTLP → Sentry Performance. Every request has a trace; spans for DB, Redis, S3, Twilio, FCM/APNs, Stripe.
- Trace IDs propagated to Sentry errors so a single `request_id` joins logs + traces + errors.

### 10.4 Alerting

| Alert | Threshold | Channel |
| --- | --- | --- |
| `http_5xx_rate` > 0.5% for 5 min | warn → PagerDuty | P1 page |
| `p99_latency` > 2× SLO for 5 min | warn | Sentry alert → Slack |
| `db_connections` > 80% | warn | Slack |
| `auth_failure_rate` > 100/min | warn | Slack + Security on-call |
| `chat_active_ws_connections` > 80% capacity | warn | Slack |
| CRITICAL report created | immediate | PagerDuty page (no threshold) |
| Stripe webhook failures > 3 / 10 min | warn | Slack |
| `app_crash_free_rate` < 99% / 1 h | warn | Sentry alert |
| `/health` non-200 > 3 min | page | BetterUptime → PagerDuty |
| Auto-rollback fired | info | Slack |

### 10.5 Health Checks
- `GET /health` — returns 200 if process up, DB ping < 200 ms, Redis ping < 200 ms. Used by Railway liveness + BetterUptime probe.
- `GET /ready` (internal-only) — returns 200 only once warm caches loaded; Railway readiness probe.
- `/metrics` exposed on a separate internal-only port (`9091`) for future Prometheus scrape (currently OTel push to Sentry).

---

## 11. Testing Strategy

### 11.1 Test Pyramid

```
                  ┌────────────────┐
                  │ Manual QA +    │  (T&S drills, panic-button device matrix)
                  │ exploratory    │
                  └────────────────┘
                ┌────────────────────┐
                │ E2E (Patrol)       │  Critical user journeys
                └────────────────────┘
            ┌──────────────────────────┐
            │ Contract (Pact)          │  Inter-module event schemas
            └──────────────────────────┘
        ┌──────────────────────────────────┐
        │ Integration (Jest + testcontain.)│  REST + WS + DB + Redis
        └──────────────────────────────────┘
   ┌──────────────────────────────────────────────┐
   │ Unit (Jest + Supertest, flutter_test)        │  80% backend / 70% mobile
   └──────────────────────────────────────────────┘
```

### 11.2 Unit Test Targets

| Area | Critical cases |
| --- | --- |
| `auth.verifyOtp` | expired code, wrong code, lockout boundary, family rotation |
| `match.like` | self-like, blocked target, simultaneous reciprocal, daily quota boundary, premium unlimited |
| `discovery.feed` | block-set exclusion, contact-block exclusion, incognito exclusion, age boundary, intent intersection, no-results path |
| `safety.createReport` | CRITICAL → immediate suspend + PagerDuty stub asserted, severity classification |
| `billing.gates.isPremium` | grace period, refund, cross-platform, expired |
| `chat.deliverEnvelope` | recipient blocked sender, recipient offline → outbox enqueue |
| `media.uploadPipeline` | CSAM match rejection, pHash duplicate, > 10 MB rejection |
| age + birth-date | `<18` rejection, leap-year edge, time-zone boundary |

### 11.3 Integration Boundaries
- **Real:** Postgres (testcontainers), Redis (testcontainers), Prisma — these are the system's truth.
- **Mocked:** Twilio Verify (stub OTP `000000`), FCM/APNs (record-only), S3 (LocalStack), Apple/Google IAP (fixture receipts), PhotoDNA (offline hash list).

### 11.4 E2E Critical Journeys (Patrol)
1. Sign-up with phone OTP → identity → first photo → feed loads.
2. Like a profile → mutual match → push notification (silent assert via mock) → open conversation.
3. Send and receive E2EE text + image messages on 2 emulators.
4. Block from chat → conversation disappears both sides within 500 ms.
5. Report CRITICAL → target suspended → audit row written.
6. Panic button 3-finger long-press → decoy screen within 500 ms.
7. Subscribe via Stripe sandbox → premium gate flips.
8. GDPR delete → re-login attempt fails → export available.

### 11.5 Performance Testing
- **k6** load script, run pre-launch and weekly thereafter against staging.
- Scenarios: `feed_browse` (60 rph/user × 1 K users), `match_flow` (likes + push), `chat_burst` (100 msg/min × 200 users), `auth_storm` (OTP 5/h cap stress).
- **Pass criteria:** all SLOs in TRD §15.1 met at 2× expected Day-1 load.

### 11.6 Test Data Management
- **Synthetic only** in non-prod. No production data ever copied to dev.
- **Staging:** nightly anonymised snapshot — names, emails, phones HMAC-replaced; geohashes shifted by a fixed seed; photos replaced with placeholders.
- **Seed fixtures:** versioned in `prisma/seed/` — covers all gender/orientation combinations + 3 admin roles + IAP fixture receipts.

---

## 12. Development Guidelines

### 12.1 Folder Structure (monorepo)

```
spectrum/
├─ backend/
│  ├─ src/
│  │  ├─ index.ts                     # Fastify bootstrap
│  │  ├─ app.ts                       # plugin registration
│  │  ├─ modules/
│  │  │  ├─ auth/
│  │  │  │  ├─ routes.ts
│  │  │  │  ├─ service.ts
│  │  │  │  ├─ repository.ts
│  │  │  │  ├─ otp.ts
│  │  │  │  ├─ jwt.ts
│  │  │  │  ├─ types.ts
│  │  │  │  └─ __tests__/
│  │  │  ├─ user/
│  │  │  ├─ discovery/
│  │  │  ├─ match/
│  │  │  ├─ chat/
│  │  │  │  ├─ ws.ts
│  │  │  │  └─ ...
│  │  │  ├─ safety/
│  │  │  ├─ notification/
│  │  │  ├─ billing/
│  │  │  ├─ media/
│  │  │  └─ admin/
│  │  ├─ shared/
│  │  │  ├─ db/prisma.ts
│  │  │  ├─ redis/upstash.ts
│  │  │  ├─ s3/client.ts
│  │  │  ├─ logger.ts
│  │  │  ├─ telemetry.ts
│  │  │  ├─ outbox/dispatcher.ts
│  │  │  ├─ middleware/
│  │  │  │  ├─ jwt.ts
│  │  │  │  ├─ rateLimit.ts
│  │  │  │  ├─ errorHandler.ts
│  │  │  │  └─ requestId.ts
│  │  │  └─ config/
│  │  │     ├─ env.ts                 # zod-validated env
│  │  │     └─ identity-options.json
│  │  └─ types/
│  ├─ prisma/
│  │  ├─ schema.prisma
│  │  ├─ migrations/
│  │  └─ seed/
│  ├─ Dockerfile
│  ├─ railway.toml
│  ├─ package.json
│  └─ tsconfig.json
├─ mobile/
│  ├─ lib/
│  │  ├─ main.dart
│  │  ├─ features/
│  │  │  ├─ auth/  discovery/  chat/  match/  safety/  billing/  profile/
│  │  ├─ core/
│  │  │  ├─ api/  storage/  signal/  router/  theme/
│  │  └─ shared/ widgets/
│  ├─ ios/
│  ├─ android/
│  └─ test/
├─ infra/
│  ├─ terraform/                       # only AWS (S3, IAM, KMS, Route53)
│  └─ railway/                          # railway.toml templates
├─ .github/workflows/
│  ├─ pr.yml
│  ├─ main.yml
│  └─ release.yml
└─ docs/
   ├─ ARCHITECTURE.md
   ├─ RUNBOOKS/
   └─ ADRs/
```

### 12.2 Naming Conventions

| Element | Rule | Example |
| --- | --- | --- |
| Files (TS) | kebab-case for files, PascalCase for components | `match-service.ts`, `MatchService` |
| Files (Dart) | snake_case | `match_card.dart` |
| Variables | camelCase | `swiperId` |
| Constants | UPPER_SNAKE | `MAX_LIKES_FREE` |
| DB tables | snake_case plural | `user_locations` |
| DB columns | snake_case | `incognito_since` |
| API routes | kebab-case under `/v1/` | `/v1/safety/contact-block` |
| Env vars | UPPER_SNAKE prefixed by domain | `JWT_PRIVATE_KEY`, `S3_BUCKET` |
| Module boundary tags | `// EXTRACT_BOUNDARY: <target_module> ← <caller_module>` | as in TRD |

### 12.3 Branching: Trunk-Based with Short-Lived Branches
- `main` is always deployable.
- Feature branches: `feat/<ticket>-short-slug`. Max lifetime 3 days.
- Hotfixes: `hotfix/<ticket>` → merged directly via expedited approval.
- Release tags: `vYYYY.MM.DD-N` on every prod deploy.

### 12.4 Commit Messages — Conventional Commits

```
<type>(<scope>): <imperative summary>

[optional body]

[optional footers: BREAKING CHANGE:, Refs: TICKET-123]
```

Types: `feat | fix | chore | refactor | perf | test | docs | ci | build | security`.
Scopes: module names (`auth`, `match`, `chat`, ...) or `infra`, `mobile`, `db`.

### 12.5 Code Review Checklist (mandatory)
1. New endpoint? Added to OpenAPI + has Zod schema + integration test.
2. New DB column? Migration is backward-compatible + indexed if read-path.
3. Cross-module call? Tagged `EXTRACT_BOUNDARY`.
4. PII added? Confirmed not logged + retention defined.
5. Secret introduced? Lives in Railway env vars + documented in `env.ts` zod schema.
6. Rate-limited path? Limit defined + Upstash key naming consistent.
7. Premium feature? Server-side enforcement present + test asserts 403 for free users.
8. Security review tag (`security:`) requires +1 from Security Engineer.

### 12.6 Tooling
- **Lint:** ESLint + `@typescript-eslint` + `eslint-plugin-security` + `eslint-plugin-import` (no relative `../../..` beyond 2 levels). Dart: `flutter_lints`.
- **Format:** Prettier (2-space, semi, single-quote). `dart format`.
- **Pre-commit (Husky + lint-staged):** lint, format, typecheck (changed files), gitleaks scan, run unit tests for changed module.
- **Commit lint:** commitlint with Conventional Commits config.
- **Codeowners:** `CODEOWNERS` enforces module owners for review.

---

## 13. Risk Register

| # | Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- | --- |
| 1 | Single-replica Railway monolith outage cascades to all features (chat, match, auth) | Medium | High | Auto-restart on failure; BetterUptime + PagerDuty; deploy multi-replica at first sign of WS or CPU pressure; rollback automation | DevOps Lead |
| 2 | SMS pumping fraud inflating Twilio Verify costs | Medium | High | Cloudflare Turnstile on `/auth/otp/request`; per-IP and per-phone rate limits in Upstash; Twilio's geo + carrier blocklists | Security Eng |
| 3 | Moderator over-reach attempts to access chat content | Low | Critical | Chat tables hold only ciphertext; admin API has no endpoint that returns plaintext; immutable audit log; quarterly access review | T&S Lead |
| 4 | E2EE key loss when user reinstalls (Signal private keys in Keychain only) | High | Medium (UX) | Documented "messages don't migrate on reinstall" UX; provide encrypted iCloud/Google Drive backup of identity key as Phase 1.5 enhancement | Mobile Lead |
| 5 | Vendor lock-in to Neon (proprietary Postgres extensions, branching) | Medium | Medium | Stick to vanilla PG features; document migration playbook to AWS RDS; quarterly export test | Backend Lead |
| 6 | Discovery GIN scan degrades sharply past 50K profiles | Medium | High | Pre-computed materialised view candidates per geohash prefix as a fallback; Meilisearch readiness as Phase 1.5; trigger plan owns the cutover playbook | Backend Lead |
| 7 | LGBTQ+ identity data leaked via app-store metadata / push payloads (outing risk) | Medium | Critical | Push payloads are generic ("New match!", no name); App Store screenshots reviewed; explicit content-declaration + 17+ rating | Privacy Eng |
| 8 | CSAM upload bypasses PhotoDNA via novel image | Low | Critical | PhotoDNA hash + perceptual hash + manual review queue; staffed moderators 24/7 from launch; NCMEC CyberTipline integration; immediate suspend on report | T&S Lead |
| 9 | Stripe / IAP receipt forgery granting fake premium | Low | Medium | Server-side verification with Apple/Google APIs on every claim; webhook signature verification on `Stripe-Signature`; no client-side premium flag trust | Backend Lead |
| 10 | DPDP localisation breach (Indian user data leaves AP region) | Low | Critical | Region-pinned Neon + Upstash + S3 for India; routing by registration country; quarterly audit | Privacy Eng |
| 11 | Refresh-token theft from compromised device | Medium | Medium | Single-use rotation + family invalidation on reuse; biometric step-up for sensitive ops | Security Eng |
| 12 | Photo storage cost spirals past credit window | Medium | Medium | 30-day retention on chat media; lifecycle rules to S3 Glacier IA after 90 days; consider CloudFront only past 25K users | DevOps Lead |
| 13 | Insider abuse: engineer queries Neon prod for user PII | Medium | High | No direct prod DB access except 2 named DBAs + break-glass with audit + JIT access via 1Password | Security Eng |
| 14 | Geohash precision 7 still narrow enough to fingerprint individuals in low-density areas | Low | Medium | Distance shown as bucketed ranges (Section 6.4); profiles in low-density geohashes get reduced precision (prefix 5) before display | Privacy Eng |
| 15 | App Store rejection of LGBTQ+ content in restrictive markets | Medium | High | Launch markets pre-cleared (US, UK, India); legal sign-off per region; content declaration accurate; have backup web app channel via Stripe | Product Lead |

---

## 14. Open Questions & Decisions Log

### Open Questions (snapshot)
1. Boost economics (duration, frequency, multiplier).
2. Photo-verification automation timeline (Rekognition vs. manual).
3. Face-blur policy for photos 2–9.
4. Refresh-token family-invalidation policy details.
5. Soft-delete retention window for blocked conversations.
6. Multi-region DR posture (RTO/RPO formalisation).
7. Admin VPN technology and ACL ownership.
8. iCloud/Google Drive backup of Signal identity key (Phase 1.5?).

### Architecture Decision Records

#### ADR-001: Monolithic-Modular over Microservices for MVP
**Status:** Decided
**Context:** Choice between the original 10-service AWS topology and a single Fastify process on Railway, before the product has PMF.
**Options considered:** (a) 10 microservices on ECS Fargate, (b) Modular monolith on Railway, (c) Lambda-per-route serverless.
**Decision:** Modular monolith on Railway.
**Consequences:** Save ~$2,479/mo; lose independent scaling per module; require disciplined `EXTRACT_BOUNDARY` annotation to retain extraction optionality; single point of failure during the 1-replica phase.

#### ADR-002: PostgreSQL GIN over Elasticsearch for Discovery
**Status:** Decided
**Context:** Original TRD specified a 3-node ES cluster ($773/mo).
**Options considered:** Elasticsearch, Meilisearch, PostgreSQL GIN+pg_trgm, OpenSearch.
**Decision:** PostgreSQL GIN with `jsonb_path_ops`.
**Consequences:** Free; sufficient to 50K profiles; complex synonym/typo handling deferred (we don't need it at MVP); Meilisearch remains Plan B at scale trigger.

#### ADR-003: Signal Protocol for Chat E2EE (mandatory)
**Status:** Decided
**Context:** Privacy is a brand pillar; user safety in LGBTQ+ context demands true E2EE.
**Options considered:** Signal Protocol, Matrix/Olm, custom AES-GCM with key exchange.
**Decision:** Signal Protocol via libsignal.
**Consequences:** Forward + future secrecy; server can't comply with content subpoenas (intentional); reinstall loses message history (acceptable, documented UX); higher mobile complexity (offset by mature SDK).

#### ADR-004: REST over GraphQL/tRPC for API
**Status:** Decided
**Context:** Need a Flutter-friendly, cache-friendly, abuse-modellable API.
**Options considered:** REST, GraphQL, tRPC.
**Decision:** REST/JSON.
**Consequences:** Simple Dio client; per-route rate limiting trivial; small endpoint surface; if BFF needs grow, GraphQL can be added on top later.

#### ADR-005: Geohash-Only Location Storage
**Status:** Decided
**Context:** LGBTQ+ users face disproportionate physical-safety risk if precise location data leaks.
**Options considered:** Exact GPS, geohash 7, geohash 5, city-only.
**Decision:** Geohash precision 7 computed client-side; server never sees coordinates.
**Consequences:** ~150 m accuracy (enough for "nearby" feel); zero server liability for GPS data; bucketed distance display; geohash adjacency requires 9-cell scan logic.

#### ADR-006: RS256 JWT with 15-min Access + Single-Use Refresh
**Status:** Decided
**Context:** Cross-module verification with rotation safety.
**Options considered:** HS256 shared-secret, RS256, opaque session tokens.
**Decision:** RS256 with `kid` rotation + single-use refresh + family invalidation.
**Consequences:** Public-key verification anywhere (future microservices); short blast radius if access token leaks; client implements refresh interceptor.

#### ADR-007: Outbox Table for Cross-Module Side Effects
**Status:** Decided (extension to TRD)
**Context:** Notification dispatch must survive process restart and eventual module extraction.
**Options considered:** Direct call, in-memory queue, outbox+poll, external queue (NATS/SQS).
**Decision:** Postgres outbox + 250 ms polling worker in-process.
**Consequences:** Atomic with business mutation (same TX); replayable; transitions cleanly to NATS/SQS when chat module extracts.

#### ADR-008: Railway Single Region (US-East) at MVP, Per-Region Production Stacks Post-Launch
**Status:** Decided
**Context:** DPDP and GDPR residency obligations vs. cost of triplicating infra.
**Options considered:** Single global region, three production stacks day 1, three stacks lazily provisioned.
**Decision:** Provision EU + AP stacks before EU/UK/India launch, not before US-only launch.
**Consequences:** Faster initial launch; DPDP/GDPR not in scope until those market launches; clear runbook to spin up regional stacks.

#### ADR-009: Cryptographic Shredding for GDPR/DPDP Delete
**Status:** Decided (extension to TRD)
**Context:** "Hard delete" of all rows referencing a user is expensive and fragile across modules + S3 + backups.
**Options considered:** Hard delete cascade, soft delete + scheduled purge, cryptographic shredding via KMS.
**Decision:** Per-user envelope key in KMS; on delete, rotate/destroy the key so plaintext PII becomes unrecoverable, then run a 30-day delayed physical purge across modules.
**Consequences:** Compliant within 30 days; backups become safely unreadable once key destroyed; small added complexity in S3 upload path (envelope-encrypt photos with user key).

#### ADR-010: VPN-bound Admin via Tailscale + Cloudflare WAF
**Status:** Decided (extension to TRD)
**Context:** TRD says "VPN IP allowlist" but doesn't pick a vendor.
**Options considered:** Self-hosted WireGuard, OpenVPN, Tailscale, AWS Client VPN.
**Decision:** Tailscale (zero-trust, free tier, ACLs as code) → fixed egress IP → Cloudflare WAF allowlist on `/admin/*`.
**Consequences:** Cheap, fast to set up, easy revocation; depends on Tailscale uptime — backup IP allowlist documented for break-glass.

---

## Documentation Completeness Checklist

| § | Section | Status | Notes |
| --- | --- | --- | --- |
| 1 | System Overview | Fully addressed | — |
| 2 | High-Level Architecture | Fully addressed | Diagrams given as structured text per request. |
| 3 | Technology Stack | Fully addressed | Versions taken from TRD where stated; mobile lib pins are current LTS. |
| 4 | Data Architecture | Fully addressed | DDL is canonical for primary tables; full schema lives in `prisma/schema.prisma`. |
| 5 | API Design | Fully addressed | 42 MVP endpoints catalogued; WS events listed. |
| 6 | System Components | Fully addressed | All 10 modules documented. |
| 7 | Security Design | Fully addressed | Threat model + OWASP + compliance covered. |
| 8 | Infrastructure & Deployment | Fully addressed | Railway-centric with multi-region plan. |
| 9 | Performance & Scalability | Fully addressed | Bottleneck/trigger table from TRD §15.3 extended. |
| 10 | Observability | Fully addressed | OTel + Sentry + Pino + PagerDuty. |
| 11 | Testing Strategy | Fully addressed | Pyramid + journeys + k6 thresholds. |
| 12 | Development Guidelines | Fully addressed | Full directory tree, conventions, commit format. |
| 13 | Risk Register | Fully addressed | 15 concrete risks identified. |
| 14 | Open Questions & ADRs | Partially addressed (Open Questions) | 8 open questions need stakeholder input — see list. |

### Items Requiring Stakeholder Input (collected)
1. **Product:** Boost product spec (duration, frequency, free allotment per tier).
2. **Trust & Safety:** photo verification timeline — Rekognition Phase 2 commit date + interim moderator headcount sizing.
3. **Privacy/Legal:** confirm soft-delete retention window for blocked conversations (proposed: 180 days).
4. **Security:** approve refresh-token family-invalidation policy as written; approve Tailscale + Cloudflare WAF combo for admin.
5. **DevOps:** approve multi-region production stack rollout calendar tied to EU + India launches.
6. **Mobile:** decide on Signal identity-key backup story (iCloud/Drive encrypted blob) — Phase 1 or Phase 1.5.
7. **Finance:** confirm budget envelope and credit-programme application status (Neon, Railway, Upstash, Twilio, Sentry).
8. **Legal:** DPO + Grievance Officer appointment dates locked to market launch dates.

---

*End of Spectrum Technical Documentation Suite v1.0.*
