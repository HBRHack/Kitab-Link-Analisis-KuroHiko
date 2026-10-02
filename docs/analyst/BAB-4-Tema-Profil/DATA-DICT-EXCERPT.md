# Data Dictionary Excerpt — BAB-4 Tema & Profil

> **Excerpt dari [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat definisi atribut `themes` — satu-satunya tabel milik chapter. Atribut **profil page** bukan tabel tersendiri: semuanya berada di kolom `pages.profileJson` (bab induk: BAB-2), sehingga dicatat di sini sebagai rujukan silang, bukan duplikasi.

> **Scope excerpt ini:** themes (+ rujukan `pages.profileJson` dari BAB-2) — lihat [`ERD-EXCERPT.md`](./ERD-EXCERPT.md) untuk relasi antar entitas.

## Entities

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

**Business Rules:** — tidak ada BR bernomor untuk entitas ini (FR BAB-4 terpetak tanpa BR di [`REQUIREMENTS-MATRIX.md`](../00-Global/REQUIREMENTS-MATRIX.md)).

---

## Rujukan Silang (bukan entitas chapter ini)

### pages.profileJson (BAB-2)

Atribut profil page chapter ini tersimpan sebagai JSON di kolom `pages.profileJson` (validation: `customCss` max 20000 char — FR-044). Field yang tersimpan di dalamnya mencakup: displayName, bio, avatar/banner path, customCss, OG image, favicon, noIndex, streamer mode. Untuk definisi kolom lengkap `pages`, lihat [`../BAB-2-Manajemen-Page/DATA-DICT-EXCERPT.md`](../BAB-2-Manajemen-Page/DATA-DICT-EXCERPT.md) / [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md).

## Full Reference

> Definisi seluruh 15 tabel: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md).
