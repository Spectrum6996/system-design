# Data Architecture

> The canonical production schema lives in `backend/prisma/schema.prisma`. The DDL below is an abridged reference.

---

## 4.1 Entity Relationship Overview

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

---

## 4.2 Primary Schema (DDL — abridged)

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

-- ============ OUTBOX (guaranteed cross-module side effects) ============
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

---

## 4.3 Indexing Strategy

| Index | Purpose | Reasoning |
| --- | --- | --- |
| `idx_users_active (status, incognito_since, last_active) WHERE status='active'` | Discovery candidate set | Partial index keeps it small; the most selective predicate is `status='active'`. |
| `idx_ul_geohash_prefix (geohash text_pattern_ops)` | Prefix range scans across 9 cells | `text_pattern_ops` lets `LEFT(geohash,5) = ANY(...)` use a B-tree range. |
| `idx_identity_open_to_gin USING GIN (open_to jsonb_path_ops)` | `open_to @> $callerGender` | GIN with `jsonb_path_ops` is 2-3× faster than the default GIN for containment. |
| `idx_ip_intent (relationship_intent)` | Intent filter | Low cardinality but combined with geohash narrows candidate set efficiently. |
| `idx_swipes_swiper (swiper_id)` | Exclude already-swiped from feed | High-write table; this index is essential to the discovery `NOT IN` join. |
| `idx_swipes_pair (swiped_id, swiper_id)` | Mutual-like lookup in `match.like` | Composite supports the symmetric check `WHERE swiper=$other AND swiped=$me`. |
| `idx_blocks_blocker (blocker_id)`, `idx_blocks_blocked (blocked_id)` | Bidirectional exclusion | Discovery loads `blocks` into Redis Set; queries also exist for fallback. |
| `idx_messages_conv_sent (conversation_id, sent_at DESC)` | Paginated message history | Always sorted DESC; covers the most common query. |
| `idx_reports_severity_status (severity, status, created_at DESC)` | T&S queue | Pinned CRITICAL at top relies on this ordering. |
| `idx_outbox_status_next_run (status, next_run_at) WHERE status='pending'` | Async worker dequeue | Partial index, small. |
| `idx_devices_user_lastseen (user_id, last_seen DESC)` | Active device lookup for push | Most-recent device first. |

---

## 4.4 Data Access Patterns

**Read-heavy:** `/discovery/feed` (60+ rps/user), `/chat/conversations` (every app open), `/users/:id` (profile inspection). All cacheable.

**Write-heavy:** `swipes` (every card view that triggers like/pass), `messages` (chat), `outbox`.

**Cache layers:**
- Upstash Redis **per-user block set** (`block:{userId}` Set) — loaded on session start, invalidated on `/safety/block` mutation.
- Upstash **geohash neighbour cache** (`geo:{prefix5}` Sorted Set) of active userIds — refreshed every 5 minutes by a Fastify scheduled job; expired naturally.
- Upstash **daily quotas** (`q:like:{userId}:{yyyymmdd}` counter with EXPIRE midnight UTC).
- Sentry/Fastify HTTP cache headers (private, no-cache) on user-specific responses; CDN on static assets only.

**Pagination:** Cursor-based for chronological lists (`messages`, `matches`, `reports`); offset-based for the discovery feed (acceptable because feeds are ephemeral and shuffled). Cursor format: opaque base64 of `(sortField, id)`.

**Hot-path query latency budget:** feed query < 80 ms p50 / < 200 ms p99 at 50K profiles, verified on Neon Launch plan.

---

## 4.5 Migration & Seeding

- **Migrations:** Prisma Migrate (`prisma migrate dev` locally, `prisma migrate deploy` in CI). Every migration must be backward-compatible across one deploy (online migration discipline: add column → backfill → switch reads → drop later).
- **Schema branching:** Neon's branch-per-PR feature gives each PR its own database; CI runs `prisma migrate deploy` then integration tests.
- **Seeding:** `prisma/seed.ts` provisions: 3 admin roles, 50 synthetic users across genders/orientations for E2E tests, an outbox-clean state, and identity-options.json content.
- **Data residency:** Two Neon projects (`spectrum-prod-eu`, `spectrum-prod-ap`) for EU and India regions; users routed by registration country. (US users default to `us-east-1` Neon project.)
