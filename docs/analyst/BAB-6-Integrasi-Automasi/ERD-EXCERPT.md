# ERD Excerpt — BAB-6 Integrasi & Automasi

> **Excerpt dari [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat 6 entitas integrasi/automasi — objek UC-043–UC-054: webhook keluar + antreannya (`webhooks`, `webhook_queue`), aturan show/hide otomatis (`scheduled_rules`), badge live (`live_badges`), widget Discord (`discord_widgets`), dan feed auto-post (`content_feeds`). Satu-satunya relasi internal chapter: `webhooks 1:N webhook_queue`.

## Entity Relationship Diagram (Excerpt)

```
+------------------+       +------------------+
|    webhooks      |       |  webhook_queue   |
+------------------+       +------------------+
| PK | id          |--1:N--| PK | id          |
| FK | user_id     |       | FK | webhook_id  |
| FK | page_id     |       |    | payload_json|
|    | event_type  |       |    | status      |
|    | target_url  |       |    | attempts    |
|    | secret      |       |    | next_retry_at
|    | created_at  |       |    | created_at  |
+------------------+       +------------------+

+------------------+  +------------------+  +------------------+
| scheduled_rules  |  |   live_badges    |  | discord_widgets  |
+------------------+  +------------------+  +------------------+
| PK | id           |  | PK | id          |  | PK | id          |
| FK | page_id      |  | FK | page_id (U) |  | FK | page_id     |
|    | trigger_type |  |    | channel_url |  |    | name        |
|    | config_json  |  |    | label       |  |    | guild_id    |
|    | target_type  |  |    | is_live     |  |    | style ...   |
|    | target_id    |  |    | video_url   |  |    | is_enabled  |
|    | last_* (3)   |  |    | last_* (2)  |  |    | created_at  |
+------------------+  +------------------+  +------------------+
   target_id --> link_groups.id ATAU links.id (polymorphic, bukan FK)

+------------------+
|  content_feeds   |
+------------------+
| PK | id           |
| FK | page_id      |
| FK | group_id     |
|    | feed_url     |
|    | label        |
|    | rich_preview |
|    | is_active    |
|    | last_item_guid
|    | last_checked_at
|    | created_at   |
+------------------+

Semua FK page_id/group_id --> pages & link_groups
   [cross-chapter — lihat tabel Cross-Chapter References]

Legend:
  PK = Primary Key  ·  FK = Foreign Key  ·  1:N = One-to-Many
  (U) = unique column  ·  polymorphic = bukan FK sungguhan
```

## Cross-Chapter References

| FK Field | Points To | Chapter Owner | Relation | Notes |
|----------|-----------|---------------|----------|-------|
| webhooks.userId | users.id | BAB-1-Otentikasi-Akun | N:1 | `onDelete: cascade` → see `00-Global/ERD-MASTER.md` |
| webhooks.pageId, scheduled_rules.pageId, live_badges.pageId, discord_widgets.pageId, content_feeds.pageId | pages.id | BAB-2-Manajemen-Page | N:1 | Semua `onDelete: cascade`; `webhooks.pageId` null = semua page → see master |
| content_feeds.groupId | link_groups.id | BAB-3-Link-Grup | N:1 | Tujuan link baru; `onDelete: cascade` → see master |
| scheduled_rules.targetId | link_groups.id ATAU links.id | BAB-3-Link-Grup | *:1 polymorphic | **Bukan FK** — `targetType` menentukan target; tidak dijamin konsisten oleh DB (BR-039) |
| webhooks.webhookId → webhook_queue | — | This chapter (internal) | 1:N | — |

## Entity Details (Excerpt)

> Detail penuh: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md)

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

## Relationship Summary (Excerpt)

| From | Cardinality | To | FK Location | In Excerpt? | Description |
|------|-------------|-----|-------------|-------------|-------------|
| webhooks | 1:N | webhook_queue | `webhookQueue.webhookId` | Yes (internal) | Antrean kirim + retry |
| webhooks | N:1 | users | `webhooks.userId` | No (cross-chapter) | See `00-Global/ERD-MASTER.md` / BAB-1 |
| webhooks | N:1 | pages | `webhooks.pageId` | No (cross-chapter) | null = semua page — see BAB-2 |
| scheduled_rules, live_badges, discord_widgets, content_feeds | N:1 | pages | `*.pageId` | No (cross-chapter) | See BAB-2 |
| content_feeds | N:1 | link_groups | `contentFeeds.groupId` | No (cross-chapter) | See BAB-3 |
| scheduled_rules | *:1 (polymorphic) | link_groups / links | `scheduledRules.targetId` | No (cross-chapter) | Bukan FK — see BAB-3 |

## Full Reference

> Untuk ERD lengkap dengan semua entitas dan relasi, lihat [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md).
