# Data Dictionary Excerpt — BAB-6 Integrasi & Automasi

> **Excerpt dari [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat definisi atribut 6 entitas integrasi/automasi: `webhooks` + `webhook_queue` (event keluar), `scheduled_rules` (show/hide otomatis), `live_badges` (status live), `discord_widgets` (widget server), dan `content_feeds` (auto-post RSS/Atom).

> **Scope excerpt ini:** webhooks, webhook_queue, scheduled_rules, live_badges, discord_widgets, content_feeds — lihat [`ERD-EXCERPT.md`](./ERD-EXCERPT.md) untuk relasi antar entitas.

## Entities

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

## Value Lists (Relevan)

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

### cron schedule (instrumentation.ts)

| Schedule | Job | Interval |
|----------|-----|----------|
| `*/10 * * * * *` | processWebhookQueue | 10 detik |
| `*/2 * * * *` | processScheduledRules | 2 menit |
| `*/2 * * * *` | processLiveBadges | 2 menit |
| `0 */6 * * *` | checkForUpdates | 6 jam |
| `*/15 * * * *` | processContentFeeds | 15 menit |
| `0 3 * * *` | checkDeadLinks | harian 03:00 |

## Full Reference

> Definisi seluruh 15 tabel: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md).
