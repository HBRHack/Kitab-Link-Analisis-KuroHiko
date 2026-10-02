# ADR-007: Master Secret File + Scrypt Per-Purpose + AES-256-GCM (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-007 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-006, ADR-002 |
| **Related Requirements** | BAB-1 (TOTP), BAB-6 (YouTube API key) |
| **Scope** | BAB-1-Otentikasi-Akun |
| **Chapter(s) Terdampak** | BAB-1, BAB-6 |
| **Confidence** | High |

## Context (Teramati)

Secret yang harus bisa didecrypt lagi (TOTP secret, YouTube API key) dienkripsi AES-256-GCM dengan key yang di-derive dari satu master secret file yang dibuat sekali di volume data.

## Evidence (Fakta — WAJIB)

- Master secret file: path `data/.secret` (env `SECRET_PATH`) — `lib/auth/master-secret.ts:11`
- Isi: `crypto.randomBytes(32)` (`:34`), ditulis atomik `flag:"wx"` `mode:0o600` (`:40`); komentar `:37-39`: "cuma SATU yang menang nulis; yang kalah baca ulang file punya si pemenang, bukan overwrite silent"
- Lazy load: komentar `:15-19` — kalau eager, `next build` ikut jalan → secret ke-bake permanen ke Docker image
- Kegunaan: sign session cookie **dan** derive key AES (komentar `:9-10`)
- Key derivation: `scryptSync(masterSecret, "kitab-link:{purpose}", length)` + cache per purpose (`:66-74`); komentar `:60-65`: scryptSync sengaja lambat ~50-100ms
- Enkripsi: AES-256-GCM, IV 12, authTag 16, `base64(iv+tag+cipher)` — `lib/auth/totp-crypto.ts:4-6,14-19`
- Purpose: `"totp-secret"` (`totp-crypto.ts:32-38`), `"youtube-api-key"` (`lib/youtube-api-key.ts:6`); kolom `users.youtubeApiKey` (`lib/db/schema.ts:41-43`)
- Pemakaian TOTP: `app/dashboard/totp-actions.ts:54`; YouTube key set/hapus: `app/dashboard/automation-actions.ts:106,112`
- Komentar `totp-crypto.ts:9-12`: "kalau master secret ilang, semuanya invalid bareng"

## Inference (Dugaan — WAJIB dilabeli jika ada)

- Alasan lazy-load & atomic write **ada di komentar** (fakta di Evidence) — bukan inferensi.
- Kenapa file-based (bukan env var): alasan tidak terdokumentasi; INFERENSI: agar secret tidak masuk `docker-compose.yml`/proses orchestrator dan otomatis ikut volume backup.

## Alternatives (Hanya Jika Teramati di Kode)

- Tidak ada alternatif teramati (tanpa KMS/env-encryption lama).

## Consequences (Teramati Saja)

### Observed Positive
- Secret tidak ke-bake ke image (ada komentar + log warning kalau volume gak ke-mount: `master-secret.ts:41-44`)
- Per-purpose isolation (session ≠ TOTP ≠ YouTube key)

### Observed Negative / Risk Pointer
- Kehilangan `data/.secret` → session invalid + TOTP/YouTube key tak terdekripsi — recovery flow tidak teramati
- Di platform tanpa persistent disk (serverless), mekanisme ini tidak jalan — kontradiksi deployment (report)

### Unknown
- Rotasi secret (ganti key tanpa mematikan 2FA user) — tidak teramati prosedur.

## References (Kode Saja)

- `lib/auth/master-secret.ts`, `lib/auth/totp-crypto.ts`, `lib/youtube-api-key.ts`
- `lib/db/schema.ts:41-43`
