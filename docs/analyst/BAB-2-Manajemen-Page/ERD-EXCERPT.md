# ERD Excerpt — BAB-2 Manajemen Page

> **Excerpt dari [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat **aggregate root** (`pages`) beserta kode akses sementaranya (`page_access_codes`) — objek utama UC-010–UC-017. Hampir semua entitas chapter lain adalah anak dari `pages`.

## Entity Relationship Diagram (Excerpt)

```
+------------------+       +------------------+
|      pages       |       | page_access_codes|
+------------------+       +------------------+
| PK | id          |--1:N--| PK | id          |
| FK | user_id     |       | FK | page_id     |
|    | slug        |       |    | code_hash   |  (argon2id)
|    | domain_type |       |    | label       |
|    | domain_value|       |    | expires_at  |
|    | domain_verified     |    | created_at  |
|    | password_hash       +------------------+
|    | profile_json|
| FK | theme_id    - - - - -> themes [see Cross-Chapter]
|    | created_at  |
+------------------+
   |
   +--1:N--> link_groups, links, scheduled_rules, discord_widgets,
             content_feeds, analytics_events, activity_logs
   +--1:1--> live_badges (unique pageId)
             (semua cross-chapter — lihat Cross-Chapter References)

Legend:
  PK = Primary Key
  FK = Foreign Key
  1:N / 1:1 = cardinality
  - - -> = relasi cross-chapter (tidak digambar box-nya)
```

## Cross-Chapter References

| FK Field | Points To | Chapter Owner | Relation | Notes |
|----------|-----------|---------------|----------|-------|
| pages.userId | users.id | BAB-1-Otentikasi-Akun | N:1 | Pemilik; `onDelete: cascade` → see `00-Global/ERD-MASTER.md` |
| pages.themeId | themes.id | BAB-4-Tema-Profil | N:1 | Theme aktif; `onDelete: set null` → see `00-Global/ERD-MASTER.md` |
| page_access_codes.pageId | pages.id | This chapter (internal) | 1:N | — |
| pages (children luar chapter) | pages.id | BAB-3, BAB-5, BAB-6 | 1:N | `link_groups`, `links`, `scheduled_rules`, `discord_widgets`, `content_feeds`, `analytics_events`, `activity_logs`, `live_badges` — see master |

## Entity Details (Excerpt)

> Detail penuh: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md)

### pages

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id page |
| userId | int (FK → users) | Yes | — | Pemilik; `onDelete: cascade` |
| slug | text | Yes (unique) | — | Path publik `/{slug}` |
| domainType | enum | Yes | `subpath` | `subpath` \| `custom_domain` |
| domainValue | text | No | null | Hostname kustom (mis. `links.contoh.com`) |
| domainVerified | boolean | Yes | false | Ditandai otomatis saat pertama kali ke-hit |
| passwordHash | text | No | null | Password proteksi halaman publik (null = publik) |
| profileJson | text | Yes | `{}` | Profil page (displayName, bio, avatar, banner, customCss, dst) |
| themeId | int (FK → themes) | No | null | Theme aktif; `onDelete: set null` |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

**Constraints:** `slug` UNIQUE; slug tidak boleh masuk `RESERVED_SLUGS` (`login`, `setup`, `dashboard`, `api`, `r`, `uploads`) dan wajib cocok `^[a-z0-9]+(-[a-z0-9]+)*$`.

### page_access_codes

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id kode |
| pageId | int (FK → pages) | Yes | — | `onDelete: cascade` |
| codeHash | text | Yes | — | argon2id |
| label | text | No | null | Nama label kode |
| expiresAt | timestamp | Yes | — | Masa berlaku; "revoke" = hapus baris |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

## Relationship Summary (Excerpt)

| From | Cardinality | To | FK Location | In Excerpt? | Description |
|------|-------------|-----|-------------|-------------|-------------|
| pages | 1:N | page_access_codes | `pageAccessCodes.pageId` | Yes (internal) | Kode akses sementara |
| pages | N:1 | users | `pages.userId` | No (cross-chapter) | See `00-Global/ERD-MASTER.md` / BAB-1 |
| pages | N:1 | themes | `pages.themeId` | No (cross-chapter) | See `00-Global/ERD-MASTER.md` / BAB-4 |
| link_groups, links, scheduled_rules, discord_widgets, content_feeds, analytics_events, activity_logs, live_badges | N:1 | pages | `*.pageId` | No (cross-chapter) | Anak `pages` di chapter lain — see master |

## Full Reference

> Untuk ERD lengkap dengan semua entitas dan relasi, lihat [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md).
