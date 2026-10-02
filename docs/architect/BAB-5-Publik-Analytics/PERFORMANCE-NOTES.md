# PERFORMANCE-NOTES — BAB-5 Halaman Publik & Analytics (Recovered)

> ⚠️ RECOVERED dari kode — perilaku perf yang TERAMATI (bukan benchmark). ADR terkait: 001 (SQLite tanpa index), 004 (cron).

## 1. Halaman Publik `/{slug}` (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Rendering | **`dynamic = "force-dynamic"`** — tanpa SSG/ISR (0 `generateStaticParams`) | `app/[slug]/page.tsx:12` (komentar `:10-11`: "data page bisa berubah kapan saja, tidak boleh di-cache statis") |
| Query utama | 1 select page `.limit(1)` | `app/[slug]/page.tsx:44` |
| Query konten | **4 query paralel** `Promise.all` (board, theme, live status, discord widgets) | `components/public-page-body.tsx:74-79` |
| Invalidation | `revalidatePath("/[slug]", "page")` dari 12+ action | `theme-actions.ts:26,36,50,58,91`, `settings-actions.ts:111,123,133`, `page-actions.ts:59,96,108`, `backup-actions.ts:141` dll |
| Discord widget | Next fetch cache **60 detik** + retry 2× (delay 400ms) | `lib/discord-widget.ts:34,31-43` |
| File upload | `Cache-Control: public, max-age=31536000, immutable` | `app/uploads/[...path]/route.ts:44` |
| Analytics write | INSERT **fire-and-forget** (non-blocking sengaja) | `lib/db/analytics.ts:10-14` (komentar `:8-9`) |

## 2. Analytics Aggregation — Beban Saat Data Tumbuh (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Lookback | **366 hari** | `lib/db/analytics.ts:65` (`YEAR_LOOKBACK_MS`) |
| Query | **1 SELECT mentah, TANPA limit/pagination** — semua baris ke memori | `lib/db/analytics.ts:196-208` |
| Bucketing | dilakukan di **JS, bukan SQL** (komentar `:88-90`) | `:91-149` |
| Breakdown | sort + `slice(0, 8)` di JS | `:151-169` |
| Total | `.filter().length` di JS | `:223-224` |
| Satu-satunya agregasi SQL | `GROUP BY linkId` (klik per link) | `:25-33` |
| Index pendukung | **TIDAK ADA** — `analyticsEvents.pageId`/`createdAt` tanpa index (grep `.index(` = 0 di schema) | `schema.ts:365-383` |

**Fakta risiko (R-05):** untuk single-user berskala kecil, query full-scan 366 hari masih murah (SQLite ratusan-ribu baris); menjadi bottleneck saat tabel analytics membesar — **tanpa ukuran data aktual di repo, dampaknya Unknown**.

## 3. Redirect Click `/r/{linkId}` (Teramati)

- Insert click fire-and-forget lalu redirect: `app/r/[linkId]/route.ts:26`.
- Link resolution per request (tanpa cache teramati).

## 4. Dashboard Fetch (Teramati)

- Fetch **per tab aktif** — tab yang tidak dibuka tidak di-query: `app/dashboard/page.tsx:207-253` (mis. activity log hanya saat `activeTab` cocok, `:253`).
- Activity log: `limit = 100` (satu-satunya `.limit()` di jalur log) — `lib/db/activity-log.ts:47-53`.

## 5. Cron yang Memengaruhi Data Publik (Teramati)

| Job | Frekuensi | Dampak publik | Evidence |
|-----|-----------|---------------|----------|
| Live status YouTube (auto show/hide) | 2 menit | grup/link muncul-hilang otomatis | `instrumentation.ts:58` |
| Live badge | 2 menit | badge ON/OFF | `instrumentation.ts:67` |
| Content feeds → insert link baru | 15 menit | link RSS muncul otomatis | `lib/content-feeds.ts:101-113` |
| Dead link check → `lastCheckStatus` | harian 03:00 | penanda link mati | `lib/dead-links.ts:26-39` |

## 6. Build Perf (cross-global, Teramati)

- `cpus: 1` saat build (serial, peak RAM predictable) — `next.config.ts:32`
- `--max-old-space-size=768` — `Dockerfile:41`
- Fonts `preload: false` (hanya fetch saat dipakai) — `lib/fonts.ts:14-21`

## Tidak Teramati di Kode

- Benchmark/measure load time (Lighthouse, CI perf budget): tidak teramati.
- Kompresi/brotli config (server-side): `next start` default — tidak ada konfigurasi eksplisit.
- Rate limiting pada halaman publik (anti scraping analytics): tidak teramati.
