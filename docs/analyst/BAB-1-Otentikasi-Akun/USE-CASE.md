# BAB-1 — Otentikasi & Akun

Chapter: `BAB-1-Otentikasi-Akun` · Blok BR: 001–050 · Blok UC: 001–009

> Dokumen ringkas (Opsi A). Alur detail per UC hanya ditulis untuk jalur kritis; sisanya dicatat ringkas.

## Use Cases

| UC | Judul | Actor | Trigger | Ringkasan Alur | Entities | BR |
|----|-------|-------|---------|----------------|----------|----|
| UC-001 | Setup akun admin pertama | Operator baru | Akses `/setup` saat DB kosong | Ambil setup token dari log → isi username/password (8–200 char) → server cek ulang DB kosong + `verifySetupToken` → hash argon2id → insert user → buat page default `main` → buat session → redirect `/dashboard` | users, pages | BR-014 |
| UC-002 | Login | Operator | Form `/login` | Trim+lowercase username → cek rate-limit `login:{username}` → lookup user → verify argon2id → kalau `totpEnabled` grant pending 2FA (5 menit) else buat session → redirect `/dashboard` | users | BR-014 |
| UC-003 | Verifikasi TOTP saat login | Operator | Setelah UC-002 butuh 2FA | Ambil userId dari cookie pending → cek rate-limit `totp:{userId}` → bila input 6 digit verify TOTP, selain itu coba backup code (sekali pakai, dihapus dari array) → buat session → redirect | users | BR-015 |
| UC-004 | Logout | Operator | Klik logout | `destroySession()` → hapus cookie `kitab_session` → redirect `/login` | — | — |
| UC-005 | Minta reset password | Operator (lupa password) | `/login/forgot-password` | Selalu balas `{sent:true}` (anti user-enumeration) → rate-limit `reset-request:{username}` → bila user ada, `generateResetToken` → token dicetak ke log container (berlaku 15 menit) | users | BR-016 |
| UC-006 | Reset password | Operator | `/login/reset-password` | Validasi username+token+password (8–200) → `verifyResetToken` → hash password baru → update → `consumeResetToken` → redirect `/login` | users | BR-014, BR-016 |
| UC-007 | Aktivasi 2FA (enroll) | Operator | Settings → 2FA | `startTotpEnrollment` (secret disimpan di cookie signed, bukan form) → tampilkan QR → `confirmTotpEnrollment` baca secret dari cookie → encrypt AES-256-GCM → generate backup codes (hash argon2id) → `totpEnabled=true` | users | BR-015 |
| UC-008 | Matikan / regenerate backup code | Operator | Settings → 2FA | Wajib verify password dulu → `disableTotpAction` null-kan secret+backup codes, ATAU `regenerateBackupCodesAction` ganti array hash lama | users | BR-015 |
| UC-009 | Buat & cabut API token | Operator | Settings → Integrations | `createApiTokenAction` filter scope ke whitelist → insert hash SHA-256 → **raw token dibalikkan sekali** → `revokeApiToken` delete by id + `userId` | apiTokens | BR-030, BR-031, BR-032 |

### Alur kritis (ditulis lengkap)

#### UC-002: Login — Main Flow

1. **Operator** isi username + password di `/login`
2. **System** normalisasi username (trim + lowercase)
3. **System** cek `isLockedOut("login:" + username)` — bila terkunci, balas error "terlalu banyak percobaan"
4. **System** cari user by username
5. **System** verify password dengan argon2id
6. **System** `clearAttempts` bila cocok
7. **System** bila `totpEnabled` → grant pending 2FA (cookie sementara 5 menit), return `{needsTotp:true}`
8. **System** selain itu → `createSession(userId)` → redirect `/dashboard`

#### UC-002: Exception Flows

- **EF-1 User tidak ada** → Step 4 gagal → `recordFailedAttempt` → balas pesan sama persis dengan password salah (anti-enumeration)
- **EF-2 Password salah** → `recordFailedAttempt` → pesan sama
- **EF-3 Rate limit tercapai** (8 percobaan / 5 menit) → balas `tooManyAttemptsError`
- **EF-4 Secret TOTP tak bisa di-decrypt** (volume tak ter-mount) → balas error spesifik, **bukan 500**; backup code tetap jalan

## Notes

- Session bersifat **stateless HMAC** — tidak ada tabel session di DB; keabsahan = signature valid + belum lewat `expires`.
- Rate limit **in-memory per proses** (reset saat restart) — disengaja untuk skala kecil; lihat `bug-report.md` BUG-003 untuk catatan pertumbuhan Map.
