# ERD Excerpt — BAB-3 Link & Grup

> **Excerpt dari [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat konten inti halaman: `link_groups` (kontainer) dan `links` (isi) — objek utama UC-018–UC-026. Keduanya anak langsung dari `pages` (BAB-2).

## Entity Relationship Diagram (Excerpt)

```
+------------------+       +------------------+
|   link_groups    |       |      links       |
+------------------+       +------------------+
| PK | id          |<--1:N-| PK | id          |
| FK | page_id     |       | FK | page_id     |
|    | name        |       | FK | group_id    |  (onDelete: set null)
|    | order_index |       |    | order_index |
|    | is_visible  |       |    | title, url  |
+------------------+       |    | description |
        ^                  |    | display_style, link_type
        |                  |    | thumbnail_path, image_*
        | (outbound FK     |    | icon, icon_position
        |  page_id ke      |    | featured, is_active
        |  pages)          |    | schedule_start/end
        |                  |    | utm_* , last_check_*
        |                  |    | consecutive_failures
   - - -+- - - - - - - - -+------------------+
   |     page_id --> pages [see Cross-Chapter]
   |
   +-- dirujuk inbound (bukan FK dari chapter ini):
       content_feeds.group_id (BAB-6), scheduled_rules.target_id (BAB-6),
       analytics_events.link_id (BAB-5, loose)

Legend:
  PK = Primary Key
  FK = Foreign Key
  1:N = One-to-Many
  - - - = relasi cross-chapter (tidak digambar box-nya)
```

## Cross-Chapter References

| FK Field | Points To | Chapter Owner | Relation | Notes |
|----------|-----------|---------------|----------|-------|
| links.pageId | pages.id | BAB-2-Manajemen-Page | N:1 | `onDelete: cascade` → see `00-Global/ERD-MASTER.md` |
| link_groups.pageId | pages.id | BAB-2-Manajemen-Page | N:1 | `onDelete: cascade` → see `00-Global/ERD-MASTER.md` |
| links.groupId | link_groups.id | This chapter (internal) | N:1 | `onDelete: set null` (link jadi ungrouped) |
| content_feeds.groupId → link_groups | — | BAB-6-Integrasi-Automasi | N:1 | Feed menulis link baru ke grup (inbound, lihat master) |
| scheduled_rules.targetId → link_groups / links | — | BAB-6-Integrasi-Automasi | *:1 polymorphic | **Bukan FK** — `targetType` menentukan target (lihat master) |
| analytics_events.linkId → links | — | BAB-5-Publik-Analytics | *:0 loose | **Bukan FK** — sengaja, hapus link tak menghapus riwayat (lihat master) |

## Entity Details (Excerpt)

> Detail penuh: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md)

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

## Relationship Summary (Excerpt)

| From | Cardinality | To | FK Location | In Excerpt? | Description |
|------|-------------|-----|-------------|-------------|-------------|
| link_groups | 1:N | links | `links.groupId` | Yes (internal) | Anggota grup (`set null` bila grup dihapus) |
| links | N:1 | pages | `links.pageId` | No (cross-chapter) | See `00-Global/ERD-MASTER.md` / BAB-2 |
| link_groups | N:1 | pages | `linkGroups.pageId` | No (cross-chapter) | See `00-Global/ERD-MASTER.md` / BAB-2 |
| content_feeds | N:1 | link_groups | `contentFeeds.groupId` | No (cross-chapter) | See BAB-6 |
| scheduled_rules | *:1 (polymorphic) | link_groups / links | `scheduledRules.targetId` | No (cross-chapter) | Bukan FK — see BAB-6 |
| analytics_events | *:0 (loose) | links | `analyticsEvents.linkId` | No (cross-chapter) | Rujukan longgar, bukan FK — see BAB-5 |

## Full Reference

> Untuk ERD lengkap dengan semua entitas dan relasi, lihat [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md).
