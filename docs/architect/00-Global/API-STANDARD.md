# API Standard (Recovered from Code): Kitab Link Self-Hosted

> ⚠️ RECOVERED — konvensi API yang TERAMATI dari kode pada 2026-10-02, sample 6 route file.

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 0.9.0 |
| **Last Updated** | 2026-10-02 |
| **Status** | Recovered |
| **Depth** | deep |

---

## 1. Style & Versioning (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Style | **REST** (Next.js Route Handlers) — bukan GraphQL/tRPC (0 dep `@trpc`/`apollo`/`graphql`) | `package.json:14-37`, 6 file `app/api/**/route.ts` |
| Versioning | **`/api/v1/`** prefix untuk API publik ber-token | `app/api/v1/links/`, `app/api/v1/discord-widgets/` |
| Non-versioned | endpoint internal/publik dashboard: `/api/health`, `/api/discord-widget*` | `app/api/health/route.ts` dll |
| Method | GET + PATCH saja di v1 (**tanpa POST/DELETE/create**) | `app/api/v1/links/route.ts:7` (GET saja), `[id]/route.ts:10,37` |
| Mutasi lain | via **Server Actions** (dashboard), bukan REST | `app/dashboard/actions.ts` (64 action) |

## 2. Response Envelope (Teramati — sample 6 endpoint)

**Temuan: envelope TIDAK konsisten antar endpoint.**

| Endpoint | Success shape | Evidence |
|----------|---------------|----------|
| `GET /api/v1/links` | wrapper **`{ links: [...] }`** | `app/api/v1/links/route.ts:31-39` |
| `GET /api/v1/links/:id` | **objek datar langsung** `{id,title,...}` | `[id]/route.ts:32` |
| `PATCH /api/v1/links/:id` | **objek datar** `{ id, isActive }` | `[id]/route.ts:70` |
| `GET /api/v1/discord-widgets/:id` | objek datar `{id,name,isEnabled}` | `discord-widgets/[id]/route.ts:23-31` |
| `GET /api/discord-widget-config` | **dua skema tergantung mode** `{config}` / `{config,data}` | `discord-widget-config/route.ts:37-42` |
| `GET /api/health` | `{ status: "ok" }` | `health/route.ts:10` |

**Error format: KONSISTEN** — `{ error: string }` + HTTP status di semua route:

| Status | Pesan (kutipan) | Evidence |
|--------|-----------------|----------|
| 401 | `"Missing bearer token"` / `"Invalid token"` | `links/route.ts:10,16` |
| 403 | `"Token tidak punya scope links:read"` | `links/route.ts:14-16` |
| 400 | `"Query param pageId wajib diisi"` | `links/route.ts:18-21` |
| 404 | `"Page tidak ditemukan"` | `links/route.ts:23-28` |
| 503 | `{ status: "error" }` (health) | `health/route.ts:13` |

> Tidak ada envelope `data:`/`meta:` seragam — **teramati sebagai inkonsistensi**, dicatat di `extraction-report-arch.md`.

## 3. Auth Contract API (Teramati)

```
Header:  Authorization: Bearer klk_<48 hex chars>
Verify:  lib/auth/api-tokens.ts:33  prefix check ("klk_")
         lib/auth/api-tokens.ts:38  WHERE tokenHash = SHA-256(raw)  (:10-12)
Scope:   dicek per endpoint, 403 kalau kurang
Owner:   innerJoin pages + pages.userId = identity.userId
```

| Endpoint | Method | Scope wajib | Evidence |
|----------|--------|-------------|----------|
| `/api/v1/links?pageId=` | GET | `links:read` | `links/route.ts:14` |
| `/api/v1/links/:id` | GET | `links:read` | `[id]/route.ts:17` |
| `/api/v1/links/:id` | PATCH | `links:write` | `[id]/route.ts:44` |
| `/api/v1/discord-widgets/:id` | GET | `discord:read` | `discord-widgets/[id]/route.ts:16` |
| `/api/v1/discord-widgets/:id` | PATCH | `discord:write` | `discord-widgets/[id]/route.ts:42` |
| `/api/discord-widget-config` | GET | **TANPA auth (publik)** | `discord-widget-config/route.ts:11` |
| `/api/discord-widget` | GET | **TANPA auth (publik)** | `discord-widget/route.ts:7` |
| `/api/health` | GET | **TANPA auth** | `health/route.ts:7` |

**Scope tersedia:** UI menawarkan 4 (`token-creator.tsx:17-20`), whitelist create cuma 2 — `VALID_SCOPES = ["links:read", "links:write"]` (`integrations-actions.ts:12`) → **kontradiksi, lihat report**.

## 4. Pagination / Filter / Sort (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Pagination API | **Tidak ada** — `GET /api/v1/links` return semua rows per pageId (tanpa limit/cursor) | `links/route.ts:31-39` |
| Filter | `pageId` wajib (query param) | `links/route.ts:18-21` |
| Sort | tidak ada param sort (urutan dari query) | tidak teramati |
| Aktivitas log | `limit = 100` server-side | `lib/db/activity-log.ts:47-53` |

## 5. Rate Limiting API (Teramati)

- Rate-limit **hanya untuk jalur auth** (login/reset/unlock/page access) — `lib/auth/unlock-rate-limit.ts`.
- **API v1 TANPA rate limit per-token** — tidak teramati di 6 route file.

## 6. Integration Index (global §9)

| Integrasi | Direction | Trigger | Chapter | Detail |
|-----------|-----------|---------|---------|--------|
| YouTube Data API / scraping | outbound | cron 2 menit ×2 | BAB-6 | `INTEGRATION-DETAIL.md` BAB-6 |
| Discord widget.json | outbound | render (revalidate 60s) | BAB-5/BAB-6 | idem |
| RSS/Atom feed | outbound | cron 15 menit | BAB-6 | idem |
| Webhook out (HMAC-SHA256) | outbound | queue 10 detik | BAB-6 | idem |
| GitHub Releases API | outbound | startup + cron 6 jam | global | idem |
| OG image fetch (SSRF-catatan) | outbound | on-demand | BAB-5 | `docs/analyst/security-report.md` A1 |

Interface lengkap per integrasi (data content/medium/frequency/volume/trigger/protocol) → `BAB-6-Integrasi-Automasi/INTEGRATION-DETAIL.md`.

## Inference Summary

> ⚠️ INFERENSI — ketidakkonsistenan envelope (`{links:[]}` vs objek datar) **kemungkinan** karena route dibuat bertahap tanpa konvensi awal; tidak ada comment/dokumen API style di kode. Alasan tidak terdokumentasi.

## References

- `app/api/**/route.ts` — 6 file
- `lib/auth/api-tokens.ts` — token contract
- `app/dashboard/token-creator.tsx` — scope UI
