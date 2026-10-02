# API-ENDPOINTS — BAB-6 Integrasi & Automasi (Recovered)

> ⚠️ RECOVERED dari kode — daftar endpoint lengkap chapter ini. Konvensi global: `00-Global/API-STANDARD.md`.

## 1. Endpoint Terautentikasi (Bearer `klk_`)

| # | Method | Path | Scope | Response success | Evidence |
|---|--------|------|-------|------------------|----------|
| 1 | GET | `/api/v1/links?pageId=` | `links:read` | `{ links: [{id,title,url,isActive,displayStyle}] }` | `app/api/v1/links/route.ts:7,14,31-39` |
| 2 | GET | `/api/v1/links/:id` | `links:read` | objek link datar | `app/api/v1/links/[id]/route.ts:10,17,32` |
| 3 | PATCH | `/api/v1/links/:id` | `links:write` | `{ id, isActive }` (body `{active?}`; kosong = toggle) | `[id]/route.ts:37,44,59-70` |
| 4 | GET | `/api/v1/discord-widgets/:id` | `discord:read` | `{id,name,isEnabled}` | `discord-widgets/[id]/route.ts:9,16,23-31` |
| 5 | PATCH | `/api/v1/discord-widgets/:id` | `discord:write` | `{ id, isEnabled }` (body `{enabled?}`; kosong = toggle) | `discord-widgets/[id]/route.ts:35,42,57-68` |

**Pola auth identik di semua 5 handler:** Bearer prefix → `verifyApiToken` (SHA-256 lookup) → scope check (403) → ownership `innerJoin pages` (404) — `lib/auth/api-tokens.ts:33-42`, `links/route.ts:23-28`.

**Catatan fakta:**
- **Tidak ada POST/DELETE** di API v1 — create/delete link hanya via server action dashboard (`app/api/v1/links/route.ts:7` GET saja; `actions.ts:245-248`).
- Scope `discord:*` **tidak bisa dibuat via UI** (whitelist `integrations-actions.ts:12` cuma `links:*`) → endpoint 4 & 5 selalu 403 untuk token buatan UI (R-07, Contradictions #2).
- PATCH sukses menulis `logActivity` dengan source `"api"` — `[id]/route.ts:67-68`.
- Use-case eksplisit: "polling status dari luar, mis. Stream Deck" (komentar `[id]/route.ts:8,35-36`).
- **Rate limit per-token: tidak teramati.**

## 2. Endpoint Publik / Internal

| # | Method | Path | Auth | Response | Evidence |
|---|--------|------|------|----------|----------|
| 6 | GET | `/api/discord-widget-config?widgetId=` | **tanpa auth** | `{config}` (iframe) / `{config,data}` (lainnya) | `discord-widget-config/route.ts:11,37-42` |
| 7 | GET | `/api/discord-widget?guildId=` | **tanpa auth** | data Discord langsung (proxy anti-CORS, cache 60s) | `discord-widget/route.ts:7,16`; komentar `:4-6` |
| 8 | GET | `/api/health` | **tanpa auth** | `{status:"ok"}` / 503 `{status:"error"}` | `health/route.ts:7-13` |

## 3. Route Publik non-API (untuk kelengkapan chapter)

| Path | Fungsi | Evidence |
|------|--------|----------|
| `/{slug}`, `/{slug}/og`, `/{slug}/privacy` | halaman publik + OG image + privacy | `app/[slug]/` |
| `/r/{linkId}` | redirect + record click | `app/r/[linkId]/route.ts:26` |
| `/uploads/*` | file serving (whitelist MIME, immutable cache) | `app/uploads/[...path]/route.ts` |

## 4. Interface Attributes (frequency/volume/trigger — Teramati)

| Endpoint | Frequency | Trigger | Volume teramati |
|----------|-----------|---------|-----------------|
| 1–3 | on-demand | klien eksternal (Stream Deck dll) | tanpa pagination → semua rows per page |
| 4–5 | on-demand | klien eksternal | idem |
| 6–7 | per render widget | client component publik | Discord fetch cache 60s |
| 8 | 30 detik sekali | Docker HEALTHCHECK | `Dockerfile:74-75` |

- Data exchanged: link metadata (id/title/url/isActive/displayStyle) & status widget Discord — lihat kolom response di atas.
- Protocol: HTTPS (di belakang reverse proxy), JSON over REST, header `Authorization: Bearer`.
- Volume/frequency klien eksternal: **tidak teramati di kode** (tergantung pengguna luar).

## Tidak Teramati di Kode

- OpenAPI/Swagger spec: tidak ada.
- Webhook **inbound** (seperti Stripe/GitHub webhook receiver): tidak teramati — webhook di chapter ini hanya OUTBOUND (lihat `INTEGRATION-DETAIL.md`).
- Endpoint create link via API: tidak ada.
