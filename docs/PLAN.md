# Smart Links — link.khabeer.online

A content hub that sits between Khabeer and the audience. Every video gets one "Drop" page with all its assets (links, prompts, files, related videos). The system also connects the audience back to us through requests, questions, notifications and rewards.

Arabic-first (RTL), mobile-first, YouTube-first. Managed from an admin panel and through a secure API/MCP server.

---

## 1. Core concepts

| Concept | Description |
|---|---|
| **Drop** | One page per video, e.g. `link.khabeer.online/47`. Shows the title, thumbnail, key info, assets, related videos and the featured slot |
| **Asset** | A reusable item: link, prompt, file, tool, video. One asset can appear in many drops. Assets are versioned |
| **Topic** | A hierarchical tag (الذكاء الاصطناعي ← كتابة المحتوى ← البرومبتات) |
| **Path** | An ordered learning path made of drops, with progress tracking |
| **Featured** | A sponsor-ready slot: scheduled, targeted by topic/drop, labeled «برعاية/إعلان» |
| **Announcement** | A global or topic-specific banner: scheduled and dismissible |
| **My Links** | The footer on every page (channels, website, contact) |
| **Gateway** | Requests (asset/topic/link), questions, notify-me, rewards |

## 2. Public pages

- `/` is the hub: announcements, latest drops, search box ("enter the video number"), paths, filters.
- `/{code}` is a drop: header (thumbnail, title, date, duration, YouTube button), featured slot, assets (copy/download/open), "watch next", notify me, ask a question, my links.
- `/t/{topic}` is a topic page (SEO-friendly).
- `/p/{path}` is a learning path with progress.
- `/library` holds the prompt library and toolbox.
- `/ask` is for requests and questions, with public status (قيد الدراسة / مخطط / تم).
- `/me` (optional magic-link login) shows history, saved items, points and rewards.

Filters: topic, level, asset type, free/gated, date. Arabic search normalizes أ/إ/آ, ة/ه, ى/ي and diacritics.

## 3. Smart asset finding (built right, runs in the background)

Goal: **the same asset is never stored twice, and the right asset is always one keystroke away.**

1. **Hybrid search** in Postgres:
   - Arabic normalization function plus `tsvector` full-text search
   - `pg_trgm` fuzzy matching (typos, partial words)
   - `pgvector` embeddings for meaning-based search ("برومبت لكتابة سكربت" finds "Script writing prompt")
   - The results are combined into one ranked score.
2. **Background indexing:** on insert or update, a job queue (Postgres-based) computes the normalized text and embedding, fetches link metadata (title, favicon, OG image) and checks link health.
3. **Deduplication:** URLs are canonicalized (tracking params stripped, trailing slashes and www removed). Prompts are fingerprinted (hash plus similarity). The admin sees "this already exists in 4 drops: reuse it?"
4. **Auto-suggest while building a drop:** paste a transcript or notes, and the system suggests existing assets, topics and related drops.
5. **Command palette in the admin panel** (Ctrl+K): search everything from anywhere.
6. **Daily link-health checks** for broken and redirected links, with alerts.

## 4. Gateway features (audience ↔ us)

| Feature | Notes |
|---|---|
| **Notify me** | For a topic, path, or "any new drop". Email goes to Mailercloud with tags. Web push and WhatsApp come later |
| **Request** | Request an asset, topic or link. Requests can be upvoted and deduplicated by similarity. Status is public, and requesters are notified when a request is fulfilled (this closes the loop) |
| **Ask a question** | Questions are tied to a drop and moderated. The best answers can be published as FAQ on the drop or turned into the next video |
| **Zero-result searches** | Logged as content ideas |

## 5. Gamification (phased, abuse-resistant)

- Points for meaningful actions only: completing a path step, a request we accept, an asked question that gets answered. Views and clicks don't count, so the points can't be farmed.
- Streaks, path-completion badges and certificates.
- **Rewards:** unlock gated packs, early access, shout-outs, a monthly prize draw.
- History and points require a magic-link login. Anonymous progress is kept on the device and merged on login.
- Rate limits and Turnstile on forms. Rewards are granted server-side only.

## 6. Leads (Mailercloud)

Each asset has a capture level:
- **Open:** no capture.
- **Soft:** "send it to my email" (optional; this also solves broken downloads in in-app browsers).
- **Gated:** email required.

Leads are synced with `POST https://cloudapi.mailercloud.com/v1/contacts/upsert` (header `Authorization: <key>`), sending `list_id`, `tags[]` and `custom_fields`:
`source_drop`, `topic`, `path`, `platform`, `asset`, `consent_at`.

- Every lead is stored locally first, and sync goes through a retry queue. A Mailercloud outage never loses a lead.
- Consent: an explicit checkbox, and we store the consent text version plus a timestamp (PDPL/GDPR). Unsubscribe status is synced back through Mailercloud webhooks.

## 7. Switchy (kept as the tracking/short-link layer)

- The API is GraphQL `https://graphql.switchy.io/v1/graphql` (reads) plus REST `POST https://api.switchy.io/v1/links/create`, with header `Api-Authorization: <token>`.
- Limits: 1,000 links/hour and 10,000/day.
- **A custom domain is required for API-created links** (`hi.switchy.io` and `swiy.io` are restricted to official integrations).
- For each drop, one Switchy link is created per platform (yt / ig / tt / x / li …). Each points to `link.khabeer.online/{code}?utm_source=…`.
- QR codes come from Switchy. Click stats are pulled into our dashboard on a schedule.
- Our system is the source of truth. Switchy IDs are stored on the drop.

## 8. Analytics

- First-party, cookieless event log: `view`, `asset_open`, `copy`, `download`, `search`, `search_zero`, `notify`, `request`, `question`, `lead`, `featured_impression`, `featured_click`.
- Anonymous visitor ID (rotating hash) plus platform from UTM, device and country (from the edge header, no raw IP stored).
- Dashboards:
  - top drops, assets and topics
  - platform → lead conversion
  - path drop-off
  - zero-result searches
  - sponsor reports (impressions/CTR per featured placement)
- Switchy clicks merged in.

## 9. Architecture

- **App:** Next.js (App Router, TypeScript), RTL. Public pages are statically generated and revalidated on change, so they stay fast and keep serving even if the database has a hiccup.
- **DB:** the existing self-hosted Supabase on the VPS (Coolify), in a **dedicated `smartlinks` schema** with its own DB role. It is not in `public` and has no cross-schema access. RLS is on, and public reads go through views or server code only. The schema can be moved to its own instance later with `pg_dump -n smartlinks`.
- **Storage:** Supabase Storage bucket `smartlinks-files` with signed URLs. Large or popular files may move to Cloudflare R2 later.
- **Jobs:** Postgres queue (`pgmq`/`graphile-worker`) for embeddings, metadata, link health, Mailercloud and Switchy sync.
- **Hosting:** Coolify on the Hostinger VPS (KVM 2: 2 vCPU, 8 GB RAM). Cloudflare in front (DNS, cache, WAF, Turnstile).
- **API:** REST `/api/v1/*` (OpenAPI documented) plus an **MCP server** (streamable HTTP) exposing the same operations.

## 10. API / MCP security

- API keys are random, **stored hashed**, shown once, with **scopes**: `read`, `drops:write`, `assets:write`, `files:write`, `leads:read`, `admin`. They can be revoked, expire, and record when they were last used.
- MCP uses the same keys and scopes. Destructive tools (delete/publish) need the matching scope and support a `dry_run`.
- Every write is recorded in an **audit log** (who, what, before/after).
- Rate limits per key and per IP. Input is validated with zod. Uploads are checked for MIME type and size, and stored with no execution.
- Admin login uses Supabase Auth with 2FA, and team roles (owner/editor/viewer).

Initial MCP tools: `search_assets`, `get_drop`, `list_drops`, `create_drop`, `update_drop`, `add_asset`, `attach_asset`, `upload_file`, `create_switchy_links`, `get_stats`, `list_requests`, `answer_question`, `create_announcement`.

## 11. Reliability and operations

- Daily encrypted `pg_dump` of the schema plus storage, copied **off the VPS** (e.g. R2/B2). Restore is tested monthly.
- Uptime monitoring (Uptime Kuma on Coolify or an external pinger), plus error tracking (Sentry or self-hosted GlitchTip).
- Staging environment on Coolify. Migrations live in the repo (`supabase/migrations`).
- Secrets live **only** in Coolify environment variables, never in the repo.
- QR codes and short links point to stable URLs, so even a changed asset never breaks a printed or published QR.

## 12. Seeding

Sources: the existing DB/Excel of past videos and/or a YouTube Data API export (title, description, thumbnail, duration, publish date, chapters).
Import pipeline:
1. Map columns.
2. Assign codes (keep existing numbers if videos already show them).
3. Extract links from descriptions into assets, deduplicated.
4. Auto-tag.
5. Review queue in the admin panel.
6. Publish.

## 13. Compliance

- Sponsored content is always labeled «برعاية» / «إعلان» (FTC, Saudi GCAM influencer ad rules, and each platform's branded-content rules).
- No cloaking and no redirect chains beyond Switchy → page → asset. Destinations are visible before opening.
- Instagram/Facebook DMs only through the official Meta API, within the 24h window (phase 3).
- Privacy policy and terms pages, a cookie-less default, and data export/delete requests honored.

## 14. Phases

1. **Foundation:** schema, admin auth, drops, assets, topics, hub and drop pages, Switchy integration, Mailercloud leads, API and MCP (scoped keys plus audit), seeding import, backups.
2. **Smart:** hybrid search and dedup, background jobs, auto-suggest, paths, notify me, requests and questions, analytics dashboard, featured slot and announcements.
3. **Engagement:** gamification and rewards, user history, web push/WhatsApp, IG/TikTok comment-to-DM, sponsor reports.

## 15. Open items

- Switchy custom domain name, and an API token (to be stored in Coolify env).
- Mailercloud: list strategy (one list with tags is recommended) and API key.
- Seed data file location / DB access.
- Where khabeer.online DNS is managed (Hostinger or Cloudflare?).
- Repo is currently **public**. Decide whether to keep it public (fine, as long as there are never secrets in it) or make it private.
