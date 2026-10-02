# System Arch Master (Recovered from Code): Kitab Link Self-Hosted

> ⚠️ RECOVERED — hasil observasi kode pada 2026-10-02. Diagram = representasi struktur teramati, bukan blueprint yang divalidasi manusia.

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 0.9.0 |
| **Last Updated** | 2026-10-02 |
| **Status** | Recovered |
| **Source** | `Kitab-Link-Self-Hosted/` |
| **Depth** | deep |

---

## 1. Big Picture (Teramati)

```
                        ┌─────────────────────────────────────────────────────┐
                        │              VPS / HOST (1 container)              │
                        │                 node:22-slim (runner)              │
┌──────────────┐        │  ┌───────────────────────────────────────────────┐  │
│  Pengunjung  │        │  │  NODE PROCESS (tunggal)                      │  │
│  (browser)   │──HTTP──┼─▶│                                               │  │
└──────────────┘        │  │  ┌─────────────┐   ┌────────────────────────┐ │  │
┌──────────────┐        │  │  │  proxy.ts   │   │  Next.js 16 App Router │ │  │
│  Operator    │──HTTP──┼─▶│  │  (Next 16   │──▶│  ├─ Server Components  │ │  │
│  (dashboard) │        │  │  │   request   │   │  ├─ Route Handlers     │ │  │
└──────────────┘        │  │  │   hook)     │   │  └─ Server Actions     │ │  │
┌──────────────┐        │  │  └──────┬──────┘   └───────────┬────────────┘ │  │
│ API consumer │──HTTP──┼─▶│         │ host resolution      │              │  │
│ (Bearer klk_)│        │  │         ▼                      ▼              │  │
└──────────────┘        │  │  ┌────────────┐  ┌──────────────────────────┐ │  │
┌──────────────┐        │  │  │ custom     │  │  instrumentation.ts      │ │  │
│  Cloud       │◀──GET──┼──│  │ domain →   │  │  ├─ migrate() saat boot  │ │  │
│  eksternal   │        │  │  │ /{slug}    │  │  └─ node-cron 6 job      │ │  │
│  (Discord,   │        │  │  └────────────┘  └──────────┬───────────────┘ │  │
│  YouTube,    │◀───────┼──│                             │                 │  │
│  GitHub,     │        │  └──────────────┬──────────────┼─────────────────┘  │
│  RSS feeds)  │        │                 │              │                    │
└──────────────┘        │                 ▼              ▼                    │
                        │        ┌─────────────────────────────────┐          │
                        │        │  SQLite (better-sqlite3, WAL)   │          │
                        │        │  /app/data/kitab-link.db        │          │
                        │        │  + /app/data/uploads/ (files)   │          │
                        │        │  + /app/data/.secret (master)   │          │
                        │        └─────────────────────────────────┘          │
                        │           1 NAMED VOLUME (docker-compose)           │
                        └─────────────────────────────────────────────────────┘
```

**Fakta kunci:** SATU proses Node menjalankan web server + cron + migrasi + queue (semua di process yang sama) — `instrumentation.ts:50-89`, `Dockerfile:80` (`CMD ["node", "server.js"]`).

---

## 2. Request Flow (Teramati)

### 2a. Halaman publik `/{slug}` (termasuk custom domain)

```
Request ──▶ proxy.ts (Next 16 request hook, Node runtime)
              │  proxy.ts:29-31   ambil header Host
              │  proxy.ts:33-37   query: pages WHERE domainType='custom_domain'
              │                    AND domainValue = host LIMIT 1
              │  proxy.ts:42-44   host = custom domain + path admin (/dashboard,
              │                    /login, /setup, /api) → 404 Not Found
              │  proxy.ts:49-51   belum verified → auto-set domainVerified=true
              │  proxy.ts:53-58   "/" → "/{slug}", "/privacy" → "/{slug}/privacy"
              ▼
app/[slug]/page.tsx  (Server Component, dynamic="force-dynamic" :12)
              │  app/[slug]/page.tsx:44  select page by slug LIMIT 1
              │  components/public-page-body.tsx:74-79  4 query PARALEL
              │    (board, theme, live status, discord widgets)
              │  components/public-page-body.tsx:68-72  recordPageView()
              │    (fire-and-forget INSERT, lib/db/analytics.ts:10-14)
              ▼
Response HTML  +  <style> custom font/CSS (public-page-body.tsx:87,97,100)
```

- Matcher proxy: `proxy.ts:61-66` — kecualikan `_next/static`, `_next/image`, `favicon.ico`, `robots.txt`, `uploads`, `r/`.
- **Tanpa cache halaman**: `generateStaticParams` = 0 match, `unstable_cache` = 0 match; semua halaman utama `dynamic = "force-dynamic"` (`app/[slug]/page.tsx:12`, `app/page.tsx:11`, `app/login/page.tsx:15`).
- Redirect click: `app/r/[linkId]/route.ts` → `recordLinkClick()` (`:26`) lalu redirect.

### 2b. Dashboard (operator)

```
/login ──▶ server action ──▶ argon2id verify (lib/auth/password.ts)
              │                └─ TOTP? (pending-2fa) ──▶ set cookie kitab_session
              ▼                                              (HMAC, 7 hari, session.ts:6,14-20)
/dashboard ──▶ requireSession() (lib/auth/require-session.ts:6-8 → redirect /login kalau kosong)
              │  per-tab fetch dibatasi (app/dashboard/page.tsx:207-253:
              │  activity log hanya di-load kalau activeTab cocok :253)
              ▼
Server Action (64 fungsi, "use server")
   ──▶ requireOwnedPage(pageId)  (lib/db/pages.ts:59-71, throw kalau bukan milik)
   ──▶ operasi Drizzle ──▶ revalidatePath("/[slug]", "page") + logActivity()
```

### 2c. API v1 (Bearer token)

```
Authorization: Bearer klk_<48hex>
   │
   ▼  lib/auth/api-tokens.ts:33  prefix "klk_" check
      lib/auth/api-tokens.ts:38  lookup WHERE tokenHash = SHA-256(raw)  (:10-12)
   ▼
scope check (per-endpoint):
   links:read        → GET  /api/v1/links           (route.ts:14)
   links:read|write  → GET/PATCH /api/v1/links/:id  ([id]/route.ts:17,44)
   discord:read|write→ GET/PATCH /api/v1/discord-widgets/:id (:16,42)
   ▼
ownership: innerJoin pages + pages.userId = identity.userId
   (app/api/v1/links/route.ts:23-28)
```

⚠️ KONTRADIKSI terkait scope — lihat `extraction-report-arch.md` § Contradictions.

---

## 3. Backend Structure (Teramati)

| Bagian | Jumlah / Lokasi | Evidence |
|--------|-----------------|----------|
| Server Actions | 64 (dari extraction-report analyst) — 12 file `app/**/actions.ts` | `app/dashboard/actions.ts` dll, semua `"use server"` baris 1 |
| Route Handlers | 6 file route API | glob `app/api/**/route.ts` |
| Public routes (non-API) | `/{slug}`, `/{slug}/og`, `/{slug}/privacy`, `/r/{linkId}`, `/uploads/*`, `/sitemap` | `app/[slug]/`, `app/r/`, `app/uploads/`, `app/sitemap.ts` |
| Request hook | `proxy.ts` (Next 16, pengganti middleware) | `proxy.ts:6-8` |
| Cron | 6 job di `instrumentation.ts` | `:50,:58,:67,:75,:81,:87` |
| Migrasi | 36 file SQL, dijalankan saat boot | `drizzle/0000…0035`, `instrumentation.ts:15-19` |

### Cron matrix (Teramati)

| Job | Jadwal | Handler | Evidence |
|-----|--------|---------|----------|
| Webhook queue | `*/10 * * * * *` (10 detik) | `processWebhookQueue` | `instrumentation.ts:50-54` |
| Scheduled rules (YouTube live auto show/hide) | `*/2 * * * *` | `processScheduledRules` | `instrumentation.ts:58-62` |
| Live badges | `*/2 * * * *` | `processLiveBadges` | `instrumentation.ts:67-71` |
| Update check GitHub | `0 */6 * * *` (+ sekali saat startup) | `checkForUpdates` | `instrumentation.ts:74-77` |
| Content feeds (RSS) | `*/15 * * * *` | `processContentFeeds` | `instrumentation.ts:81-83` |
| Dead links | `0 3 * * *` | `checkDeadLinks` | `instrumentation.ts:87-89` |

Semua dibungkus `.catch` (evidence: tiap callback di atas). In-process, **satu instance saja yang aman** — multi-container = dobel eksekusi (lihat `TECH-RISK-MASTER.md`).

---

## 4. Frontend Architecture (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Router | App Router (Server Component default) | folder `app/`, `next.config.ts` |
| Client boundary | 54 file `"use client"` (baris 1, semua `.tsx`) | grep `^["']use client["']` |
| State mgmt | **Tanpa library** — React 19 `useActionState` + `useTransition` | `app/login/login-form.tsx:3,13-14`; grep zustand/redux/query = 0 di source |
| Form pattern | Server Actions sebagai form action + `revalidatePath` | `app/dashboard/theme-actions.ts:1` dll |
| UI lib | shadcn/ui (10 komponen `components/ui/*`), style `base-nova` | `components.json:2-4,15-21` |
| Drag & drop | @dnd-kit (board + social links) | `app/dashboard/board.tsx:16`, `components/social-links-manager.tsx:12-14` |
| Theme anti-FOUC | IIFE script baca `localStorage("kl-theme")` + `prefers-color-scheme` | `app/layout.tsx:24-30`, disuntik `:40` (`dangerouslySetInnerHTML`) |
| Rendering strategi | **Semua halaman utama force-dynamic, tanpa SSG/ISR** | `app/[slug]/page.tsx:12`, `app/page.tsx:11`; grep `generateStaticParams` = 0 |
| i18n | Custom dictionary `lib/i18n.ts` (bukan next-intl) | `lib/i18n.ts`, `lib/i18n-actions.ts` |
| Fonts | 10 Google Font `next/font/google`, preload off | `lib/fonts.ts:1-12,14-21` |

---

## 5. Deployment Architecture (Teramati — detail di INFRA-CLOUD-MASTER)

```
git tag v* ──▶ GitHub Actions release.yml
                  │ buildx multi-arch (amd64+arm64, QEMU)
                  ▼
            ghcr.io/kurohiko-id/kitab-link-self-hosted:{version,latest}
                  │
      ┌───────────┴─────────────┐
      ▼                         ▼
docker compose up -d      install.sh (curl | bash)
(image :latest,           auto-install Docker + Caddy
 1 named volume,          deteksi VPS existing
 port 3000)               (install.sh:6-10)
```

- Multi-stage build: deps → builder (`next build`) → runner (hanya `.next/standalone`) — `Dockerfile:13,25,45,62-64`.
- Container non-root (`USER node`, `Dockerfile:68`), healthcheck internal (`Dockerfile:74-75` → `GET /api/health`).

---

## 6. Inference Summary (dilabeli)

> ⚠️ INFERENSI — tidak ada ADR/dokumen desain di kode yang mengonfirmasi:
>
> - **Monolit sengaja dipilih** demi satu-container self-host (konsisten dengan komentar `Dockerfile:54-59` tentang satu volume + ownership, dan `instrumentation.ts:48-49` "cron proses webhook_queue ... jalan di proses Next.js yang sama (bukan service terpisah)" — **komentar ini fakta**, tapi alasan bisnisnya tidak ditulis).
> - **Tanpa halaman statis** kemungkinan karena konten public page selalu berubah (revalidatePath dipanggil dari 12+ action) — namun kenapa tidak pakai ISR+tag tidak terdokumentasi.

## Unknown (tidak bisa dipastikan dari kode)

- Perilaku concurrent multi-instance (2 container share 1 SQLite file) — tidak diuji/didefinisikan di kode.
- Apakah `proxy.ts` query ke SQLite per-request atau ada koneksi keep-alive di hook — teramati query dibuat per-invocation (`proxy.ts:33-37`), tapi pooling tidak teramati.

## References

- `proxy.ts` — host resolution & admin blocking
- `instrumentation.ts` — boot sequence + cron
- `next.config.ts` — standalone & build constraint
- `app/[slug]/page.tsx`, `components/public-page-body.tsx` — public render path
