# High-Level Architecture

---

## Component Layer Diagram

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
│           Single Node 24 / Fastify 5 process — 1 replica             │
│   ┌──────────────────────────────────────────────────────────────┐   │
│   │                    Cross-cutting middleware                  │   │
│   │  JWT verify (RS256) │ Rate limit (Upstash) │ Request log     │   │
│   │  Zod validation     │ Error envelope       │ OpenTelemetry   │   │
│   └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   ┌──────────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐ ┌─────────┐  │
│   │ modules/ │ │ modules/ │ │  modules/  │ │ modules/ │ │modules/ │  │
│   │   auth   │ │   user   │ │ discovery  │ │  match   │ │  chat   │  │
│   │OAuth+JWT │ │profile+  │ │geohash feed│ │like/pass │ │WSS+E2EE │  │
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
   │ 18 + GIN idx │    │rate-limit,   │    │ media,      │
   │ PgBouncer    │    │geohash sets, │    │ encrypted   │
   │              │    │block sets    │    │ ciphertext  │
   └──────────────┘    └──────────────┘    └─────────────┘

   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐
   │ Google OAuth  │    │FCM v1 + APNs │    │Stripe + IAP │
   │   (auth)      │    │(push)        │    │(billing)    │
   └──────────────┘    └──────────────┘    └─────────────┘

   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐
   │ Sentry       │    │BetterUptime  │    │ NCMEC/      │
   │ (errors)     │    │ (uptime)     │    │ PhotoDNA    │
   └──────────────┘    └──────────────┘    └─────────────┘
```

---

## Module Responsibilities

| Module | Responsibility |
| --- | --- |
| `auth` | Google OAuth sign-in, JWT mint, refresh rotation, device registry, logout, session lockout |
| `user` | Profile CRUD, identity options, visibility settings, account export, deletion pipeline |
| `discovery` | Geohash feed query, filters, scoring, contact-block enforcement |
| `match` | Swipe state machine, mutual-match creation, daily-like quotas, unmatch, undo (premium) |
| `chat` | WSS gateway, Signal pre-key store, ciphertext message persistence, media-key envelope, typing indicators |
| `safety` | Block graph, report intake, severity routing, moderator action log, panic-mode hooks |
| `notification` | Device-token registry, FCM/APNs dispatch, privacy-safe payloads, outbox replay |
| `billing` | Stripe subscriptions, Apple/Google IAP verification, webhook handling, premium flag mutation |
| `media` | S3 presigned URLs, face-blur queue, CSAM hash check, pHash duplicate prevention |
| `admin` | T&S dashboard, RBAC, audit log, moderator action surface, system metrics |

---

## Data Flow Narratives

### Sign-up + first feed load (cold start)

1. Mobile client initiates Google OAuth → receives `id_token` → posts `/auth/google` → middleware: Turnstile + per-IP Upstash rate limit → `auth.googleSignIn()` verifies `id_token` with Google.
2. On first time, `user.createUser()` (DB write, `is_verified=true`); JWT pair signed RS256.
3. Client uploads geohash via `/users/location` → `discovery.upsertLocation()` writes `user_locations` + updates Upstash geohash set.
4. Client requests `/discovery/feed?limit=50` → JWT middleware → `discovery.feed(userId)` → reads geohash, fetches 9 adjacent prefixes from Upstash, runs PostgreSQL GIN query with `open_to @> caller_gender`, filters swipes+blocks (loaded into Redis Set at session start), scores, returns 50 cards.

### Like → mutual match → push notification (hot path)

1. `POST /matches/like/:targetId` → JWT middleware extracts `userId` → `match.like(userId, targetId)`.
2. `match.like` opens a transaction: `INSERT INTO swipes ... liked=true`, checks reverse swipe — if reverse row exists and `liked=true`, inserts `matches` row, inserts `conversations` row.
3. On match success, `// EXTRACT_BOUNDARY: notification ← match` → `notificationModule.dispatch({ userId: otherUser, type: 'match_new', matchId })`. Notification module enqueues to outbox table, returns; Fastify response unblocked.
4. Async worker (same process) reads outbox → posts FCM/APNs → marks delivered.

### Send an E2EE message

1. Client opens WSS `/chat?token=<jwt>` (Fastify @fastify/websocket).
2. Client computes Signal session (or initiates X3DH on first message), encrypts payload, sends `{type:'message', conversation_id, ciphertext, ...}`.
3. `chat.ws.onMessage` persists ciphertext in `messages` (server cannot decrypt), updates `conversations.last_message_at`, pushes envelope down recipient's open WSS if connected, otherwise dispatches a "you have a new message" push via Notification.
4. Recipient client decrypts locally using Double Ratchet state, displays.

### Report a user (CRITICAL severity)

1. `POST /safety/report { reportedUserId, category, description }` → `safety.createReport()`.
2. Severity classifier maps category → if CRITICAL, immediate `users.status='suspended'` on target inside the same transaction, write `moderation_actions` row, push to PagerDuty via webhook.
3. Reporter receives 202; CRITICAL row pinned in T&S queue with red badge.

---

## External Integration Map

```
┌──────────────┐  id_token verify ┌────────────┐
│ modules/auth │ ────────────────▶│  Google    │
└──────────────┘                   │  OAuth     │
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
