# ERD Excerpt — BAB-1 Otentikasi & Akun

> **Excerpt dari [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat dua entitas milik akun: kredensial/session (`users`) dan kredensial programatik (`api_tokens`) — keduanya adalah objek yang di-autentikasi dalam UC-001–UC-009.

## Entity Relationship Diagram (Excerpt)

```
+------------------+                    +------------------+
|      users       |                    |    api_tokens    |
+------------------+                    +------------------+
| PK | id          |----1:N-------------| PK | id          |
|    | username    |                    | FK | user_id     |
|    | display_name|                    |    | name        |
|    | avatar_path |                    |    | token_hash  |  (UNIQUE)
|    | password_hash                    |    | scopes_json |
| FK | primary_page_id - - - - - - - -> |    | created_at  |
|    | last_seen_app_version     [see   +------------------+
|    | latest_release_json    Cross-     |
|    | totp_secret            Chapter    |
|    | totp_enabled        References]   |
|    | totp_backup_codes_json            |
|    | youtube_api_key                   |
|    | created_at                        |
+------------------+

Legend:
  PK = Primary Key
  FK = Foreign Key
  1:N = One-to-Many
  - - -> = relasi cross-chapter (tidak digambar box-nya, lihat tabel di bawah)
```

## Cross-Chapter References

| FK Field | Points To | Chapter Owner | Relation | Notes |
|----------|-----------|---------------|----------|-------|
| users.primaryPageId | pages.id | BAB-2-Manajemen-Page | 1:1 (optional) | Page yang tampil di root `/`; `onDelete: set null` → see `00-Global/ERD-MASTER.md` |
| api_tokens.userId | users.id | This chapter (internal) | 1:N | — |

## Entity Details (Excerpt)

> Detail penuh: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md)

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
| latestReleaseJson | text | No | null | Cache hasil cek GitHub Releases |
| totpSecret | text | No | null | TOTP secret, **di-encrypt AES-256-GCM** |
| totpEnabled | boolean | Yes | false | Status 2FA |
| totpBackupCodesJson | text | No | null | Array hash argon2id backup code (sekali pakai) |
| youtubeApiKey | text | No | null | API key YouTube, **di-encrypt**; null = deteksi via scraping |
| createdAt | timestamp | Yes | `unixepoch()` | Waktu pembuatan |

**Constraints:** `username` UNIQUE; `primaryPageId` → `pages.id`.
**Indexes:** implisit via PK/UNIQUE.

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

## Relationship Summary (Excerpt)

| From | Cardinality | To | FK Location | In Excerpt? | Description |
|------|-------------|-----|-------------|-------------|-------------|
| users | 1:N | api_tokens | `apiTokens.userId` | Yes (internal) | Token API milik akun |
| users | 1:1 (optional) | pages | `users.primaryPageId` | No (cross-chapter) | See `00-Global/ERD-MASTER.md` / BAB-2 |
| users | 1:N | pages, themes, webhooks | `*.userId` | No (cross-chapter) | Relasi lain ke luar chapter — see master |

## Full Reference

> Untuk ERD lengkap dengan semua entitas dan relasi, lihat [`00-Global/ERD-MASTER.md`](../00-Global/ERD-MASTER.md).
