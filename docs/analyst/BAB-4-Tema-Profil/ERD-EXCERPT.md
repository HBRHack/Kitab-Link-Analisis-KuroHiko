# ERD Excerpt — BAB-4 Tema & Profil

> **Excerpt dari [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat satu entitas inti: `themes` — library design tokens milik user yang dipakai ulang lintas page (UC-027–UC-034). Profil page sendiri disimpan di `pages.profileJson` (milik BAB-2), bukan tabel terpisah.

## Entity Relationship Diagram (Excerpt)

```
+------------------+
|      themes      |
+------------------+
| PK | id          |
| FK | user_id     - - - - -> users [see Cross-Chapter]
|    | name        |
|    | tokens_json |  (JSON design tokens)
|    | created_at  |
+------------------+
   ^
   | (inbound FK dari pages.theme_id — cross-chapter)
   - - - pages [see Cross-Chapter References]

Legend:
  PK = Primary Key
  FK = Foreign Key
  - - -> = relasi cross-chapter (tidak digambar box-nya)
```

## Cross-Chapter References

| FK Field | Points To | Chapter Owner | Relation | Notes |
|----------|-----------|---------------|----------|-------|
| themes.userId | users.id | BAB-1-Otentikasi-Akun | N:1 | Pemilik library; `onDelete: cascade` → see `00-Global/ERD-MASTER.md` |
| pages.themeId | themes.id | BAB-2-Manajemen-Page (inbound) | 1:N | Theme yang dipakai page; `onDelete: set null` → see `00-Global/ERD-MASTER.md` |

## Entity Details (Excerpt)

> Detail penuh: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md)

### themes

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | int (PK, auto) | Yes | autoincrement | Id theme |
| userId | int (FK → users) | Yes | — | Pemilik library; `onDelete: cascade` |
| name | text | Yes | — | Nama theme; unik per user via helper `uniqueThemeName()` (app-level, tanpa constraint DB) |
| tokensJson | text | Yes | `{}` | Design tokens (warna/font/spacing) dalam JSON |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

**Catatan:** library theme **lintas page** — "dipakai di page mana" dicatat di `pages.themeId`, bukan di tabel ini (komentar `lib/db/schema.ts:51-52`).

**Catatan chapter (profiling):** atribut profil page (`displayName`, bio, avatar, banner, `customCss`, OG image, favicon, noIndex, streamer mode) **bukan kolom terpisah** — semuanya masuk `pages.profileJson` milik BAB-2. Excerpt ini sengaja tidak menggandakan `pages`.

## Relationship Summary (Excerpt)

| From | Cardinality | To | FK Location | In Excerpt? | Description |
|------|-------------|-----|-------------|-------------|-------------|
| themes | N:1 | users | `themes.userId` | No (cross-chapter) | See `00-Global/ERD-MASTER.md` / BAB-1 |
| pages | N:1 | themes | `pages.themeId` | No (cross-chapter) | 1:N ke `themes`; see `00-Global/ERD-MASTER.md` / BAB-2 |
| users | 1:N | themes | `themes.userId` | No (cross-chapter, inbound) | Library lintas page milik akun — see master |

## Full Reference

> Untuk ERD lengkap dengan semua entitas dan relasi, lihat [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md).
