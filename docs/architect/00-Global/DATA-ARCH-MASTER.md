# Data Arch Master (Recovered from Code): Kitab Link Self-Hosted

> ⚠️ RECOVERED — hasil observasi kode pada 2026-10-02. Semua klaim ada `file:baris`.

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 0.9.0 |
| **Last Updated** | 2026-10-02 |
| **Status** | Recovered |
| **Depth** | deep |

---

## 1. Database Engine & Koneksi (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Engine | SQLite (file-based) | `lib/db/index.ts:8` (`new Database(dbPath)`) |
| Driver | better-sqlite3 `^13.0.3` — **sync, native binding** | `package.json:20` |
| Lokasi file | env `DATABASE_PATH`, default `data/kitab-link.db` | `lib/db/index.ts:6` (nama key saja) |
| Mode | WAL | `lib/db/index.ts:9` (`pragma("journal_mode = WAL")`) |
| FK enforcement | ON | `lib/db/index.ts:10` (`pragma("foreign_keys = ON")`) |
| Pooling | **Tidak ada** — 1 koneksi, dibuka saat module load | `lib/db/index.ts:8` |
| Pragma lain (busy_timeout, synchronous, cache_size) | **Tidak teramati di kode** | `lib/db/index.ts` cuma 12 baris |

> ⚠️ Catatan: tanpa `busy_timeout`, konfigurasi default SQLite (5 detik) yang berlaku — **INFERENSI**: kemungkinan cukup karena single-process; tidak ada comment yang mengonfirmasi.

## 2. ORM & Migrasi (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| ORM | Drizzle ORM `^0.45.2` | `package.json:23`, `lib/db/index.ts:12` (`drizzle(sqlite, { schema })`) |
| Dialect | `sqlite` | `drizzle.config.ts:5` |
| Schema | 1 file master: `lib/db/schema.ts` (408 baris) | tabel §3 |
| Migrasi | **36 file SQL** `0000`–`0035` | glob `drizzle/*.sql`; terbaru `0035_flaky_supernaut.sql` |
| Migrasi runner | `drizzle-orm/better-sqlite3/migrator` saat **boot**, di-await | `instrumentation.ts:15-19` |
| Migrasi CLI | `drizzle-kit` = devDependency (gak ikut image produksi) | `package.json:46`; alasan di komentar `instrumentation.ts:13-14` |
| File migrasi dibawa ke standalone | lewat `outputFileTracingIncludes` | `next.config.ts:39-41` (komentar `:35-38`: dibaca `readFileSync`, bukan import) |
| Seed | `scripts/seed-user.ts` via `npm run db:seed` | `package.json:11` |

**Boot guarantee (Teramati):** `migrate()` di-await sebelum server terima request — kalau throw, server gagal boot total (komentar `instrumentation.ts:5-11`). Tidak ada window schema-rusak.

**Contradiction signal (dicek):** TIDAK ada dual ORM — hanya Drizzle (Prisma/Mongoose tidak ada di `package.json`).

## 3. Schema — 15 Tabel (Teramati)

Semua dari `lib/db/schema.ts` (baris awal tabel):

| # | Tabel SQL | Export | Baris | Unique | Catatan |
|---|-----------|--------|-------|--------|---------|
| 1 | `users` | `users` | 4 | `username` (:9) | kolom `youtubeApiKey` ter-encrypt (:41-43) |
| 2 | `themes` | `themes` | 49 | — | `tokensJson` = design tokens |
| 3 | `pages` | `pages` | 63 | `slug` (:68); **`domainValue` TANPA unique** | aggregate root |
| 4 | `link_groups` | `linkGroups` | 89 | — | FK `pageId` (:91-93) tanpa index |
| 5 | `links` | `links` | 99 | — | FK `pageId`/`groupId` (:101-106) tanpa index; enum `displayStyle` (:116), `linkType` 10 nilai (:146-148) |
| 6 | `page_access_codes` | `pageAccessCodes` | 172 | — | kode akses sementara, hash argon2id |
| 7 | `api_tokens` | `apiTokens` | 185 | `tokenHash` (:191) | `scopesJson` (:192) |
| 8 | `webhooks` | `webhooks` | 198 | — | `eventType` 4 nilai (:205-207) |
| 9 | `webhook_queue` | `webhookQueue` | 215 | — | antrean proses in-process |
| 10 | `scheduled_rules` | `scheduledRules` | 233 | — | `triggerType`: youtube_live/manual/weekly_schedule (:238-240) |
| 11 | `live_badges` | `liveBadges` | 261 | `pageId` (:263-265) | "1 row per page" (komentar :260) |
| 12 | `discord_widgets` | `discordWidgets` | 288 | — | config widget publik |
| 13 | `content_feeds` | `contentFeeds` | 342 | — | RSS/Atom source |
| 14 | `analytics_events` | `analyticsEvents` | 365 | — | FK `pageId` (:367-369) tanpa index; `linkId` sengaja bukan FK (:371-373) |
| 15 | `activity_logs` | `activityLogs` | 389 | — | `source`: dashboard/automation/api (:400-402) |

### Index & FK Summary (Teramati — scan menyeluruh)

- **`.index(...)` Drizzle: 0 match** di seluruh `schema.ts`.
- `CREATE INDEX` non-unique di seluruh `drizzle/*.sql`: **0 statement** (semua CREATE INDEX = UNIQUE: `pages_slug_unique`, `users_username_unique`, `api_tokens_token_hash_unique`, `live_badges_page_id_unique` — lihat `drizzle/0000_odd_jackpot.sql:42`, `0023_rename_email_to_username.sql:4`, `0001_slow_sharon_ventura.sql:11`, `0016_uneven_mach_iv.sql:12`).
- FK **tanpa index** (hanya `.references()`): `links.pageId` (:101), `links.groupId` (:104), `linkGroups.pageId` (:91), `analyticsEvents.pageId` (:367), `activityLogs.pageId` (:391).
- → **RISIKO**: query publik per halaman (`WHERE pageId = ?`) + agregasi analytics (`WHERE pageId IN (...) AND createdAt >= ?` — `lib/db/analytics.ts:196-208`) jalan full-scan saat tabel membesar. Detail: `TECH-RISK-MASTER.md`.
- `pages.domainValue` tanpa UNIQUE + `proxy.ts:33-37` pakai `.limit(1)` → resolusi tidak deterministik kalau duplikat (KNOWN dari ISSUE.md #5).

### Migrasi legacy (Teramati)

- Tabel `preview_tokens` pernah ada (`drizzle/0019_thick_tattoo.sql`) → DROP di `drizzle/0022_smart_night_nurse.sql:1`, tidak ada di schema sekarang. (Kode menunjukkan penghapusan — bukan Superseded ADR karena tidak ada keputusan terdokumentasi.)

## 4. Data Flow per Kategori (Teramati)

### Writes

| Data | Jalur | Evidence |
|------|-------|----------|
| Page view | INSERT fire-and-forget, non-blocking sengaja | `lib/db/analytics.ts:10-14` (komentar `:8-9`) |
| Link click | INSERT fire-and-forget dari redirect route | `lib/db/analytics.ts:16-20`, `app/r/[linkId]/route.ts:26` |
| Activity log | INSERT fire-and-forget, pesan dikompose dari i18n | `lib/db/activity-log.ts:35-45`; komentar `schema.ts:385-388` |
| Webhook delivery | UPDATE status + backoff 5 attempt | `lib/webhooks.ts:37-49`, `MAX_ATTEMPTS` di `:6` |
| Board reorder | **Transaksi** multi-update `orderIndex` | `app/dashboard/actions.ts:300-…` (persistBoard) |
| Upload file | `randomUUID().ext` ke `data/uploads/` | `lib/images/storage.ts:21-27` |

### Reads berat (tanpa limit)

- Agregasi analytics: **satu SELECT mentah 366 hari tanpa LIMIT**, semua baris dimuat ke memori lalu bucketing di **JS, bukan SQL** — `lib/db/analytics.ts:196-208`, komentar `:88-90`, `YEAR_LOOKBACK_MS` di `:65`. Satu-satunya agregasi SQL: `GROUP BY linkId` (`:25-33`).
- Satu-satunya query dengan `.limit()` yang teramati di jalur log: `getActivityLogForPage(pageId, limit = 100)` — `lib/db/activity-log.ts:47-53`.
- Halaman publik: 1 select page (`.limit(1)`, `app/[slug]/page.tsx:44`) + 4 query paralel (`components/public-page-body.tsx:74-79`).

## 5. Caching (§5)

**Tidak teramati cache layer** — tanpa Redis/Memcached/`unstable_cache` (bukti scan negatif: `package.json:14-37`, grep = 0). Yang ada:

| Mekanisme | TTL/Lokasi | Evidence |
|-----------|-----------|----------|
| Next fetch cache Discord widget | `revalidate: 60` (60 detik) | `lib/discord-widget.ts:34` |
| In-process `channelIdCache` | Map, tanpa TTL, per-proses | `lib/youtube-live.ts:145` |
| File upload | `Cache-Control: public, max-age=31536000, immutable` | `app/uploads/[...path]/route.ts:44` |
| Rate-limit Map | window 5 menit + cleanup | `lib/auth/unlock-rate-limit.ts:13,17-22` |
| Data lokal | `next build` + force-dynamic (tanpa halaman cache) | `app/[slug]/page.tsx:12` |

Invalidation halaman: `revalidatePath("/[slug]", "page")` dipanggil dari 12+ server action (`app/dashboard/theme-actions.ts:26,36,50,58,91`, `settings-actions.ts:111,123,133`, `page-actions.ts:59,96,108`, `backup-actions.ts:141`, dll).

## 6. Backup & Retention (Teramati)

- **Backup app-level (bukan DB dump):** `exportPageDataAction` → JSON `PageBackup v3` berisi theme/profile/links **opsional per scope**; gambar/thumbnail sengaja TIDAK ikut (komentar `app/dashboard/backup-actions.ts:14-18`), path di-strip (`:35-41`).
- **Restore** = hapus total links+groups lama lalu insert (semantik replace — komentar `:115-118`).
- Backup otomatis DB, retention, atau RPO/RTO: **tidak teramati di kode** → lihat `extraction-report-arch.md` § What Was NOT Found.

## Inference Summary

> ⚠️ INFERENSI — tanpa index sama sekali pada FK kemungkinan karena skala single-user (ratusan baris) dianggap cukup; **tidak ada comment/dokumentasi yang mengonfirmasi**. Untuk usage multi-user/traffic tinggi, keputusan ini jadi risiko performa (TECH-RISK-MASTER).

## References

- `lib/db/schema.ts` — 15 tabel
- `lib/db/index.ts` — koneksi/WAL
- `drizzle/` — 36 migrasi
- `instrumentation.ts:15-19` — boot migration
- `lib/db/analytics.ts` — pola read/write analytics
