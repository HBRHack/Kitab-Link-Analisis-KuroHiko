# Business Rules — Kitab Link

Katalog 35 aturan bisnis **BR-014 – BR-048** yang teridentifikasi dari pembacaan kode (ekstraksi, mode Opsi A). Sebelum file ini ada, seluruh BR hanya didefinisikan tersebar sebagai catatan per-entity di [`00-Global/DATA-DICT-MASTER.md`](00-Global/DATA-DICT-MASTER.md) — file ini mengompilasinya jadi satu sumber rujukan tanpa mengubah isi aturan.

> **Aturan main file ini:**
> - **Rule** = pernyataan dari DATA-DICT-MASTER (identik, tidak diparafrase) atau kode.
> - **Status** = `Confirmed` bila terpetak di [`00-Global/REQUIREMENTS-MATRIX.md`](00-Global/REQUIREMENTS-MATRIX.md) (semua BR di bawah Confirmed).
> - **Category** = klasifikasi reviewer (Validation / Calculation / Authorization / Workflow / Constraint).
> - **Priority** = penilaian reviewer (Must = keamanan/integritas data; Should = lainnya) — ⚠️ *bukan* nilai dari kode.
> - **Nomor tidak boleh dipakai ulang** ([`NUMBERING-LOG.md`](NUMBERING-LOG.md)). Range BR-001–BR-013 tidak pernah ada (repo clone tanpa legacy numbering).
> - BAB-4 (Tema & Profil) sengaja **tidak punya BR** — semua FR-nya terpetak tanpa aturan (`—` di matrix).

---

## BAB-1 Otentikasi & Akun (users, api_tokens)

### BR-014: Password 8–200 karakter, divalidasi di server

| Field | Value |
|-------|-------|
| **ID** | BR-014 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: server action login/setup (ekstraksi) |

**Rule:** Password wajib 8–200 karakter, dicek ulang di server (bukan cuma `minLength` HTML).

**Terkait:** UC-001, UC-002, UC-034 · FR-001, FR-002, FR-008

### BR-015: Secret TOTP di-encrypt AES-256-GCM

| Field | Value |
|-------|-------|
| **ID** | BR-015 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/auth/totp-secret.ts` (ekstraksi) |

**Rule:** Secret TOTP di-encrypt AES-256-GCM dengan key yang di-derive (scrypt) dari master secret; tidak bisa di-hash satu arah karena wajib bisa diverifikasi ulang.

**Terkait:** UC-003, UC-007 · FR-003

### BR-016: Reset password via token di log (tanpa email)

| Field | Value |
|-------|-------|
| **ID** | BR-016 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/auth/reset-password-token.ts` (ekstraksi) |

**Rule:** Reset password tidak lewat email — token dicetak ke log container, berlaku 15 menit.

**Terkait:** UC-005, UC-006 · FR-005

### BR-030: API token di-hash SHA-256 (bukan argon2)

| Field | Value |
|-------|-------|
| **ID** | BR-030 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: token create/verify (ekstraksi) |

**Rule:** Token di-hash SHA-256 biasa (bukan argon2) karena entropinya tinggi dan dihasilkan sistem — butuh lookup cepat.

**Terkait:** UC-009, UC-043–UC-047 · FR-007, FR-069

### BR-031: Raw token hanya tampil sekali

| Field | Value |
|-------|-------|
| **ID** | BR-031 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: token create (ekstraksi) |

**Rule:** Raw token hanya muncul sekali saat pembuatan; setelah itu hanya hash tersimpan.

**Terkait:** UC-009 · FR-007

### BR-032: Scope di luar whitelist dibuang diam-diam

| Field | Value |
|-------|-------|
| **ID** | BR-032 |
| **Category** | Validation |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `filter` saat create token (ekstraksi) |

**Rule:** Scope yang tidak ada di whitelist saat create **dibuang diam-diam** (`filter`), bukan error.

> ⚠️ Terkait BUG-001: UI menawarkan 4 scope, whitelist create hanya memuat 2 — lihat [`bug-report.md`](bug-report.md).

**Terkait:** UC-009 · FR-007

---

## BAB-2 Manajemen Page (pages, page_access_codes)

### BR-017: Slug valid dan bukan reserved

| Field | Value |
|-------|-------|
| **ID** | BR-017 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: slugify + validasi slug (ekstraksi) |

**Rule:** Slug harus lolos `SLUG_PATTERN` dan bukan `RESERVED_SLUGS` (`login`, `setup`, `dashboard`, `api`, `r`, `uploads`) — slug yang sama dengan route top-level akan di-shadow permanen.

**Terkait:** UC-010, UC-011 · FR-014

### BR-018: Page terakhir tidak boleh dihapus

| Field | Value |
|-------|-------|
| **ID** | BR-018 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: guard `userPages.length <= 1` (ekstraksi) |

**Rule:** Page terakhir tidak boleh dihapus (guard `userPages.length <= 1`).

**Terkait:** UC-010, UC-015 · FR-013

### BR-019: Custom domain tidak boleh mengakses path admin

| Field | Value |
|-------|-------|
| **ID** | BR-019 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `proxy.ts` (ekstraksi) |

**Rule:** Custom domain tidak boleh mengakses path admin (`/dashboard`, `/login`, `/setup`, `/api`) — di-block 404 oleh `proxy.ts`.

**Terkait:** UC-017, UC-036 · FR-018

### BR-020: domainVerified auto-set saat ke-hit pertama

| Field | Value |
|-------|-------|
| **ID** | BR-020 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `proxy.ts` (ekstraksi) |

**Rule:** `domainVerified` diset `true` otomatis pada request pertama lewat host custom (bukti DNS + reverse proxy sudah benar).

**Terkait:** UC-017, UC-036 · FR-018, FR-089

### BR-027: Kode akses maksimal 30 hari

| Field | Value |
|-------|-------|
| **ID** | BR-027 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: clamp `durationHours` 1–720 (ekstraksi) |

**Rule:** Kode berlaku maksimal 30 hari — batas atas supaya tidak terpakai sebagai password permanen kedua.

**Terkait:** UC-013 · FR-016

### BR-028: Revoke = hapus baris

| Field | Value |
|-------|-------|
| **ID** | BR-028 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: delete by `codeId` + `pageId` (ekstraksi) |

**Rule:** Revoke = hapus baris (tidak ada soft-delete).

**Terkait:** UC-014 · FR-016

### BR-029: Kode dicoba setelah password utama, hanya yang belum expired

| Field | Value |
|-------|-------|
| **ID** | BR-029 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `app/[slug]/password-actions.ts` (ekstraksi) |

**Rule:** Kode dicoba **setelah** password utama gagal, dan hanya yang belum lewat `expiresAt`.

**Terkait:** UC-013, UC-014, UC-037 · FR-016

---

## BAB-3 Link & Grup (link_groups, links)

### BR-021: Show/hide manual dan automasi hidup berdampingan

| Field | Value |
|-------|-------|
| **ID** | BR-021 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: tulis `isVisible` dari dua sumber (ekstraksi) |

**Rule:** `is_visible` manual dan `scheduled_rules` bisa berjalan bersama — automasi menulis state-nya sendiri ke target.

**Terkait:** UC-022, UC-023 · FR-027

### BR-022: Semua gambar wajib lewat processImage() → WebP

| Field | Value |
|-------|-------|
| **ID** | BR-022 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/images.ts` (ekstraksi) |

**Rule:** Semua upload gambar wajib lewat `processImage()` (sharp → resize → WebP); file asli tidak pernah disimpan.

**Terkait:** UC-018, UC-033 · FR-026

### BR-023: Link tipe file — URL eksternal ATAU upload, maks 20MB

| Field | Value |
|-------|-------|
| **ID** | BR-023 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `saveLinkAction` (ekstraksi) |

**Rule:** Tipe `file` menerima URL eksternal ATAU upload sendiri; upload menang bila keduanya diisi. Maks 20MB.

**Terkait:** UC-018 · FR-025

### BR-024: Link mati setelah 2x gagal berturut-turut

| Field | Value |
|-------|-------|
| **ID** | BR-024 |
| **Category** | Calculation |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: cron `checkDeadLinks` (ekstraksi) |

**Rule:** Link ditandai `dead` hanya setelah **2x gagal berturut-turut**; `consecutiveFailures` di-reset ke 0 tiap cek sukses.

**Terkait:** UC-040 (implied) · FR-063, FR-088

### BR-025: Klik selalu lewat /r/{linkId} dulu

| Field | Value |
|-------|-------|
| **ID** | BR-025 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `app/r/[linkId]/route.ts` (ekstraksi) |

**Rule:** Semua klik diarahkan lewat `/r/{linkId}` dulu (bukan langsung ke tujuan) supaya analytics terekam sebelum redirect 302.

**Terkait:** UC-038 · FR-058

### BR-026: displayStyle plain text tanpa CHECK constraint

| Field | Value |
|-------|-------|
| **ID** | BR-026 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/db/schema.ts` enum plain text (ekstraksi) |

**Rule:** `displayStyle` lama `rounded` dimigrasi ke `pill` — kolom plain text tanpa CHECK constraint DB, jadi perubahan union type tidak butuh migration SQL.

**Terkait:** UC-018 · FR-025

---

## BAB-5 Halaman Publik & Analytics (analytics_events, activity_logs)

### BR-045: analytics.linkId sengaja bukan FK

| Field | Value |
|-------|-------|
| **ID** | BR-045 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/db/schema.ts` (ekstraksi) |

**Rule:** `linkId` sengaja bukan FK — link dihapus tidak boleh menghapus riwayat klik.

**Terkait:** UC-038, UC-041 · FR-058

### BR-046: country null bila tak ada header geo (tidak menebak)

| Field | Value |
|-------|-------|
| **ID** | BR-046 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/db/analytics.ts` (ekstraksi) |

**Rule:** `country` selalu null bila reverse proxy tidak mengirim header geo — aplikasi **tidak menebak** lokasi.

**Terkait:** UC-041 · FR-061, FR-087

### BR-047: Pencatatan analytics fire-and-forget

| Field | Value |
|-------|-------|
| **ID** | BR-047 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `.catch` insert (ekstraksi) |

**Rule:** Pencatatan bersifat fire-and-forget (`.catch`), kegagalan insert tidak menggagalkan request.

**Terkait:** UC-038, UC-041 · FR-058

### BR-048: activity_logs.detail = data mentah, pesan dari kamus i18n

| Field | Value |
|-------|-------|
| **ID** | BR-048 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `logActivity` + i18n (ekstraksi) |

**Rule:** `detail` menyimpan data mentah (judul/nama), **pesan tampilan** dikomposisi dari kamus i18n agar tetap benar saat user ganti bahasa.

**Terkait:** UC-018, UC-045, UC-051, UC-037 · FR-046, FR-065

---

## BAB-6 Integrasi & Automasi (webhooks, webhook_queue, scheduled_rules, live_badges, discord_widgets, content_feeds)

### BR-033: Payload webhook ditandatangani HMAC-SHA256

| Field | Value |
|-------|-------|
| **ID** | BR-033 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/webhooks.ts` (ekstraksi) |

**Rule:** Payload ditandatangani HMAC-SHA256 dengan `secret`, dikirim sebagai header `X-KitabLink-Signature`.

**Terkait:** UC-048, UC-049, UC-050 · FR-070

### BR-034: Pengiriman lewat antrean + retry (bukan sinkron)

| Field | Value |
|-------|-------|
| **ID** | BR-034 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/webhooks.ts` (ekstraksi) |

**Rule:** Pengiriman lewat antrean + retry (bukan sinkron) dengan backoff eksponensial.

**Terkait:** UC-050 · FR-070

### BR-035: Maks 5 percobaan → failed permanen

| Field | Value |
|-------|-------|
| **ID** | BR-035 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/webhooks.ts` (ekstraksi) |

**Rule:** Maksimal 5 percobaan; setelah itu status `failed` permanen.

**Terkait:** UC-050 · FR-070

### BR-036: Antrean webhook diproses cron tiap ~10 detik

| Field | Value |
|-------|-------|
| **ID** | BR-036 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `instrumentation.ts` `*/10 * * * * *` (ekstraksi) |

**Rule:** Diproses cron tiap ~10 detik oleh `processWebhookQueue()`.

**Terkait:** UC-050 · FR-070

### BR-037: configJson shape per trigger type

| Field | Value |
|-------|-------|
| **ID** | BR-037 |
| **Category** | Validation |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/db/scheduled-rules.ts` (ekstraksi) |

**Rule:** `configJson` shape: `youtube_live` → `{channelUrl}`; `weekly_schedule` → `{days: number[0-6], mode: show_during|hide_during, untilDate}`.

**Terkait:** UC-051 · FR-071

### BR-038: lastCheckedAt tiap tick, lastTriggeredAt hanya saat state berubah

| Field | Value |
|-------|-------|
| **ID** | BR-038 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/db/scheduled-rules.ts` (ekstraksi) |

**Rule:** `lastCheckedAt` di-update tiap tick polling; `lastTriggeredAt` hanya saat `lastState` benar-benar berubah.

**Terkait:** UC-051 · FR-071

### BR-039: targetId polymorphic, bukan FK

| Field | Value |
|-------|-------|
| **ID** | BR-039 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/db/schema.ts` (ekstraksi) |

**Rule:** `targetId` bukan FK karena bisa menunjuk `link_groups` ATAU `links` tergantung `targetType` — **tidak dijamin konsisten oleh DB**.

**Terkait:** UC-051 · FR-071

### BR-040: live_badges dipisah dari scheduled_rules

| Field | Value |
|-------|-------|
| **ID** | BR-040 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `lib/db/schema.ts` (ekstraksi) |

**Rule:** Sengaja dipisah dari `scheduledRules` agar badge dan toggle group/link bisa dipakai independen.

**Terkait:** UC-052 · FR-072

### BR-041: style iframe abaikan toggle show*/title

| Field | Value |
|-------|-------|
| **ID** | BR-041 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: komponen widget (ekstraksi) |

**Rule:** `style: "iframe"` mengabaikan seluruh toggle `show*`/`title` (tidak bisa dikontrol lintas origin).

**Terkait:** UC-054 · FR-074

### BR-042: Widget non-aktif = 404 di endpoint config

| Field | Value |
|-------|-------|
| **ID** | BR-042 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: endpoint config widget (ekstraksi) |

**Rule:** Widget dimatikan (`isEnabled=false`) diperlakukan sama dengan "not found" (404) di endpoint config.

**Terkait:** UC-054 · FR-074

### BR-043: Hanya item lebih baru dari lastItemGuid yang diproses

| Field | Value |
|-------|-------|
| **ID** | BR-043 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `processContentFeeds` (ekstraksi) |

**Rule:** Hanya item **lebih baru** dari `lastItemGuid` yang dibuat jadi link — cek berikutnya tidak mengulang.

**Terkait:** UC-053 · FR-073

### BR-044: RSS generik, bukan scraper per-platform

| Field | Value |
|-------|-------|
| **ID** | BR-044 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Kode: `rss-parser` (ekstraksi) |

**Rule:** Pendekatan RSS generik disengaja (bukan scraper per-platform) — YouTube punya RSS native, platform lain bisa dijembatani RSS-bridge.

**Terkait:** UC-053 · FR-073

---

## Indeks Cepat

| ID | Entitas | Bab | FR |
|----|---------|-----|----|
| BR-014 | users | BAB-1 | FR-001, FR-002, FR-008 |
| BR-015 | users | BAB-1 | FR-003 |
| BR-016 | users | BAB-1 | FR-005 |
| BR-030 | api_tokens | BAB-1 | FR-007, FR-069 |
| BR-031 | api_tokens | BAB-1 | FR-007 |
| BR-032 | api_tokens | BAB-1 | FR-007 |
| BR-017 | pages | BAB-2 | FR-014 |
| BR-018 | pages | BAB-2 | FR-013 |
| BR-019 | pages | BAB-2 | FR-018 |
| BR-020 | pages | BAB-2 | FR-018, FR-089 |
| BR-027 | page_access_codes | BAB-2 | FR-016 |
| BR-028 | page_access_codes | BAB-2 | FR-016 |
| BR-029 | page_access_codes | BAB-2 | FR-016 |
| BR-021 | link_groups | BAB-3 | FR-027 |
| BR-022 | links | BAB-3 | FR-026 |
| BR-023 | links | BAB-3 | FR-025 |
| BR-024 | links | BAB-3 | FR-063, FR-088 |
| BR-025 | links | BAB-3 | FR-058 |
| BR-026 | links | BAB-3 | FR-025 |
| BR-045 | analytics_events | BAB-5 | FR-058 |
| BR-046 | analytics_events | BAB-5 | FR-061, FR-087 |
| BR-047 | analytics_events | BAB-5 | FR-058 |
| BR-048 | activity_logs | BAB-5 | FR-046, FR-065 |
| BR-033 | webhooks | BAB-6 | FR-070 |
| BR-034 | webhooks | BAB-6 | FR-070 |
| BR-035 | webhook_queue | BAB-6 | FR-070 |
| BR-036 | webhook_queue | BAB-6 | FR-070 |
| BR-037 | scheduled_rules | BAB-6 | FR-071 |
| BR-038 | scheduled_rules | BAB-6 | FR-071 |
| BR-039 | scheduled_rules | BAB-6 | FR-071 |
| BR-040 | live_badges | BAB-6 | FR-072 |
| BR-041 | discord_widgets | BAB-6 | FR-074 |
| BR-042 | discord_widgets | BAB-6 | FR-074 |
| BR-043 | content_feeds | BAB-6 | FR-073 |
| BR-044 | content_feeds | BAB-6 | FR-073 |

**Total: 35 BR (BR-014 – BR-048)** · BAB-4: 0 BR · sumber definisi: [`00-Global/DATA-DICT-MASTER.md`](00-Global/DATA-DICT-MASTER.md) · traceability: [`00-Global/REQUIREMENTS-MATRIX.md`](00-Global/REQUIREMENTS-MATRIX.md)
