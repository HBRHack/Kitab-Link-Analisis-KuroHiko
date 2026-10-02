# SECURITY-RULES — BAB-1 Otentikasi & Akun (Recovered)

> ⚠️ RECOVERED dari kode — permission & kontrol per jalur auth. Mekanisme global: `00-Global/SECURITY-ARCH-MASTER.md`; ADR: 006, 007, 008, 010.

## Aturan per Jalur (Teramati)

| # | Jalur | Aturan | Evidence |
|---|-------|--------|----------|
| 1 | Register akun pertama (`/setup`) | Butuh **setup token in-memory** (sekali pakai, timing-safe, invalid setelah restart) | `lib/auth/setup-token.ts:5-8,16-17,22-25`; cetak boot `instrumentation.ts:21-35` |
| 2 | Login | argon2id verify (param OWASP eksplisit) → rate-limit `login:{username}` **8 attempt/5 menit** | `lib/auth/password.ts:8-13`; `app/login/actions.ts:31`; `unlock-rate-limit.ts:12-13` |
| 3 | Login + TOTP aktif | challenge `pending-2fa` sebelum cookie diterbitkan; verify otplib toleransi ±30 detik | `lib/auth/pending-2fa.ts`; `lib/auth/totp.ts:14-18` |
| 4 | Session | cookie `kitab_session` HMAC-signed, TTL 7 hari, httpOnly + sameSite=lax + secure dinamis | `lib/auth/session.ts:5,6,14-20` |
| 5 | Akses dashboard | `requireSession()` → `redirect("/login")` kalau kosong | `lib/auth/require-session.ts:6-8` |
| 6 | Reset password | rate-limit `reset-request:{username}`; token ke log (15 menit); balasan selalu `{sent:true}` (anti-enumeration) | `app/login/actions.ts:129`; UC-005/006 analyst |
| 7 | Ganti password | action `changePasswordAction` (verifikasi password lama via argon2id) | `app/dashboard/settings-actions.ts:139` |
| 8 | Enable TOTP | secret dienkripsi AES-256-GCM purpose `totp-secret`; backup code ×10 hashed SHA-256, sekali pakai | `lib/auth/totp-crypto.ts:32-38`; `lib/auth/totp.ts:21-27,44-49` |
| 9 | API token | raw `klk_`+24 byte hanya tampil saat dibuat; DB simpan SHA-256 hash; scope dicek per endpoint | `lib/auth/api-tokens.ts:6,15,26-27,33-42` |
| 10 | Master secret | `data/.secret` `0o600` atomic `wx`; lazy (anti ke-bake ke image) | `lib/auth/master-secret.ts:11,15-19,40` |

## Knowledge-Separation Rules (Teramati)

- **User tidak bisa membaca data user lain**: tidak teramati jalur multi-tenant; skema mendukung multi-user (`users` + `pages.userId`) tapi `setupFirstAdmin` hanya jalan saat DB kosong (analyst: asumsi single-user — Medium confidence).
- **Operator = root**: siapa pun dengan akses `docker logs` bisa setup/reset token (by-design, ADR-008).

## Known Gaps (cross-ref)

- Rate-limit key = username bebas → R-09 (`TECH-RISK-MASTER.md`)
- Tanpa rate-limit IP di lapisan proxy (ISSUE.md #3)
- `zod` tidak dipakai untuk validasi form auth → ADR-011

## Tidak Teramati di Kode

- Account lockout permanen / expiry akun: tidak teramati.
- Audit trail login gagal (login attempt logging): tidak teramati — hanya rate-limit counter.
- Pembatasan session per-device: tidak teramati (cookie tunggal, tanpa daftar sesi).
