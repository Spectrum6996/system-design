# Risk Register

| # | Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- | --- |
| 1 | Single-replica Railway monolith outage cascades to all features (chat, match, auth) | Medium | High | Auto-restart on failure; BetterUptime + PagerDuty; deploy multi-replica at first sign of WS or CPU pressure; rollback automation | DevOps Lead |
| 2 | Automated account creation / bot sign-ups via Google OAuth | Low | Medium | Cloudflare Turnstile on `/auth/google`; per-IP rate limits in Upstash; Google's own OAuth abuse detection | Security Eng |
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
| 14 | Geohash precision 7 still narrow enough to fingerprint individuals in low-density areas | Low | Medium | Distance shown as bucketed ranges; profiles in low-density geohashes get reduced precision (prefix 5) before display | Privacy Eng |
| 15 | App Store rejection of LGBTQ+ content in restrictive markets | Medium | High | Launch markets pre-cleared (US, UK, India); legal sign-off per region; content declaration accurate; have backup web app channel via Stripe | Product Lead |
