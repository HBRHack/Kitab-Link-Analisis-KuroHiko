# ERD Excerpt — BAB-5 Halaman Publik & Analytics

> **Excerpt dari [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat dua entitas catatan-peristiwa: `analytics_events` (metrik views/klik halaman publik) dan `activity_logs` (riwayat perubahan dashboard) — objek pendukung UC-035–UC-042. Keduanya anak langsung `pages` (BAB-2); tidak ada relasi antar keduanya.

## Entity Relationship Diagram (Excerpt)

```
+------------------+                     +------------------+
| analytics_events |                     |  activity_logs   |
+------------------+                     +------------------+
| PK | id           |                    | PK | id          |
| FK | page_id      |                    | FK | page_id     |
|    | event_type   |                    |    | action      |
|    | link_id      |  (BUKAN FK)        |    | detail      |
|    | referrer     |  - - - - - - - -> links [see Cross-Chapter]
|    | device_type  |                    |    | source      |
|    | country      |                    |    | source_label|
|    | created_at   |                    |    | created_at  |
+------------------+                    +------------------+
        |                                       |
        +-- page_id --> pages [see Cross-Chapter]+-- page_id --> pages

Legend:
  PK = Primary Key
  FK = Foreign Key
  - - -> = relasi ke entitas lain (bukan FK sungguhan / cross-chapter)
```

## Cross-Chapter References

| FK Field | Points To | Chapter Owner | Relation | Notes |
|----------|-----------|---------------|----------|-------|
| analytics_events.pageId | pages.id | BAB-2-Manajemen-Page | N:1 | `onDelete: cascade` → see `00-Global/ERD-MASTER.md` |
| activity_logs.pageId | pages.id | BAB-2-Manajemen-Page | N:1 | `onDelete: cascade` → see `00-Global/ERD-MASTER.md` |
| analytics_events.linkId | links.id | BAB-3-Link-Grup | *:0 loose | **BUKAN FK** — sengaja, supaya hapus link tak menghapus riwayat klik (lihat `links` catatan di master) |

## Entity Details (Excerpt)

> Detail penuh: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md)

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

## Relationship Summary (Excerpt)

| From | Cardinality | To | FK Location | In Excerpt? | Description |
|------|-------------|-----|-------------|-------------|-------------|
| analytics_events | N:1 | pages | `analyticsEvents.pageId` | No (cross-chapter) | See `00-Global/ERD-MASTER.md` / BAB-2 |
| activity_logs | N:1 | pages | `activityLogs.pageId` | No (cross-chapter) | See `00-Global/ERD-MASTER.md` / BAB-2 |
| analytics_events | *:0 (loose) | links | `analyticsEvents.linkId` | No (cross-chapter) | Bukan FK — see BAB-3 |
| analytics_events ↔ activity_logs | — | — | — | Tidak ada relasi langsung | Keduanya independen, hanya berbagi induk `pages` |

## Full Reference

> Untuk ERD lengkap dengan semua entitas dan relasi, lihat [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md).
