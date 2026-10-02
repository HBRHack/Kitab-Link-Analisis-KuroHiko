# Data Dictionary — Master

Definisi seluruh atribut model data **Kitab Link** — 15 tabel di SQLite (better-sqlite3, WAL mode) via Drizzle ORM. Sumber kebenaran: `lib/db/schema.ts` (migrasi di `drizzle/0000`–`0035`).

> **Naming:** entity PascalCase di dokumen, kolom DB snake_case, properti TS camelCase. Tabel `users` sengaja berisi 1 akun (single-user) tapi strukturnya mendukung multi-user.

## Entities

### users

Akun operator dashboard. Aplikasi single-user secara praktik, tapi skema tidak memaksakan itu.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id akun |
| username | text | Yes | — | UNIQUE, non-empty | Identifier login (dulu `email`, diganti karena tanpa SMTP) |
| displayName | text | No | null | max 100 char (dipotong di UI) | Nama tampil sidebar; fallback ke username |
| avatarPath | text | No | null | path relatif upload | Avatar akun dashboard (beda dari avatar page) |
| passwordHash | text | Yes | — | argon2id PHC | Hash password |
| primaryPageId | integer FK→pages | No | null | harus milik user | Page yang tampil di root `/` |
| lastSeenAppVersion | text | No | null | — | Versi rilis terakhir di-dismiss dari notif |
| latestReleaseJson | text | No | null | JSON | Cache respons GitHub Releases |
| totpSecret | text | No | null | ter-encrypt AES-256-GCM | Secret TOTP (bukan plaintext) |
| totpEnabled | boolean | Yes | false | — | Status 2FA |
| totpBackupCodesJson | text | No | null | JSON array hash argon2id | Backup code sekali pakai |
| youtubeApiKey | text | No | null | ter-encrypt (purpose `youtube-api-key`) | API key YouTube opsional |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-014: Password wajib 8–200 karakter, dicek ulang di server (bukan cuma `minLength` HTML).
- BR-015: Secret TOTP di-encrypt AES-256-GCM dengan key yang di-derive (scrypt) dari master secret; tidak bisa di-hash satu arah karena wajib bisa diverifikasi ulang.
- BR-016: Reset password tidak lewat email — token dicetak ke log container, berlaku 15 menit.

**Relationships:**
- users --1:N-- pages (via `pages.userId`)
- users --1:N-- themes (via `themes.userId`)
- users --1:N-- api_tokens, webhooks
- users --1:1(optional)-- pages (via `users.primaryPageId`)

---

### themes

Library tema milik user, dipakai ulang lintas page — halaman mana yang memakainya dicatat di `pages.themeId`, bukan di sini.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id theme |
| userId | integer FK→users | Yes | — | cascade delete | Pemilik library |
| name | text | Yes | — | unik per user via `uniqueThemeName()` (app-level, **tanpa** constraint DB) | Nama theme |
| tokensJson | text | Yes | `{}` | JSON object design tokens | Tokens warna/font/spacing |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Relationships:**
- themes --N:1-- users (via `themes.userId`)
- themes --1:N-- pages (via `pages.themeId`, `set null` saat theme dihapus)

---

### pages

Aggregate root: induk dari link, grup, tema aktif, analytics, dan seluruh automasi.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id page |
| userId | integer FK→users | Yes | — | cascade delete | Pemilik |
| slug | text | Yes | — | UNIQUE, `^[a-z0-9]+(-[a-z0-9]+)*$`, bukan reserved | Path publik `/{slug}` |
| domainType | enum | Yes | `subpath` | `subpath`\|`custom_domain` | Mode domain |
| domainValue | text | No | null | hostname regex (subdomain OK, tanpa skema/path) | Domain kustom |
| domainVerified | boolean | Yes | false | — | Ditandai otomatis saat pertama ke-hit |
| passwordHash | text | No | null | argon2id | Password proteksi halaman publik |
| profileJson | text | Yes | `{}` | JSON, customCss max 20000 char | Profil page |
| themeId | integer FK→themes | No | null | set null saat theme dihapus | Theme aktif |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-017: Slug harus lolos `SLUG_PATTERN` dan bukan `RESERVED_SLUGS` (`login`, `setup`, `dashboard`, `api`, `r`, `uploads`) — slug yang sama dengan route top-level akan di-shadow permanen.
- BR-018: Page terakhir tidak boleh dihapus (guard `userPages.length <= 1`).
- BR-019: Custom domain tidak boleh mengakses path admin (`/dashboard`, `/login`, `/setup`, `/api`) — di-block 404 oleh `proxy.ts`.
- BR-020: `domainVerified` diset `true` otomatis pada request pertama lewat host custom (bukti DNS + reverse proxy sudah benar).

**Relationships:**
- pages --1:N-- link_groups, links, page_access_codes, scheduled_rules, discord_widgets, content_feeds, analytics_events, activity_logs
- pages --1:1-- live_badges (unique `pageId`)
- pages --N:1-- themes (via `pages.themeId`)

---

### link_groups

Grup kontainer untuk link; mendukung show/hide manual dan automasi.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id grup |
| pageId | integer FK→pages | Yes | — | cascade delete | Page pemilik |
| name | text | Yes | — | non-empty setelah trim | Nama grup |
| orderIndex | integer | Yes | 0 | — | Urutan tampil |
| isVisible | boolean | Yes | true | — | Show/hide manual |

**Business Rules:**
- BR-021: `is_visible` manual dan `scheduled_rules` bisa berjalan bersama — automasi menulis state-nya sendiri ke target.

---

### links

Entitas inti. Kolom `url` dipakai ulang untuk semua `linkType`.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id link |
| pageId | integer FK→pages | Yes | — | cascade delete | Page pemilik |
| groupId | integer FK→link_groups | No | null | set null saat grup dihapus | Grup induk |
| orderIndex | integer | Yes | 0 | — | Urutan dalam grup |
| title | text | Yes | — | non-empty | Judul |
| url | text | Yes | — | non-empty; makna bergantung `linkType` | Tujuan / nilai link |
| description | text | No | null | — | Deskripsi opsional |
| displayStyle | enum | Yes | `pill` | `pill`\|`rich`\|`icon`\|`image` | Gaya tampil |
| thumbnailPath | text | No | null | selalu hasil `processImage()` → WebP | Thumbnail |
| imageHideBorder | boolean | Yes | false | — | Override tema (khusus `image`) |
| imageHideBackground | boolean | Yes | false | — | Override tema (khusus `image`) |
| imageShowTitle | boolean | Yes | false | — | Tumpuk judul di gambar |
| imageShowContent | boolean | Yes | false | — | Tumpuk icon/emoji di gambar |
| imageContentPath | text | No | null | — | Gambar kecil tumpukan |
| imageRadius | integer | No | null | — | Radius override (px) |
| imageShadow | enum | Yes | `theme` | `theme`\|`none`\|`sm`\|`md`\|`lg` | Bayangan override |
| icon | text | No | null | `brand:*` / `generic:*` / null=auto | Icon tombol |
| scheduleStart | timestamp | No | null | — | Jendela tampil terjadwal |
| scheduleEnd | timestamp | No | null | — | Jendela tampil terjadwal |
| isActive | boolean | Yes | true | — | Show/hide manual |
| linkType | enum | Yes | `url` | 10 nilai (lihat Value Lists) | Cara render + href |
| featured | boolean | Yes | false | — | Tandai unggulan |
| utmSource | text | No | null | — | UTM param |
| utmMedium | text | No | null | — | UTM param |
| utmCampaign | text | No | null | — | UTM param |
| iconPosition | enum | Yes | `top` | `top`\|`bottom` | Posisi baris icon (khusus `icon`) |
| lastCheckedAt | timestamp | No | null | — | Cek link mati terakhir |
| lastCheckStatus | enum | No | null | `ok`\|`dead` | Hasil cek terakhir |
| consecutiveFailures | integer | Yes | 0 | threshold 2 | Gagal beruntun |

**Business Rules:**
- BR-022: Semua upload gambar wajib lewat `processImage()` (sharp → resize → WebP); file asli tidak pernah disimpan.
- BR-023: Tipe `file` menerima URL eksternal ATAU upload sendiri; upload menang bila keduanya diisi. Maks 20MB.
- BR-024: Link ditandai `dead` hanya setelah **2x gagal berturut-turut**; `consecutiveFailures` di-reset ke 0 tiap cek sukses.
- BR-025: Semua klik diarahkan lewat `/r/{linkId}` dulu (bukan langsung ke tujuan) supaya analytics terekam sebelum redirect 302.
- BR-026: `displayStyle` lama `rounded` dimigrasi ke `pill` — kolom plain text tanpa CHECK constraint DB, jadi perubahan union type tidak butuh migration SQL.

**Relationships:**
- links --N:1-- pages
- links --N:1-- link_groups (opsional, set null)

---

### page_access_codes

Password tambahan yang berlaku sementara.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id kode |
| pageId | integer FK→pages | Yes | — | cascade delete | Page pemilik |
| codeHash | text | Yes | — | argon2id, min 4 char input | Hash kode |
| label | text | No | null | — | Label identifikasi |
| expiresAt | timestamp | Yes | — | clamp 1–720 jam (30 hari) | Masa berlaku |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-027: Kode berlaku maksimal 30 hari — batas atas supaya tidak terpakai sebagai password permanen kedua.
- BR-028: Revoke = hapus baris (tidak ada soft-delete).
- BR-029: Kode dicoba **setelah** password utama gagal, dan hanya yang belum lewat `expiresAt`.

---

### api_tokens

Token akses API eksternal (`Bearer klk_...`).

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id token |
| userId | integer FK→users | Yes | — | cascade delete | Pemilik |
| name | text | Yes | `"Untitled token"` bila kosong | — | Nama token |
| tokenHash | text | Yes | — | UNIQUE, SHA-256 hex | Hash token |
| scopesJson | text | Yes | `[]` | whitelist scope | Izin scope |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-030: Token di-hash SHA-256 biasa (bukan argon2) karena entropinya tinggi dan dihasilkan sistem — butuh lookup cepat.
- BR-031: Raw token hanya muncul sekali saat pembuatan; setelah itu hanya hash tersimpan.
- BR-032: Scope yang tidak ada di whitelist saat create **dibuang diam-diam** (`filter`), bukan error.

**Relationships:**
- api_tokens --N:1-- users

---

### webhooks

Registrasi endpoint keluar untuk event.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id webhook |
| userId | integer FK→users | Yes | — | cascade delete | Pemilik |
| pageId | integer FK→pages | No | null | null = semua page | Cakupan page |
| eventType | enum | Yes | — | 4 nilai (lihat Value Lists) | Jenis event |
| targetUrl | text | Yes | — | trim + non-empty (TANPA validasi skema/host) | URL tujuan |
| secret | text | Yes | — | `randomBytes(24)` hex | Secret HMAC signature |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-033: Payload ditandatangani HMAC-SHA256 dengan `secret`, dikirim sebagai header `X-KitabLink-Signature`.
- BR-034: Pengiriman lewat antrean + retry (bukan sinkron) dengan backoff eksponensial.

---

### webhook_queue

Antrean pengiriman + mekanisme retry.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id antrean |
| webhookId | integer FK→webhooks | Yes | — | cascade delete | Webhook induk |
| payloadJson | text | Yes | — | JSON | Payload event |
| status | enum | Yes | `pending` | `pending`\|`sent`\|`failed` | Status kirim |
| attempts | integer | Yes | 0 | maks 5 | Percobaan |
| nextRetryAt | timestamp | Yes | `unixepoch()` | backoff 10s/30s/1m/5m/30m | Jadwal retry |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu antre |

**Business Rules:**
- BR-035: Maksimal 5 percobaan; setelah itu status `failed` permanen.
- BR-036: Diproses cron tiap ~10 detik oleh `processWebhookQueue()`.

---

### scheduled_rules

Automasi show/hide grup/link.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id rule |
| pageId | integer FK→pages | Yes | — | cascade delete | Page pemilik |
| triggerType | enum | Yes | — | `youtube_live`\|`manual`\|`weekly_schedule` | Jenis pemicu |
| configJson | text | Yes | `{}` | JSON; shape per trigger type | Konfigurasi |
| targetType | enum | Yes | — | `group`\|`link` | Jenis target |
| targetId | integer | Yes | — | **bukan FK** (polymorphic) | Id target |
| lastCheckedAt | timestamp | No | null | — | Polling terakhir |
| lastState | enum | No | null | `live`\|`offline` | State tampil/sembunyi |
| lastTriggeredAt | timestamp | No | null | — | Perubahan state terakhir |

**Business Rules:**
- BR-037: `configJson` shape: `youtube_live` → `{channelUrl}`; `weekly_schedule` → `{days: number[0-6], mode: show_during|hide_during, untilDate}`.
- BR-038: `lastCheckedAt` di-update tiap tick polling; `lastTriggeredAt` hanya saat `lastState` benar-benar berubah.
- BR-039: `targetId` bukan FK karena bisa menunjuk `link_groups` ATAU `links` tergantung `targetType` — **tidak dijamin konsisten oleh DB**.

---

### live_badges

Badge "sedang live" di halaman publik.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id badge |
| pageId | integer FK→pages | Yes (unique) | — | UNIQUE, cascade delete | 1 row per page |
| channelUrl | text | Yes | — | — | URL channel YouTube |
| label | text | No | null | — | Teks badge; fallback nama profil |
| isLive | boolean | Yes | false | — | Status live |
| videoUrl | text | No | null | null bila offline | URL video live |
| lastCheckedAt | timestamp | No | null | — | Polling terakhir |
| lastLiveAt | timestamp | No | null | — | Edge offline → live terakhir |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-040: Sengaja dipisah dari `scheduledRules` agar badge dan toggle group/link bisa dipakai independen.

---

### discord_widgets

Konfigurasi widget server Discord (data member di-fetch live, tidak di-cache di tabel ini).

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id widget |
| pageId | integer FK→pages | Yes | — | cascade delete | Page pemilik |
| name | text | Yes | — | — | Nama internal dashboard |
| guildId | text | Yes | — | hanya digit di endpoint publik | Id server Discord |
| style | enum | Yes | `custom` | `custom`\|`discord`\|`classic`\|`iframe` | Gaya render |
| placementMode | enum | Yes | `inline` | `floating`\|`inline` | Mode penempatan |
| floatingPosition | enum | No | null | 6 nilai (lihat Value Lists) | Posisi floating |
| title | text | No | null | null = nama server asli | Override nama publik |
| showMemberCount | boolean | Yes | true | — | Tampilkan jumlah member |
| showAvatars | boolean | Yes | true | — | Tampilkan avatar |
| showVoiceChannels | boolean | Yes | true | — | Tampilkan voice channel aktif |
| showJoinButton | boolean | Yes | true | — | Tampilkan tombol join |
| width | integer | No | null | clamp 200–600 px | Lebar (px) |
| height | integer | No | null | clamp 150–800 px | Tinggi (px) |
| isEnabled | boolean | Yes | true | — | Toggle tanpa hapus konfigurasi |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-041: `style: "iframe"` mengabaikan seluruh toggle `show*`/`title` (tidak bisa dikontrol lintas origin).
- BR-042: Widget dimatikan (`isEnabled=false`) diperlakukan sama dengan "not found" (404) di endpoint config.

---

### content_feeds

Auto-post item baru dari RSS/Atom menjadi link.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id feed |
| pageId | integer FK→pages | Yes | — | cascade delete | Page pemilik |
| groupId | integer FK→link_groups | Yes | — | cascade delete | Tujuan link baru |
| feedUrl | text | Yes | — | URL valid | URL RSS/Atom |
| label | text | No | null | — | Label |
| richPreview | boolean | Yes | true | — | `rich` + thumbnail vs `pill` polos |
| isActive | boolean | Yes | true | — | Status aktif |
| lastItemGuid | text | No | null | — | GUID item terakhir diproses |
| lastCheckedAt | timestamp | No | null | — | Cek terakhir |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-043: Hanya item **lebih baru** dari `lastItemGuid` yang dibuat jadi link — cek berikutnya tidak mengulang.
- BR-044: Pendekatan RSS generik disengaja (bukan scraper per-platform) — YouTube punya RSS native, platform lain bisa dijembatani RSS-bridge.

---

### analytics_events

Metrik views & klik.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id event |
| pageId | integer FK→pages | Yes | — | cascade delete | Page terkait |
| eventType | enum | Yes | — | `page_view`\|`link_click` | Jenis event |
| linkId | integer | No | null | **bukan FK** | Id link (null untuk page_view) |
| referrer | text | No | null | hostname saja, bukan URL penuh | Asal kunjungan |
| deviceType | enum | No | null | `mobile`\|`tablet`\|`desktop` | Deteksi User-Agent |
| country | text | No | null | dari header proxy (`cf-ipcountry`, dst) | Negara |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu event |

**Business Rules:**
- BR-045: `linkId` sengaja bukan FK — link dihapus tidak boleh menghapus riwayat klik.
- BR-046: `country` selalu null bila reverse proxy tidak mengirim header geo — aplikasi **tidak menebak** lokasi.
- BR-047: Pencatatan bersifat fire-and-forget (`.catch`), kegagalan insert tidak menggagalkan request.

---

### activity_logs

Riwayat perubahan per page.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id log |
| pageId | integer FK→pages | Yes | — | cascade delete | Page terkait |
| action | text | Yes | — | key generik | Jenis aksi |
| detail | text | No | null | — | Judul/nama terkait |
| source | enum | Yes | `dashboard` | `dashboard`\|`automation`\|`api` | Pemicu aksi |
| sourceLabel | text | No | null | null bila `dashboard` | Nama rule / token |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu aksi |

**Business Rules:**
- BR-048: `detail` menyimpan data mentah (judul/nama), **pesan tampilan** dikomposisi dari kamus i18n agar tetap benar saat user ganti bahasa.

---

## Value Lists

### display_style (links.display_style)

| Code | Label | Description |
|------|-------|-------------|
| pill | Pill | Tombol rounded polos (default) |
| rich | Rich card | Kartu dengan thumbnail + deskripsi |
| icon | Icon | Ikon kecil, dua baris (`iconPosition` top/bottom) |
| image | Image | Gambar jadi "wajah" tombol |

### link_type (links.link_type)

| Code | Label | isi kolom `url` |
|------|-------|-----------------|
| url | URL biasa | URL tujuan |
| email | Email | Alamat email |
| phone | Telepon | Nomor telepon |
| whatsapp | WhatsApp | Nomor (auto-format) |
| file | File | Path upload lokal ATAU URL eksternal |
| embed | Embed | URL yang di-iframe |
| copy | Copy to clipboard | Teks bebas (kode promo, no. rekening) |
| accordion | Accordion list | JSON array `{label, value}` |
| countdown | Countdown | JSON `{endsAt, url}` |
| discord_widget | Discord widget | Id row `discord_widgets` sebagai string angka |

### domain_type (pages.domain_type)

| Code | Label | Description |
|------|-------|-------------|
| subpath | Subpath | Akses lewat `/{slug}` di domain utama |
| custom_domain | Custom domain | Akses lewat hostname sendiri (`domainValue`) |

### event_type (webhooks.event_type)

| Code | Label | Description |
|------|-------|-------------|
| subscriber.new | Subscriber baru | — |
| link.clicked | Link diklik | — |
| page.live_status_changed | Status live berubah | — |
| test.ping | Test ping | Dikirim manual dari dashboard |

### queue_status (webhook_queue.status)

| Code | Label | Description |
|------|-------|-------------|
| pending | Pending | Menunggu / sedang di-retry |
| sent | Sent | Berhasil terkirim |
| failed | Failed | Gagal setelah 5 percobaan |

### trigger_type (scheduled_rules.trigger_type)

| Code | Label | Description |
|------|-------|-------------|
| youtube_live | YouTube Live | Polling status live channel |
| manual | Manual | Ditoggle manual dari dashboard |
| weekly_schedule | Jadwal mingguan | Hari + mode show/hide |

### target_type (scheduled_rules.target_type)

| Code | Label | Description |
|------|-------|-------------|
| group | Grup | Target = `link_groups.id` |
| link | Link | Target = `links.id` |

### last_state / live_status (shared enum `live`/`offline`)

| Code | Label | Description |
|------|-------|-------------|
| live | Live / tampil | Target sedang ditampilkan |
| offline | Offline / sembunyi | Target sedang disembunyikan |

### last_check_status (links.last_check_status)

| Code | Label | Description |
|------|-------|-------------|
| ok | OK | Terakhir kali berhasil diakses |
| dead | Dead | Gagal 2x berturut-turut |

### widget_style (discord_widgets.style)

| Code | Label | Description |
|------|-------|-------------|
| custom | Custom | Mengikuti warna/font theme page |
| discord | Discord brand | Kulit warna brand Discord |
| classic | Classic | Niru layout iframe premade Discord |
| iframe | Iframe asli | `<iframe>` resmi Discord; toggle `show*`/`title` diabaikan |

### placement_mode (discord_widgets.placement_mode)

| Code | Label | Description |
|------|-------|-------------|
| floating | Floating | Menempel di posisi tetap layar (desktop saja) |
| inline | Inline | Tidak dirender otomatis; ditempatkan sebagai link biasa |

### floating_position (discord_widgets.floating_position)

| Code | Label |
|------|-------|
| left-top | Kiri atas |
| left-middle | Kiri tengah |
| left-bottom | Kiri bawah |
| right-top | Kanan atas |
| right-middle | Kanan tengah |
| right-bottom | Kanan bawah |

### event_type analytics (analytics_events.event_type)

| Code | Label | Description |
|------|-------|-------------|
| page_view | Page view | Halaman publik dilihat |
| link_click | Link click | Link diklik (lewat `/r/{id}`) |

### device_type (analytics_events.device_type)

| Code | Label | Detection |
|------|-------|-----------|
| mobile | Mobile | `Mobi|iPhone|Android` |
| tablet | Tablet | `iPad|Android(non-Mobile)|Tablet` |
| desktop | Desktop | default / tidak terdeteksi |

### log_source (activity_logs.source)

| Code | Label | sourceLabel |
|------|-------|-------------|
| dashboard | Dashboard | null |
| automation | Automation | nama rule |
| api | API | nama token |

### api_scope (api_tokens.scopes_json)

| Code | Label | Akses |
|------|-------|-------|
| links:read | Baca link | `GET /api/v1/links`, `GET /api/v1/links/:id` |
| links:write | Tulis link | `PATCH /api/v1/links/:id` |
| discord:read | Baca widget | `GET /api/v1/discord-widgets/:id` |
| discord:write | Tulis widget | `PATCH /api/v1/discord-widgets/:id` |

> ⚠️ **Temuan BUG-001:** UI menawarkan 4 scope, tapi whitelist saat pembuatan hanya memuat 2 scope pertama — lihat `bug-report.md`.

### reserved_slug (pages.slug — bukan kolom, aturan validasi)

`login`, `setup`, `dashboard`, `api`, `r`, `uploads`

### cron schedule (instrumentation.ts)

| Schedule | Job | Interval |
|----------|-----|----------|
| `*/10 * * * * *` | processWebhookQueue | 10 detik |
| `*/2 * * * *` | processScheduledRules | 2 menit |
| `*/2 * * * *` | processLiveBadges | 2 menit |
| `0 */6 * * *` | checkForUpdates | 6 jam |
| `*/15 * * * *` | processContentFeeds | 15 menit |
| `0 3 * * *` | checkDeadLinks | harian 03:00 |

---

## Data Flow Summary

| Source | Target | Data | Frequency | Method |
|--------|--------|------|-----------|--------|
| Dashboard (Server Action) | SQLite `links`, `link_groups` | CRUD link/grup | on-demand | Drizzle ORM |
| Pengunjung halaman publik | `analytics_events` | page_view / link_click | per request | insert fire-and-forget |
| `/r/{linkId}` | browser tujuan | redirect 302 | per klik | `NextResponse.redirect` |
| Cron `*/10s` | `webhooks.targetUrl` | payload event | 10 detik | `fetch` POST + HMAC header |
| Cron `*/2m` | YouTube (scraping/API) → `scheduled_rules`, `live_badges` | status live | 2 menit | `fetch` + timeout |
| Cron `*/15m` | RSS/Atom feed → `links` | item baru | 15 menit | `rss-parser` |
| Cron harian 03:00 | URL link type `url` | HEAD/GET health check | harian | `fetch` HEAD→GET |
| Cron `0 */6h` | GitHub Releases → `users.latestReleaseJson` | info rilis | 6 jam | `fetch` |
| Discord `widget.json` | halaman publik | data member live | per render | `fetch` + revalidate 60s |
| Browser upload | `data/uploads/**` | gambar WebP / file / font | on-demand | `sharp` + `fs` |
| Reverse proxy header | `analytics_events.country` | kode negara | per request | header lookup |
