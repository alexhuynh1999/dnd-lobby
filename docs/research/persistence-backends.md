# Persistence backends for a static GitHub Pages frontend

Research for ticket **Research: persistence backends for a static GitHub Pages frontend** (issue #2, map: #1). This is *findings to react to*, not the decision: the decision is the blocker ticket [Decide: persistence backend for the v1 lobby app](https://github.com/alexhuynh1999/dnd-lobby/issues/3).

Access date for all sources: 2026-02-10 (approximately). Verify hard numbers at build time — free tiers change.

## The constraint set (from charting)

- Frontend is **static on GitHub Pages** — no server of our own; the backend must be a hosted cloud service.
- **No accounts.** Identity is a per-browser display name + a per-browser token; no cross-device merge (Q9 settled at charting).
- **Trusted group** — everyone who reaches the app is a friend, but the Pages URL is public.
- Shared container with **locally-triggered saves** — no realtime requirement in v1.
- Mobile-first consumers, near-zero ops, free tier, tiny data volumes (a handful of lobbies, each with a handful of sheets).

## Candidates

### A. Supabase (Postgres + Data API + Auth)

**Free tier facts** — source: [supabase.com/pricing](https://supabase.com/pricing)

- Unlimited API requests; 50,000 monthly active users; **500 MB database**; shared CPU; 5 GB egress; 1 GB file storage; community support; limit of 2 active projects.
- ⚠️ **Free projects are paused after 1 week of inactivity.** For a lobby app used on game nights — weekly or less, with long dormant stretches between campaigns — this boundary will be crossed repeatedly. Paused projects do not serve requests until restored (restore is a dashboard action, not something the app can do), so a game-night open after a quiet week is a manual chore for whoever owns the project.

**Identity mapping to Q9** — source: [supabase.com/docs/guides/auth/auth-anonymous](https://supabase.com/docs/guides/auth/auth-anonymous)

- Two distinct things: the **anon key** (public, in the client, uses the anonymous Postgres role and can grant open read/write if policies allow) and **anonymous sign-ins** (`signInAnonymously()`). The latter creates a real per-browser user with a uid, used with the `authenticated` role, carrying an `is_anonymous` JWT claim, and linkable to a real account later (e.g. if the group's trust model ever changes). That maps 1:1 onto Q9: the anonymous uid *is* the per-browser token; it survives within a browser, is lost if browsing data is cleared or on a new device.

**Access control** — Postgres **RLS**, deny-by-default on new projects (no policies = no access). Policies key off JWT claims (`auth.uid()`, `is_anonymous`, custom claims). Risk from a leaked anon key: with RLS properly keyed on anonymous *users*, the anon key alone grants nothing — policies still gate by uid.

**Migration/lock-in** — SQL (portable via `pg_dump`), Supabase is open-source and self-hostable (escape hatch), Postgres skills transfer. Least lock-in of the hosted trio.

**Setup** — Dashboard project, SQL policies; tables + RLS are trivial for this schema. Has a realtime layer (unused in v1, available for phase 2's DM overview without a rewrite).

### B. Firebase: Cloud Firestore + anonymous auth (Spark free plan)

**Free tier facts** — source: [firebase.google.com/docs/firestore/quotas](https://firebase.google.com/docs/firestore/quotas) and [firebase.google.com/pricing](https://firebase.google.com/pricing)

- Firestore free quota: **1 GiB stored, 50,000 document reads/day, 20,000 document writes/day, 20,000 document deletes/day, 10 GiB egress/month**; exactly **one free database per project**.
- **No idle pause.** Spark stays available; billing is only required to exceed quota. An every-few-weeks game-night usage pattern costs nothing and never pauses.
- Anonymous auth is part of Firebase Authentication, included on the free tier (the 50K-MAU limit applies to the paid Identity Platform upgrade; standard anonymous auth is free — verify current terms at build time).
- The free quota is huge for this app: a lobby app with a handful of users does hundreds of operations per session, not tens of thousands.

**Identity mapping to Q9** — [firebase.google.com/docs/auth](https://firebase.google.com/docs/auth): **anonymous auth** creates a temporary per-browser account with a uid; it can later be upgraded to a permanent account. Same 1:1 mapping onto Q9, same clear-browsing-data caveat as Supabase.

**Access control** — Firestore **security rules**, evaluated per request server-side. Rules keying on `request.auth.uid` (own-sheet writes, lobby-scoped reads via membership) are idiomatic and compact. Leaked client config (API key is public by design) grants nothing beyond what rules allow.

**Migration/lock-in** — document DB: the roster is a collection query, a sheet is a document; no joins. Vendors into Google Cloud; data export exists but schema/code is locked to Firestore. Rules engine has a learning curve but is small for this schema.

**Gotchas** — every read/write operation counts against the daily quota (a sheet save that touches several sub-documents counts several writes — still negligible at this scale); one free DB per project; document-model thinking differs from SQL.

### C. Cloudflare D1 + Pages Functions (the DIY one)

**Free tier facts** — source: [developers.cloudflare.com/d1/platform/pricing](https://developers.cloudflare.com/d1/platform/pricing)

- Workers free plan: **5M rows read/day, 100K rows written/day, 5 GB storage**, scale-to-zero (no cost when idle). Pages Functions run on the same worker platform (verify the free-plan request cap at build time).

**Identity mapping to Q9** — **none built in.** The database is reached via a shared service binding/token; there are no users. Per-browser identity means hand-rolling a tokens table, issuing tokens, and writing all authorization logic in the Worker. The Q9 model becomes your engineering problem.

**Access control** — everything is code you write. Leaked client credentials = whoever has them can do anything your Worker lets them.

**Migration** — vanilla SQLite, the most portable data; matches the "SQLite" instinct from charting (Q7), but the identity/authz DIY is the price.

### D. Also seen (one-liners)

- **Turso** (hosted libSQL) — free: 100 databases, 5 GB storage, 500 M rows read/mo, 10 M rows written/mo, 1-day point-in-time restore ([turso.tech/pricing](https://turso.tech/pricing)). HTTP API, shared API token, no per-user identity — same DIY caveat as D1.
- **JSONBin / JSON storage boxes** — public-bin JSON storage with tiny free tiers and no access control worth the name; fine for demo data, not for multi-user append/update traffic. Not a real contender.

## Comparison

| Axis | Supabase | Firebase (Firestore) | Cloudflare D1 + Pages Functions |
|---|---|---|---|
| Free tier | 50K MAU, 500 MB, **pauses after 1 wk idle**, 2-project cap | 1 GiB + 50K reads/20K writes/day, **no pause** | 5M rows read/100K writes/day, 5 GB, no idle cost |
| Q9 identity mapping | `signInAnonymously()` → uid | Anonymous auth → uid | **None** (DIY tokens) |
| Per-user access control | RLS, deny-by-default | Security rules per request | Hand-written authz |
| Leaked client credential | grants only what RLS allows | grants only what rules allow | full access your code permits |
| Schema model | SQL (relational) | documents | SQLite (relational) |
| Lock-in | least (open source, `pg_dump`) | most (Google Cloud) | middle (SQLite, but Cloudflare platform) |
| Setup effort | low–medium (SQL policies) | medium (console + rules) | high (identity + authz are DIY) |

## Recommendation to react to (not the decision)

Both **Supabase** and **Firebase** map 1:1 onto the settled identity model and both are free at this scale. The sharpest differentiator for *this* app is the **Supabase free-tier idle pause vs Firestore's always-on Spark**: a lobby used on game nights, with long dormant stretches, will keep hitting the one-week pause and each time needs a manual dashboard restore before a session. Firestore's Spark also keeps anonymous auth free with no keep-alive obligations.

Firebase wins on ops quietness; Supabase wins if you'd rather have Postgres/RLS with an open-source escape hatch and can tolerate (or periodically log in to) the pause, or would rather a realtime path for phase 2's DM overview without Firestore's document model.

So: **Firebase (Cloud Firestore + anonymous auth) as the primary take**, **Supabase as the strong alternate** — the deciding factors for [Decide: persistence backend for the v1 lobby app](https://github.com/alexhuynh1999/dnd-lobby/issues/3) should be: tolerance for Supabase's idle pause, document-vs-SQL schema preference, and lock-in stance (open-source Supabase vs Google Cloud).

## Sources

- Supabase pricing (free tier + pause): https://supabase.com/pricing
- Supabase anonymous sign-ins: https://supabase.com/docs/guides/auth/auth-anonymous
- Firebase Firestore quotas (free quota table): https://firebase.google.com/docs/firestore/quotas
- Firebase pricing (Auth / Firestore Spark): https://firebase.google.com/pricing
- Firebase Authentication (anonymous auth): https://firebase.google.com/docs/auth
- Cloudflare D1 pricing: https://developers.cloudflare.com/d1/platform/pricing/
- Turso pricing: https://turso.tech/pricing