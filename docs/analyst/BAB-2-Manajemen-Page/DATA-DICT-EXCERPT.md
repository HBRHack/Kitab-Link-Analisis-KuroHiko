# Data Dictionary Excerpt — BAB-2 Manajemen Page

> **Excerpt dari [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat definisi atribut aggregate root `pages` (plus kolom `profileJson` yang menampung seluruh profil & custom CSS) dan entitas kode akses sementara `page_access_codes`.

> **Scope excerpt ini:** pages, page_access_codes — lihat [`ERD-EXCERPT.md`](./ERD-EXCERPT.md) untuk relasi antar entitas.

## Entities

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
- pages --N:1-- users, themes

**Catatan `profileJson`:** bukan kolom terpisah — seluruh atribut profil page (displayName, bio, avatar, banner, `customCss`, OG image, favicon, noIndex, streamer mode) tersimpan sebagai JSON di kolom ini (profil page chapter BAB-4 merujuk ke sini).

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

## Value Lists (Relevan)

### domain_type (pages.domain_type)

| Code | Label | Description |
|------|-------|-------------|
| subpath | Subpath | Akses lewat `/{slug}` di domain utama |
| custom_domain | Custom domain | Akses lewat hostname sendiri (`domainValue`) |

### reserved_slug (pages.slug — bukan kolom, aturan validasi)

`login`, `setup`, `dashboard`, `api`, `r`, `uploads`

## Full Reference

> Definisi seluruh 15 tabel: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md).
