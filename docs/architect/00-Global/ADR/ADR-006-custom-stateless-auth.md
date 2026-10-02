# ADR-006: Custom Auth — argon2id + Stateless HMAC Cookie + TOTP (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-006 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-007 (master secret), ADR-008 (token via log), ADR-010 (rate-limit) |
| **Related Requirements** | BAB-1 (UC-001–UC-009), FR-001+ |
| **Scope** | BAB-1-Otentikasi-Akun |
| **Chapter(s) Terdampak** | BAB-1 (utama), BAB-2 (page access) |
| **Confidence** | High |

## Context (Teramati)

Login, session, dan 2FA dibangun manual di `lib/auth/*` (15 file) tanpa next-auth/clerk/supabase; session berupa cookie HMAC stateless yang tidak disimpan di server.

## Evidence (Fakta — WAJIB)

- Hashing: `lib/auth/password.ts:8-13` — Argon2id (`algorithm: 2`), `memoryCost: 19456` (19 MiB), `timeCost: 2`, `parallelism: 1`; komentar `:3-4`: param OWASP dieksplisitkan "biar gak diam-diam berubah kalau default upstream ganti"
- Session: cookie `kitab_session` (`session.ts:5`), value `"{userId}.{expires}.{HMAC}"` (`:10-11`), TTL 7 hari (`:6`), flags httpOnly/lax/secure-dinamis (`:14-20`)
- HMAC: `lib/auth/hmac.ts:5` (`sha256` + `getMasterSecret()`), verify `timingSafeEqual` (`:10-12`)
- Secure flag dari header proxy: `lib/auth/secure-cookie.ts:8-11` (`x-forwarded-proto === "https"`)
- Guard: `requireSession()` → `redirect("/login")` (`lib/auth/require-session.ts:6-8`)
- TOTP: `otplib` (`lib/auth/totp.ts:1`), issuer `"Kitab Link"` (`:9`), toleransi ±30 detik (`:14-18`), 10 backup code `randomBytes(5)` hashed SHA-256 (`:21-27`, alasan komentar `:29-36`)
- Page access terpisah: cookie `kitab_page_{id}`, TTL 30 hari (`lib/auth/page-session.ts:5,7-9`)
- **0 import** library auth eksternal (grep `next-auth|@clerk|@supabase|jsonwebtoken` = 0 di source)
- Versi: `@node-rs/argon2@^2.2.1`, `otplib@^13.5.0` (`package.json:19,27`)

## Inference (Dugaan — WAJIB dilabeli jika ada)

> ⚠️ INFERENSI — kenapa tidak pakai next-auth/clerk **tidak terdokumentasi** di kode/README. Kemungkinan karena self-host tanpa dependensi layanan eksternal + tanpa SMTP; alasan tidak terdokumentasi.
> Per-parameter (anti-drift OWASP) punya komentar fakta — bukan inferensi.

## Alternatives (Hanya Jika Teramati di Kode)

- Tidak ada alternatif teramati (tidak ada sisa file auth library / dependensi terhapus di lockfile yang menunjukkan migrasi).

## Consequences (Teramati Saja)

### Observed Positive
- Tanpa tabel session → restart server tidak memutus login (kecuali master secret ganti — `master-secret.ts:41-44`)
- Param hashing eksplisit anti-drift; verify timing-safe

### Observed Negative / Risk Pointer
- Session invalid otomatis saat master secret hilang/ganti — log warning saja (`master-secret.ts:41-44`)
- Rate-limit in-memory (ADR-010) melemahkan brute-force protection
- Seluruh alur auth jadi tanggung jawab sendiri (audit manual — security-report.md)

### Unknown
- Distribusi waktu clock-drift pengguna (toleransi TOTP 30 detik) — tanpa data.

## References (Kode Saja)

- `lib/auth/password.ts`, `session.ts`, `hmac.ts`, `secure-cookie.ts`, `require-session.ts`, `totp.ts`, `page-session.ts`
- `docs/analyst/security-report.md` — audit menyeluruh
