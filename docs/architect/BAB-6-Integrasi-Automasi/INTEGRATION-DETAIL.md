# INTEGRATION-DETAIL — BAB-6 Integrasi & Automasi (Recovered)

> ⚠️ RECOVERED dari kode — 7 elemen interface per integrasi (content · material · medium · frequency · volume · trigger · protocol). Yang tidak teramati ditulis eksplisit.

## Index Chapter Ini

| # | Integrasi | Direction | SDK | Trigger utama |
|---|-----------|-----------|-----|---------------|
| 1 | YouTube Data API / scraping | outbound | **tanpa SDK** (fetch) | cron 2 menit ×2 |
| 2 | Discord widget (publik) | outbound | tanpa SDK | per render (cache 60s) |
| 3 | RSS/Atom feed | outbound | `rss-parser` | cron 15 menit |
| 4 | Webhook keluar | outbound | tanpa SDK (HMAC manual) | queue ±10 detik |
| 5 | GitHub Releases | outbound | tanpa SDK | startup + cron 6 jam |
| 6 | OG image fetch (SSRF-catatan) | outbound | fetch | on-demand render |
| 7 | YouTube live badge / scheduled rules | (konsumen #1) | — | cron 2 menit |

---

## 1. YouTube (live status + API key opsional)

| Elemen | Teramati | Evidence |
|--------|----------|----------|
| Content | status live video channel (`live`/`offline`/`unknown`), playlist uploads, channelId | `lib/youtube-live.ts:73,81-104` |
| Physical/material | — (layanan API SaaS Google) | — |
| Medium | HTTPS `fetch` | `:115,131` |
| Frequency | **2 menit** (scheduled rules) + **2 menit** (live badge) = 2 cron terpisah | `instrumentation.ts:58,67` |
| Volume | kuota: komentar `:111-114` — 3 panggilan murah ≈2 unit/cek, ≈2.900 unit/hari dari kuota 10.000/hari (estimasi **di komentar kode**, fakta dokumentasi) | `lib/youtube-live.ts:111-114` |
| Trigger | cron; konsumen `processScheduledRules` (`scheduled-rules.ts:51-72`) | `instrumentation.ts:58` |
| Protocol | (a) **tanpa key**: scraping `https://www.youtube.com/<ch>/live` + UA browser, parse `ytInitialPlayerResponse` (`:31-41,42-71,226-230`); (b) **dengan key**: `https://www.googleapis.com/youtube/v3` (`channels`,`playlistItems`,`videos`) key via query param `key` (`:115,130-131`) | idem |
| API key | disimpan **ter-encrypt di DB** (`users.youtubeApiKey`, purpose `youtube-api-key`) — **bukan env var**; validasi 1 panggilan saat simpan (`:193-203`); tanpa key → fallback scraping (`:226-230`) | `lib/youtube-api-key.ts:6,15-27` |
| Fallback akurasi | `unknown` → `null` → rule tidak mengubah state (`scheduled-rules.ts:58`) | idem |
| Env key | **tidak ada** | scan negatif |
| Volume data | channelIdCache in-process (`:145`) | idem |

## 2. Discord Widget

| Elemen | Teramati | Evidence |
|--------|----------|----------|
| Content | nama guild, member count, presence, invite, voice channels (camelCase-mapped) | `lib/discord-widget.ts:47-62` |
| Medium | HTTPS fetch | `:5` |
| Frequency | per render halaman/widget + form preview; **revalidate 60 detik** | `:34` |
| Volume | retry max 2× (delay 400ms), 404 stop | `:31-43` |
| Trigger | client component `DiscordWidgetLinkCard` → `/api/discord-widget-config` | `discord-widget-config/route.ts:5-10` |
| Protocol | `https://discord.com/api/guilds/{guildId}/widget.json` — **endpoint publik, tanpa bot token/OAuth** (komentar `:1-4`) | `:5` |
| Auth ke Discord | tidak perlu | idem |

## 3. Content Feeds (RSS/Atom)

| Elemen | Teramati | Evidence |
|--------|----------|----------|
| Content | item feed → jadi `links` baru (`pageId,groupId,title,url,description,thumbnail`) | `lib/content-feeds.ts:101-112` |
| Medium | HTTPS via `rss-parser` | `:1,71` |
| Frequency | **15 menit** (`*/15 * * * *`) | `instrumentation.ts:81-83` |
| Volume | **max 5 item baru/run** (`MAX_NEW_ITEMS_PER_RUN`) | `content-feeds.ts:11` |
| Trigger | cron; run pertama = baseline saja (tidak backfill) `:77-83` | idem |
| Protocol | `parser.parseURL` (HTTP GET feed), timeout 8 detik `:20-29` | idem |
| Side effect | `logActivity(..., "link_created", ..., "automation")` `:113` | idem |

## 4. Webhook Keluar (outbound)

| Elemen | Teramati | Evidence |
|--------|----------|----------|
| Content | payload JSON per event | `lib/webhooks.ts:10-15` |
| Material | secret per-webhook di kolom DB (HMAC) | `:17-19` |
| Medium | HTTPS POST | `:26-35` |
| Frequency | pemrosesan queue **±10 detik**; kirim tergantung event | `instrumentation.ts:50` |
| Volume | retry **max 5** dengan backoff `[10s,30s,60s,5m,30m]` | `webhooks.ts:6-8` |
| Trigger | `enqueueWebhookEvent` — saat ini **hanya event `test.ping`** dari dashboard (`integrations-actions.ts:66-70`); enum `subscriber.new`/`link.clicked`/`page.live_status_changed` (`schema.ts:205-207`) **belum dipanggil** | scan |
| Protocol | HTTP POST, header `X-KitabLink-Signature` = HMAC-SHA256(body, secret) hex + `X-KitabLink-Event`, timeout 10 detik | `webhooks.ts:17-19,26-35` |
| Status tracking | `webhook_queue`: `sent`/`failed` + `nextRetryAt` | `:37-49` |

## 5. GitHub Releases (update check)

| Elemen | Teramati | Evidence |
|--------|----------|----------|
| Content | `tag_name`, `name`, `url`, `publishedAt` → kolom `users.latestReleaseJson` | `lib/update-check.ts:22-25` |
| Medium | HTTPS fetch | `:14-16` |
| Frequency | sekali saat startup + **6 jam** | `instrumentation.ts:74-77` |
| Trigger | cron/startup; UI tawarkan link update (`getAvailableUpdate` `:33-42`) | idem |
| Protocol | `https://api.github.com/repos/Kurohiko-id/Kitab-Link-Self-Hosted/releases/latest`, header `Accept: application/vnd.github+json`, **tanpa token** | `:5,14-16` |
| Persistensi | simpan di DB bukan variable module (komentar `:9-11`: masalah chunk standalone Next.js) | idem |

## 6. OG Image Fetch

| Elemen | Teramati | Evidence |
|--------|----------|----------|
| Content | buffer gambar dari URL link | analyst `security-report.md` A1 |
| Trigger | on-demand saat render OG | `app/[slug]/og` |
| Protocol | `fetch` URL bebas — **tanpa validasi host/internal-range** (SSRF potensial, admin-only input) | `security-report.md` A1 |

## 7. Dead Link Checker

| Elemen | Teramati | Evidence |
|--------|----------|----------|
| Content | status `ok`/`dead` per link (`consecutiveFailures >= 2`) | `lib/dead-links.ts:33-39` |
| Frequency | **harian 03:00** | `instrumentation.ts:87-89` |
| Volume | timeout 8 detik/check, UA `KitabLinkBot/1.0`, hanya `linkType="url"` aktif | `dead-links.ts:5,8-9,26-29` |
| Protocol | HTTP HEAD, fallback GET saat 405/403 | `:16-19` |

---

## Tidak Teramati di Kode

- **Webhook INBOUND** (penerima event dari pihak ketiga): tidak teramati.
- SDK resmi (discord.js, googleapis, @aws-sdk): tidak dipakai — semua raw `fetch` (kecuali `rss-parser`).
- Retry/timeout YouTube API: `FETCH_TIMEOUT_MS = 10_000` (`youtube-live.ts:1`) — retry khusus YouTube tidak teramati.
- Volume trafik aktual / SLA integrasi: tidak teramati.
- Persisnya kapan event `subscriber.new`/`link.clicked` akan dipicu: **belum ada pemanggil** (i18n `lib/i18n.ts:373` mengonfirmasi "baru terpicu otomatis kalau fitur sumbernya sudah ada").
