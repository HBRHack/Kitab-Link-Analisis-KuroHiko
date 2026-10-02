# Data Dictionary Excerpt — BAB-1 Otentikasi & Akun

> **Excerpt dari [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat definisi atribut dua entitas kredensial: `users` (otentikasi sesi) dan `api_tokens` (otentikasi programatik) — pasangan yang sama dengan [`ERD-EXCERPT.md`](./ERD-EXCERPT.md).

> **Scope excerpt ini:** users, api_tokens — lihat [`ERD-EXCERPT.md`](./ERD-EXCERPT.md) untuk relasi antar entitas.

## Entities

### users

Akun operator dashboard. Aplikasi single-user secara praktik, tapi skema tidak memaksakan itu.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id akun |
| username | text | Yes | — | UNIQUE, non-empty | Identifier login (dulu `email`, diganti karena tanpa SMTP) |
| displayName | text | No | null | max 100 char (dipotong di UI) | Nama tampil sidebar; fallback ke username |
| avatarPath | text | No | null | path relatif upload | Avatar akun dashboard (beda dari avatar page) |
| passwordHash | text | Yes | — | argon2id PHC | Hash password |
| primaryPageId | integer FK→pages | No | null | harus milik user | Page yang tampil di root `/` |
| lastSeenAppVersion | text | No | null | — | Versi rilis terakhir di-dismiss dari notif |
| latestReleaseJson | text | No | null | JSON | Cache respons GitHub Releases |
| totpSecret | text | No | null | ter-encrypt AES-256-GCM | Secret TOTP (bukan plaintext) |
| totpEnabled | boolean | Yes | false | — | Status 2FA |
| totpBackupCodesJson | text | No | null | JSON array hash argon2id | Backup code sekali pakai |
| youtubeApiKey | text | No | null | ter-encrypt (purpose `youtube-api-key`) | API key YouTube opsional |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-014: Password wajib 8–200 karakter, dicek ulang di server (bukan cuma `minLength` HTML).
- BR-015: Secret TOTP di-encrypt AES-256-GCM dengan key yang di-derive (scrypt) dari master secret; tidak bisa di-hash satu arah karena wajib bisa diverifikasi ulang.
- BR-016: Reset password tidak lewat email — token dicetak ke log container, berlaku 15 menit.

**Relationships:**
- users --1:N-- pages, themes, api_tokens, webhooks (via `*.userId`)
- users --1:1(optional)-- pages (via `users.primaryPageId`)

---

### api_tokens

Token akses API eksternal (`Bearer klk_...`).

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id token |
| userId | integer FK→users | Yes | — | cascade delete | Pemilik |
| name | text | Yes | `"Untitled token"` bila kosong | — | Nama token |
| tokenHash | text | Yes | — | UNIQUE, SHA-256 hex | Hash token |
| scopesJson | text | Yes | `[]` | whitelist scope | Izin scope |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu pembuatan |

**Business Rules:**
- BR-030: Token di-hash SHA-256 biasa (bukan argon2) karena entropinya tinggi dan dihasilkan sistem — butuh lookup cepat.
- BR-031: Raw token hanya muncul sekali saat pembuatan; setelah itu hanya hash tersimpan.
- BR-032: Scope yang tidak ada di whitelist saat create **dibuang diam-diam** (`filter`), bukan error.

**Relationships:**
- api_tokens --N:1-- users

---

## Value Lists (Relevan)

### api_scope (api_tokens.scopes_json)

| Code | Label | Akses |
|------|-------|-------|
| links:read | Baca link | `GET /api/v1/links`, `GET /api/v1/links/:id` |
| links:write | Tulis link | `PATCH /api/v1/links/:id` |
| discord:read | Baca widget | `GET /api/v1/discord-widgets/:id` |
| discord:write | Tulis widget | `PATCH /api/v1/discord-widgets/:id` |

> ⚠️ **Temuan BUG-001:** UI menawarkan 4 scope, tapi whitelist saat pembuatan hanya memuat 2 scope pertama — lihat [`bug-report.md`](../bug-report.md).

## Full Reference

> Definisi seluruh 15 tabel: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md).
