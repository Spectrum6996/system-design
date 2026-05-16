# System Components — Module Design

Each module owns a specific domain, exposes typed interfaces to other modules, and is tagged with `EXTRACT_BOUNDARY` comments at every cross-module call site. This makes future microservice extraction mechanical.

---

## modules/auth

- **Purpose:** Google OAuth identity verification, JWT lifecycle, device-bound refresh tokens.
- **Sub-components:** `routes.ts`, `service.ts`, `repository.ts`, `oauth.ts` (Google id_token verifier), `jwt.ts` (RS256 with kid header), `device.ts`.
- **Interfaces (internal):** `auth.verifyJWT(token)`, `auth.requireRole(roles[])`, `auth.invalidateFamily(userId)`.
- **Dependencies:** Google OAuth (id_token verification), Upstash (rate limits + lockouts), Prisma.
- **State owned:** `users (auth fields)`, `devices`, `auth_sessions`.
- **Edge cases:** Duplicate Google account (same `google_sub` on re-registration returns existing user); Google token expiry (force re-OAuth after JWT age > 24h on suspicious device fingerprint); refresh-token reuse (invalidate entire family, force re-login).

---

## modules/user

- **Purpose:** Profile, identity, account lifecycle, consent log, deletion pipeline.
- **Sub-components:** `routes.ts`, `service.ts`, `identityOptions.ts`, `consent.ts`, `deletion.ts`.
- **Interfaces:** `user.get(id)`, `user.updateProfile(id, patch)`, `user.startDeletion(id)`.
- **Dependencies:** Prisma, S3 (export bucket), Notification (for deletion confirmation).
- **State owned:** `users`, `identity_profiles`, `consent_log`.
- **Edge cases:** Underage report → immediate suspension + audit; identity custom-text length overflow → 422; account deletion in flight → all reads return 404 to other users (caller sees grace screen).

---

## modules/discovery

- **Purpose:** Build personalised candidate feed using geohash + identity filters.
- **Sub-components:** `routes.ts`, `feed.ts` (query builder), `geo.ts` (geohash neighbour math), `score.ts`.
- **Interfaces:** `discovery.feed(userId, filters)`, `discovery.refreshBlockSet(userId)`.
- **Dependencies:** Prisma (GIN query), Upstash (block sets + geohash cache).
- **State owned:** `user_locations`, `discovery_filters`.
- **Edge cases:** New user with no geohash → returns city-fallback feed; user travelling rapidly → geohash updated only on app foreground; contact-block matched users excluded silently (never appear); incognito mode → user removed from any other user's feed within 5 min (verified across all 9 cells).

---

## modules/match

- **Purpose:** Like/pass/unmatch state machine; mutual match creation; daily quotas.
- **Sub-components:** `routes.ts`, `service.ts`, `quota.ts`.
- **Interfaces:** `match.like(swiper, target)`, `match.unmatch(matchId, userId)`.
- **Dependencies:** Prisma (transactional), Notification (EXTRACT_BOUNDARY), Upstash (quota counters).
- **State owned:** `swipes`, `matches`.
- **Edge cases:** Self-like (422), like blocked user (404), simultaneous reciprocal like (DB unique constraint resolves; idempotent on retry), undo on non-premium (403), undo when more than the last swipe is involved (only the most recent pass is undoable).

---

## modules/chat

- **Purpose:** WSS gateway + ciphertext storage + Signal pre-key store + media keys.
- **Sub-components:** `ws.ts` (Fastify WS plugin), `routes.ts` (REST for history + keys), `service.ts`, `keys.ts`.
- **Interfaces:** `chat.deliver(envelope)`, `chat.getKeys(userId)`, `chat.consumeOTK(userId)`.
- **Dependencies:** Prisma, Upstash (presence — `presence:{userId}` with TTL), S3 (encrypted media), Notification (offline push).
- **State owned:** `conversations`, `messages`, pre-key bundles (in `chat_keys` table — sub-table of users).
- **Edge cases:** Recipient offline (push fallback with privacy-safe body), recipient blocked sender (silent drop, returns 200 to sender), invalid ciphertext (server doesn't validate semantics; size cap enforced 64KB), one-time pre-key exhaustion (client warned at < 10 remaining via `chat.replenish_otks` event).

---

## modules/safety

- **Purpose:** Bidirectional block graph; report intake & severity classification; panic mode hooks.
- **Sub-components:** `routes.ts`, `service.ts`, `reportClassifier.ts`, `panic.ts`.
- **Interfaces:** `safety.isBlocked(a,b)`, `safety.createReport(...)`, `safety.escalateCritical(reportId)`.
- **Dependencies:** Prisma, Upstash (block sets), PagerDuty (webhook), Notification.
- **State owned:** `blocks`, `reports`, `moderation_actions` (write-only by admins).
- **Edge cases:** Block during open WSS (server force-closes the conversation socket frame to both sides); reporter blocks reported (block proceeds normally — report still queued for moderator); duplicate report (rate-limited at 10/day; duplicates merged in queue UI).

---

## modules/notification

- **Purpose:** Outbox-driven dispatch of FCM/APNs notifications + in-app store.
- **Sub-components:** `service.ts` (internal API), `dispatcher.ts` (worker tick), `templates.ts`, `outbox.ts`.
- **Interfaces:** `notification.dispatch(userId, kind, payload)` — enqueues to outbox.
- **Dependencies:** Prisma (outbox + devices), FCM Admin SDK, node-apn.
- **State owned:** `outbox`, `devices`, `notification_log`.
- **Edge cases:** Panic mode active → suppress dispatch (server suppression key is `users.panic_active` cached in Redis); stale token → APNs feedback parsed daily; user has > 3 devices → keep latest 3, drop oldest.

---

## modules/billing

- **Purpose:** Subscription state-of-truth, IAP receipt validation, Stripe.
- **Sub-components:** `routes.ts`, `apple.ts`, `google.ts`, `stripe.ts`, `webhooks.ts`, `gates.ts` (used by other modules).
- **Interfaces:** `billing.isPremium(userId)`, `billing.activate(...)`, `billing.cancel(...)`.
- **Dependencies:** Apple App Store Server API, Google Play Developer API, Stripe, Prisma.
- **State owned:** `subscriptions`.
- **Edge cases:** Refund / chargeback → status→`revoked`, `is_premium=false`, premium features disabled mid-session via JWT short TTL; cross-platform downgrade (user buys on Android then logs in on iOS — single subscription per user enforced); grace period (Apple 16-day, Google 3-day) honoured.

---

## modules/media

- **Purpose:** Photo upload pipeline + face-blur stub + CSAM hash check + pHash dedupe.
- **Sub-components:** `routes.ts`, `presign.ts`, `faceBlur.ts` (Sharp), `phash.ts`, `csam.ts` (PhotoDNA).
- **Interfaces:** `media.attachPhoto(userId, s3Key)`, `media.markBlurred(photoId)`.
- **Dependencies:** S3, Sharp, PhotoDNA/NCMEC, Prisma.
- **State owned:** `user_photos`.
- **Edge cases:** CSAM hash match → upload rejected, user suspended, NCMEC report triggered, action logged immutably; pHash duplicate of previously flagged image → reject; corrupt file → 422; over-size (> 10 MB) → 413.

---

## modules/admin

- **Purpose:** T&S dashboard back-end, RBAC, immutable audit log, user-management actions.
- **Sub-components:** `routes.ts`, `rbac.ts`, `audit.ts`, `dashboard.ts`.
- **Interfaces:** Internal only — not exposed to mobile clients.
- **Dependencies:** All modules (read-only) + `moderation_actions` writes; VPN allowlist middleware; FIDO2/TOTP MFA.
- **State owned:** `admin_users`, `moderation_actions`.
- **Edge cases:** Moderator attempts to read chat content via admin API → API returns 403 unconditionally (system-enforced — chat tables have no admin-readable plaintext column); admin account compromised → all sessions revoked via `auth.invalidateFamily` + audit alert.
