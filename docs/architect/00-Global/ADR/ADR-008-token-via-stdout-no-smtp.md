# ADR-008: Setup & Reset Token Dicetak ke Stdout — Tanpa SMTP (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-008 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-002, ADR-006 |
| **Related Requirements** | FR-005 (REQUIREMENTS-MATRIX: "Reset password via token di log container (tanpa SMTP)"), UC-005/006 |
| **Scope** | BAB-1-Otentikasi-Akun |
| **Chapter(s) Terdampak** | BAB-1 |
| **Confidence** | High |

## Context (Teramati)

Tidak ada integrasi email; token sekali-pakai untuk setup akun pertama dan reset password dibuat in-memory/di-hash lalu dicetak ke log container/stdout.

## Evidence (Fakta — WAJIB)

- Setup token: in-memory saja, gak dipersist (`lib/auth/setup-token.ts:5-8,12`), generate `randomBytes(16)` + `console.log` (`:16-17`), komentar `:9-11`: "Dicetak ke stdout (bukan file) biar kebaca lewat docker logs / podman logs / dashboard log viewer"
- Verifikasi timing-safe: `setup-token.ts:22-25`
- Dicetak saat boot kalau DB kosong: `instrumentation.ts:21-35`
- Reset password: token dicetak via log, berlaku 15 menit — `docs/analyst/BAB-1.../USE-CASE.md` UC-005; `lib/auth/reset-password-token.ts`
- Anti user-enumeration: `requestPasswordReset` selalu balas `{sent:true}` (security-report.md, BR-016)
- 0 dep email (`resend|sendgrid|nodemailer|smtp` tidak ada di `package.json:14-37`)

## Inference (Dugaan — WAJIB dilabeli jika ada)

- Komentar `setup-token.ts:9-11` menjelaskan mekanisme (fakta). Alasan tanpa SMTP: **tidak terdokumentasi sebagai keputusan**; INFERENSI: menghindari konfigurasi email untuk self-host sederhana (REQUIREMENTS FR-005 mencatat fakta "tanpa SMTP" tanpa alasan).

## Alternatives (Hanya Jika Teramati di Kode)

- Tidak ada (tanpa driver email yang dikomentari/dihapus).

## Consequences (Teramati Saja)

### Observed Positive
- Setup anti-hijack akun pertama tanpa dependensi eksternal (kontrol #6 security-report)
- Restart sebelum setup selesai → token lama invalid otomatis (komentar `setup-token.ts:5-8`)

### Observed Negative / Risk Pointer
- Siapa pun yang bisa baca `docker logs` bisa reset password → "akses log = akses root" (security-report.md:162, by-design untuk self-host)
- Operator harus tahu `docker logs` — UX bergantung pada akses terminal/container

### Unknown
- Tingkat kegagalan operator yang kehilangan akses log — tidak ada data.

## References (Kode Saja)

- `lib/auth/setup-token.ts`, `lib/auth/reset-password-token.ts`
- `instrumentation.ts:21-35`
- `docs/analyst/00-Global/REQUIREMENTS-MATRIX.md` (FR-005)
