# ERD: Kitab Link (Self-Hosted Link-in-Bio)

Model data aplikasi link-in-bio self-hosted berskala single-user, multi-page. Seluruh data di SQLite (15 tabel) lewat Drizzle ORM; `pages` adalah aggregate root yang jadi induk dari hampir semua konten.

## Entity Relationship Diagram

+------------------+                    +------------------+
|      users       |                    |     themes       |
+------------------+                    +------------------+
| PK | id          |----1:N-------------| PK | id          |
|    | username    |                    | FK | user_id     |
|    | display_name|                    |    | name        |
|    | avatar_path |                    |    | tokens_json |
|    | password_hash                    |    | created_at  |
| FK | primary_page_id  <----1:N (self) |    +------------------+
|    | last_seen_app_version
|    | latest_release_json
|    | totp_secret   |
|    | totp_enabled  |
|    | totp_backup_codes_json
|    | youtube_api_key
|    | created_at    |
+------------------+
        |
        | 1:N
        v
+------------------+       +------------------+
|      pages       |       |  page_access_codes|
+------------------+       +------------------+
| PK | id          |--1:N--| PK | id          |
| FK | user_id     |       | FK | page_id     |
|    | slug        |       |    | code_hash   |
|    | domain_type |       |    | label       |
|    | domain_value|       |    | expires_at  |
|    | domain_verified     |    | created_at  |
|    | password_hash       |    +------------------+
|    | profile_json|
| FK | theme_id    |----N:1 (themes)
|    | created_at  |
+------------------+
   |   |   |   |   |
   |   |   |   |   +--1:N--+---> analytics_events
   |   |   |   +--------1:N--+---> activity_logs
   |   |   +------------1:N--+---> discord_widgets
   |   +----------------1:N--+---> content_feeds
   +--------------------1:N--+---> scheduled_rules
                              (lanjut di bawah)

+------------------+       +------------------+
|   link_groups    |       |      links       |
+------------------+       +------------------+
| PK | id          |<--1:N-| PK | id          |
| FK | page_id     |       | FK | page_id     |
|    | name        |       | FK | group_id    |  (onDelete: set null)
|    | order_index |       |    | order_index |
|    | is_visible  |       |    | title, url  |
+------------------+       |    | description |
        |                  |    | display_style
        | 1:N              |    | link_type   |
        +----------------->|    | thumbnail_path, image_*
                           |    | icon, icon_position
                           |    | featured, is_active
                           |    | schedule_start/end
                           |    | utm_source/medium/campaign
                           |    | last_checked_at, last_check_status
                           |    | consecutive_failures
                           +------------------+

+------------------+       +------------------+
|    api_tokens    |       |    webhooks      |
+------------------+       +------------------+
| PK | id          |       | PK | id          |
| FK | user_id     |       | FK | user_id     |
|    | name        |       | FK | page_id     |  (null = semua page)
|    | token_hash  |       |    | event_type  |
|    | scopes_json |       |    | target_url  |
|    | created_at  |       |    | secret      |
+------------------+       |    | created_at  |
                           +------------------+
                                    | 1:N
                                    v
                           +------------------+
                           |  webhook_queue   |
                           +------------------+
                           | PK | id          |
                           | FK | webhook_id  |
                           |    | payload_json|
                           |    | status      |
                           |    | attempts    |
                           |    | next_retry_at
                           |    | created_at  |
                           +------------------+

+------------------+  +------------------+  +------------------+
| scheduled_rules  |  |   live_badges    |  | discord_widgets  |
+------------------+  +------------------+  +------------------+
| PK | id           |  | PK | id          |  | PK | id          |
| FK | page_id      |  | FK | page_id (U) |  | FK | page_id     |
|    | trigger_type |  |    | channel_url |  |    | name        |
|    | config_json  |  |    | label       |  |    | guild_id    |
|    | target_type  |  |    | is_live     |  |    | style       |
|    | target_id    |  |    | video_url   |  |    | placement_mode
|    | last_checked_at   |  | last_checked_at  | floating_position
|    | last_state   |  |    | last_live_at|  |    | title       |
|    | last_triggered_at | | created_at   |  |    | show_* (4 kolom)
+------------------+  +------------------+  |    | width, height
   target_id --> link_groups.id             |    | is_enabled  |
                atau links.id               |    | created_at  |
                (polymorphic, bukan FK)     +------------------+

+------------------+       +------------------+
|  content_feeds   |       | analytics_events |
+------------------+       +------------------+
| PK | id           |      | PK | id          |
| FK | page_id      |      | FK | page_id     |
| FK | group_id     |      | FK | link_id     |  (BUKAN FK sungguhan)
|    | feed_url     |      |    | event_type  |
|    | label        |      |    | referrer    |
|    | rich_preview |      |    | device_type |
|    | is_active    |      |    | country     |
|    | last_item_guid     | | created_at     |
|    | last_checked_at    | +------------------+
+------------------+

+------------------+
|  activity_logs   |
+------------------+
| PK | id          |
| FK | page_id     |
|    | action      |
|    | detail      |
|    | source      |
|    | source_label|
|    | created_at  |
+------------------+


Legend:
  PK  = Primary Key
  FK  = Foreign Key
  1:N = One-to-Many
  N:1 = Many-to-One
  1:1 = One-to-One (unique constraint)
  (U) = unique column

## Entity Details

### users

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id akun |
| username | text | Yes (unique) | — | Identifier login (dulu `email`, diganti — app tanpa SMTP) |
| displayName | text | No | null | Nama tampil sidebar; fallback ke username |
| avatarPath | text | No | null | Avatar akun dashboard (beda dari avatar per-page) |
| passwordHash | text | Yes | — | argon2id (19 MiB, t=2, p=1) |
| primaryPageId | int (FK → pages) | No | null | Page yang tampil di `/` root; `onDelete: set null` |
| lastSeenAppVersion | text | No | null | Versi rilis terakhir yang di-dismiss dari notif bell |
| latestReleaseJson | text | No | null | Cache hasil cek GitHub Releases (sumber kebenaran update) |
| totpSecret | text | No | null | TOTP secret, **di-encrypt AES-256-GCM** |
| totpEnabled | boolean | Yes | false | Status 2FA |
| totpBackupCodesJson | text | No | null | Array hash argon2id backup code (sekali pakai) |
| youtubeApiKey | text | No | null | API key YouTube, **di-encrypt**; null = deteksi via scraping |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

**Constraints:** `username` UNIQUE; `primaryPageId` → `pages.id`.
**Indexes:** implisit via PK/UNIQUE.

### themes

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id theme |
| userId | int (FK → users) | Yes | — | Pemilik library; `onDelete: cascade` |
| name | text | Yes | — | Nama theme; unik per user via helper `uniqueThemeName()` (app-level, tanpa constraint DB) |
| tokensJson | text | Yes | `{}` | Design tokens (warna/font/spacing) dalam JSON |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

**Catatan:** library theme **lintas page** — "dipakai di page mana" dicatat di `pages.themeId`, bukan di tabel ini (komentar `lib/db/schema.ts:51-52`).

### pages

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id page |
| userId | int (FK → users) | Yes | — | Pemilik; `onDelete: cascade` |
| slug | text | Yes (unique) | — | Path publik `/{slug}` |
| domainType | enum | Yes | `subpath` | `subpath` \| `custom_domain` |
| domainValue | text | No | null | Hostname kustom (mis. `links.contoh.com`) |
| domainVerified | boolean | Yes | false | Tandai otomatis saat pertama kali ke-hit |
| passwordHash | text | No | null | Password proteksi halaman publik (null = publik) |
| profileJson | text | Yes | `{}` | Profil page (displayName, bio, avatar, banner, customCss, dst) |
| themeId | int (FK → themes) | No | null | Theme aktif; `onDelete: set null` |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

**Constraints:** `slug` UNIQUE; slug tidak boleh masuk `RESERVED_SLUGS` (`login`, `setup`, `dashboard`, `api`, `r`, `uploads`) dan wajib cocok `^[a-z0-9]+(-[a-z0-9]+)*$`.

### link_groups

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id grup |
| pageId | int (FK → pages) | Yes | — | `onDelete: cascade` |
| name | text | Yes | — | Nama grup |
| orderIndex | int | Yes | 0 | Urutan tampil (drag & drop) |
| isVisible | boolean | Yes | true | Show/hide manual |

### links

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id link |
| pageId | int (FK → pages) | Yes | — | `onDelete: cascade` |
| groupId | int (FK → link_groups) | No | null | `onDelete: set null` (link jadi ungrouped) |
| orderIndex | int | Yes | 0 | Urutan dalam grup/ungrouped |
| title | text | Yes | — | Judul tampil |
| url | text | Yes | — | **Kolom serbaguna**: URL / email / nomor / path file / teks copy / JSON |
| description | text | No | null | Deskripsi opsional |
| displayStyle | enum | Yes | `pill` | `pill` \| `rich` \| `icon` \| `image` |
| thumbnailPath | text | No | null | Thumbnail (rich/image), selalu WebP |
| imageHideBorder | boolean | Yes | false | Override tema (displayStyle `image`) |
| imageHideBackground | boolean | Yes | false | Override tema (displayStyle `image`) |
| imageShowTitle | boolean | Yes | false | Judul ditumpuk di atas gambar |
| imageShowContent | boolean | Yes | false | Icon/emoji ditumpuk di atas gambar |
| imageContentPath | text | No | null | Gambar kecil ditumpuk |
| imageRadius | int | No | null | Radius override (px) |
| imageShadow | enum | Yes | `theme` | `theme` \| `none` \| `sm` \| `md` \| `lg` |
| icon | text | No | null | Format `brand:whatsapp` / `generic:Mail`; null = auto favicon |
| scheduleStart | timestamp | No | null | Jendela tampil terjadwal |
| scheduleEnd | timestamp | No | null | Jendela tampil terjadwal |
| isActive | boolean | Yes | true | Show/hide manual |
| linkType | enum | Yes | `url` | `url`,`email`,`phone`,`whatsapp`,`file`,`embed`,`copy`,`accordion`,`countdown`,`discord_widget` |
| featured | boolean | Yes | false | Tandai link unggulan |
| utmSource / utmMedium / utmCampaign | text | No | null | Parameter UTM ditambahkan ke tujuan |
| iconPosition | enum | Yes | `top` | `top` \| `bottom` (hanya displayStyle `icon`) |
| lastCheckedAt | timestamp | No | null | Cek link mati terakhir |
| lastCheckStatus | enum | No | null | `ok` \| `dead` |
| consecutiveFailures | int | Yes | 0 | Reset ke 0 saat cek sukses |

**Constraints:** `displayStyle`/`linkType`/`iconPosition`/`imageShadow`/`lastCheckStatus` enum plain-text (tanpa CHECK constraint DB — diverifikasi di lapisan TS).
**Catatan:** `linkId` di `analytics_events` **bukan** FK agar riwayat klik link terhapus tidak ikut hilang.

### page_access_codes

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id kode |
| pageId | int (FK → pages) | Yes | — | `onDelete: cascade` |
| codeHash | text | Yes | — | argon2id |
| label | text | No | null | Nama label kode |
| expiresAt | timestamp | Yes | — | Masa berlaku; "revoke" = hapus baris |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

### api_tokens

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id token |
| userId | int (FK → users) | Yes | — | `onDelete: cascade` |
| name | text | Yes | — | Nama token |
| tokenHash | text | Yes (unique) | — | SHA-256 (token entropi tinggi, bukan password) |
| scopesJson | text | Yes | `[]` | Daftar scope izin |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

**Constraints:** `tokenHash` UNIQUE; raw token prefix `klk_` hanya ada sekali saat pembuatan.

### webhooks

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id webhook |
| userId | int (FK → users) | Yes | — | `onDelete: cascade` |
| pageId | int (FK → pages) | No | null | null = semua page milik user |
| eventType | enum | Yes | — | `subscriber.new`,`link.clicked`,`page.live_status_changed`,`test.ping` |
| targetUrl | text | Yes | — | URL tujuan (divalidasi kepanjangan/format minimum) |
| secret | text | Yes | — | HMAC-SHA256 signature `X-KitabLink-Signature` |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

### webhook_queue

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id antrean |
| webhookId | int (FK → webhooks) | Yes | — | `onDelete: cascade` |
| payloadJson | text | Yes | — | Payload event |
| status | enum | Yes | `pending` | `pending` \| `sent` \| `failed` |
| attempts | int | Yes | 0 | Percobaan kirim (maks 5) |
| nextRetryAt | timestamp | Yes | `unixepoch()` | Jadwal retry (backoff 10s→30s→1m→5m→30m) |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu antre |

### scheduled_rules

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id rule |
| pageId | int (FK → pages) | Yes | — | `onDelete: cascade` |
| triggerType | enum | Yes | — | `youtube_live` \| `manual` \| `weekly_schedule` |
| configJson | text | Yes | `{}` | Konfigurasi per trigger (channelUrl / days+mode+untilDate) |
| targetType | enum | Yes | — | `group` \| `link` |
| targetId | int | Yes | — | **Polymorphic (bukan FK)** → `link_groups.id` ATAU `links.id` |
| lastCheckedAt | timestamp | No | null | Polling terakhir |
| lastState | enum | No | null | `live` \| `offline` |
| lastTriggeredAt | timestamp | No | null | Perubahan state terakhir |

### live_badges

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id badge |
| pageId | int (FK → pages, **unique**) | Yes | — | 1 row per page |
| channelUrl | text | Yes | — | URL channel YouTube |
| label | text | No | null | Teks badge; fallback nama profil |
| isLive | boolean | Yes | false | Status live |
| videoUrl | text | No | null | URL video live (null bila offline) |
| lastCheckedAt | timestamp | No | null | Polling terakhir |
| lastLiveAt | timestamp | No | null | Edge offline → live terakhir |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

### discord_widgets

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id widget |
| pageId | int (FK → pages) | Yes | — | `onDelete: cascade`; 1 page bisa banyak widget |
| name | text | Yes | — | Nama internal dashboard (bukan tampilan publik) |
| guildId | text | Yes | — | Id server Discord |
| style | enum | Yes | `custom` | `custom` \| `discord` \| `classic` \| `iframe` |
| placementMode | enum | Yes | `inline` | `floating` \| `inline` |
| floatingPosition | enum | No | null | `left-top`…`right-bottom` (hanya `floating`) |
| title | text | No | null | Override nama publik; null = nama server asli |
| showMemberCount / showAvatars / showVoiceChannels / showJoinButton | boolean | Yes | true | Toggle tampilan |
| width / height | int | No | null | Diklaim 200–600 / 150–800 px di action |
| isEnabled | boolean | Yes | true | Toggle tanpa hapus konfigurasi |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

### content_feeds

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id feed |
| pageId | int (FK → pages) | Yes | — | `onDelete: cascade` |
| groupId | int (FK → link_groups) | Yes | — | Tujuan link baru; `onDelete: cascade` |
| feedUrl | text | Yes | — | URL RSS/Atom |
| label | text | No | null | Label |
| richPreview | boolean | Yes | true | Link baru `rich` + thumbnail, atau `pill` polos |
| isActive | boolean | Yes | true | Aktif/tidak |
| lastItemGuid | text | No | null | GUID item terakhir yang diproses |
| lastCheckedAt | timestamp | No | null | Cek terakhir |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

### analytics_events

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id event |
| pageId | int (FK → pages) | Yes | — | `onDelete: cascade` |
| eventType | enum | Yes | — | `page_view` \| `link_click` |
| linkId | int | No | null | **Bukan FK** — sengaja, supaya hapus link tak menghapus riwayat |
| referrer | text | No | null | Hostname referrer (bukan URL penuh) |
| deviceType | enum | No | null | `mobile` \| `tablet` \| `desktop` |
| country | text | No | null | Hanya terisi kalau reverse proxy kirim header geo |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu event |

### activity_logs

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id log |
| pageId | int (FK → pages) | Yes | — | `onDelete: cascade` |
| action | text | Yes | — | Key generik (`link_created`, `group_hidden`, `theme_changed`, …) |
| detail | text | No | null | Judul/nama terkait |
| source | enum | Yes | `dashboard` | `dashboard` \| `automation` \| `api` |
| sourceLabel | text | No | null | Nama rule / nama token API |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu aksi |

## Relationship Summary

| From | Cardinality | To | FK Location | Description |
|------|-------------|-----|-------------|-------------|
| users | 1:N | pages | `pages.userId` | Satu akun punya banyak page |
| users | 1:N | themes | `themes.userId` | Theme adalah library lintas page |
| users | 1:N | api_tokens | `apiTokens.userId` | Token API milik akun |
| users | 1:N | webhooks | `webhooks.userId` | Webhook milik akun |
| users | 1:1 (optional) | pages | `users.primaryPageId` | Page yang tampil di root `/` |
| pages | N:1 | themes | `pages.themeId` | Theme yang dipakai page (`set null`) |
| pages | 1:N | link_groups | `linkGroups.pageId` | Grup per page |
| pages | 1:N | links | `links.pageId` | Link per page |
| link_groups | 1:N | links | `links.groupId` | Anggota grup (`set null` bila grup dihapus) |
| pages | 1:N | page_access_codes | `pageAccessCodes.pageId` | Kode akses sementara |
| pages | 1:N | scheduled_rules | `scheduledRules.pageId` | Aturan show/hide otomatis |
| pages | 1:1 | live_badges | `liveBadges.pageId` (unique) | Badge live, maksimal 1 per page |
| pages | 1:N | discord_widgets | `discordWidgets.pageId` | Banyak widget per page |
| pages | 1:N | content_feeds | `contentFeeds.pageId` | Feed auto-post per page |
| pages | 1:N | webhooks | `webhooks.pageId` | null = semua page |
| link_groups | 1:N | content_feeds | `contentFeeds.groupId` | Feed menulis link ke grup ini |
| webhooks | 1:N | webhook_queue | `webhookQueue.webhookId` | Antrean kirim + retry |
| pages | 1:N | analytics_events | `analyticsEvents.pageId` | Metrik views/klik |
| pages | 1:N | activity_logs | `activityLogs.pageId` | Riwayat perubahan |
| scheduled_rules | *:1 (polymorphic) | link_groups / links | `scheduledRules.targetId` | Bukan FK — target bisa grup atau link |
| analytics_events | *:0 (loose) | links | `analyticsEvents.linkId` | Rujukan longgar, bukan FK |
