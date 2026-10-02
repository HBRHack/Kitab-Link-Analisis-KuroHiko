# Security Arch Master (Recovered from Code): Kitab Link Self-Hosted

> ⚠️ RECOVERED — hasil observasi kode pada 2026-10-02. Mekanisme TERAMATI; ini bukan audit keamanan (audit penuh = `docs/analyst/security-report.md`).

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 0.9.0 |
| **Last Updated** | 2026-10-02 |
| **Status** | Recovered |
| **Depth** | deep |

---

## 1. Auth Architecture (Teramati)

**Model: single-operator, stateless session — TANPA library auth eksternal.**

```
LOGIN FLOW
──────────
username+password ──▶ argon2id verify (lib/auth/password.ts:8-13)
                          │  Argon2id · memoryCost 19456 KiB · timeCost 2 · p 1
                          │  (param dieksplisitkan anti-drift — komentar :3-4)
                          ▼
                     TOTP aktif? ──▶ pending-2fa challenge (lib/auth/pending-2fa.ts)
                          │  verify: lib/auth/totp.ts (otplib, epochTolerance 30, :14-18)
                          ▼
                     SET cookie kitab_session
                        value = "{userId}.{expires}.{HMAC-SHA256}"
                        (lib/auth/session.ts:10-11; HMAC via hmac.ts:5;
                         key = getMasterSecret())
                        flags: httpOnly, sameSite=lax, path=/
                        secure = (x-forwarded-proto == "https")
                        (session.ts:14-20; secure-cookie.ts:8-11)
                        TTL 7 hari (session.ts:6)

VERIFIKASI
──────────
requireSession() (lib/auth/require-session.ts:4)
  └─ belum login → redirect("/login")  (:6-8)
verify: split "." + timingSafeEqual + expiry check (session.ts:34-37, hmac.ts:10-12)
```

### Komponen auth (peta `lib/auth/*` — 15 file)

| Mekanisme | Detail | Evidence |
|-----------|--------|----------|
| Password hashing | **Argon2id**, param OWASP eksplisit (19 MiB, t=2, p=1) | `lib/auth/password.ts:8-13` |
| Hash korup | treat sebagai `false` (bukan throw) | `lib/auth/password.ts:25-28` |
| Session | stateless HMAC cookie `kitab_session`, 7 hari, **bukan JWT, tanpa server-side store** | `lib/auth/session.ts:5,6,10-11` |
| Cookie verify | `crypto.timingSafeEqual` + panjang buffer | `lib/auth/hmac.ts:10-12` |
| Master secret | `data/.secret`, 32 byte, write atomik `flag:"wx"` `mode:0o600`, **lazy** (anti bake ke image) | `lib/auth/master-secret.ts:11,34,40`; komentar `:15-19,37-39` |
| Key derivation | `scryptSync(masterSecret, "kitab-link:{purpose}", len)` per-purpose + cache | `lib/auth/master-secret.ts:66-74` |
| Enkripsi secret | **AES-256-GCM**, IV 12, authTag 16, format `base64(iv+tag+cipher)` | `lib/auth/totp-crypto.ts:4-6,14-19` |
| Purpose enkripsi | `"totp-secret"` (`totp-crypto.ts:32-38`), `"youtube-api-key"` (`lib/youtube-api-key.ts:6`) |
| TOTP | otplib, issuer `"Kitab Link"`, 6 digit, toleransi ±30 detik | `lib/auth/totp.ts:9,14-18` |
| Backup code TOTP | 10 kode, `randomBytes(5)`; hash **SHA-256** (komentar `:29-36`: kode random → Argon2id nambah beban tanpa keamanan riil) | `lib/auth/totp.ts:21-27,29-36` |
| Setup token (akun pertama) | one-time, **in-memory saja** (gak dipersist), dicetak ke stdout, `timingSafeEqual` | `lib/auth/setup-token.ts:5-11,16-17,22-25` |
| Rate-limit | in-memory Map — **8 attempt / 5 menit**, cleanup tiap record | `lib/auth/unlock-rate-limit.ts:12,13,15,17-22` |
| API token | prefix `klk_` + `randomBytes(24)`; disimpan **SHA-256 hash** (komentar `:8-9`: token entropi tinggi, bukan password) | `lib/auth/api-tokens.ts:6,15,10-12` |
| Page access (visitor) | cookie terpisah `kitab_page_{id}`, HMAC sign, TTL 30 hari | `lib/auth/page-session.ts:5,7-9,13-26` |
| Password reset | token di-hash, dicetak ke log (tanpa SMTP) | `lib/auth/reset-password-token.ts`; REQUIREMENTS FR-005 |

## 2. Authorization (Teramati)

| Layer | Mekanisme | Evidence |
|-------|-----------|----------|
| Dashboard | `requireOwnedPage(pageId)` — select `WHERE id AND userId`, throw kalau bukan milik | `lib/db/pages.ts:59-71` (komentar tujuan `:57-58`) |
| API v1 | Bearer + scope + **`innerJoin(pages)` + `pages.userId = identity.userId`** | `app/api/v1/links/route.ts:23-28` |
| Admin path di custom domain | **404 Not Found** (bukan redirect) | `proxy.ts:42-44`; tujuan komentar `:10-13` |
| Server action ownership | dipanggil di ~12 lokasi (`actions.ts:46,62,74,82,93,106,117,133,301`, `page-actions.ts:42,64,78,104,125`, `settings-actions.ts:24,117,127`) | grep `requireOwnedPage` |
| **Known gap** | operasi baris anak (deleteLink dll) TIDAK menyertakan `pageId` di WHERE | `app/dashboard/actions.ts:92-95` — lihat `TECH-RISK-MASTER` KNOWN-002 |

## 3. Host / Network Controls (Teramati — `proxy.ts`)

```
Host header ──▶ proxy.ts:29-31
  ├─ resolve custom domain (proxy.ts:33-37) → page?
  │    ├─ ya + path admin → 404 (proxy.ts:42-44)
  │    ├─ belum verified → auto-verify DNS sekali (proxy.ts:49-51)
  │    └─ "/" → "/{slug}", "/privacy" → "/{slug}/privacy" (proxy.ts:53-58)
  └─ bukan custom domain → lanjut apa adanya (proxy.ts:39)
```

- Matcher mengecualikan `_next/static`, `_next/image`, `favicon.ico`, `robots.txt`, `uploads`, `r/` — `proxy.ts:61-66`.
- Access code page: dicek di **aplikasi** (`app/[slug]/password-actions.ts`), bukan proxy (fakta: tidak ada cek kode di `proxy.ts`).

## 4. Data-at-Rest & File Security (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Secret at rest | AES-256-GCM, key scrypt per-purpose dari master secret | `totp-crypto.ts:14-19`, `master-secret.ts:66-74` |
| Master secret file | `data/.secret`, `0o600`, atomic `wx` | `master-secret.ts:11,40` |
| Upload naming | `randomUUID().{ext}` (anti guessable) | `lib/images/storage.ts:24` |
| Upload serving | whitelist MIME per ekstensi (`webp,ttf,otf,woff,woff2,pdf,zip`), fallback `application/octet-stream` | `app/uploads/[...path]/route.ts:7-15,42` |
| Anti path-traversal | validasi per-segmen (`..`, `/`, `\` → 404) | `app/uploads/[...path]/route.ts:20-24` |
| Stream upload | `createReadStream` (anti RAM spike — komentar `:35-37`) | `app/uploads/[...path]/route.ts:38-45` |
| Link file upload | ukuran max 20MB; **ekstensi dari nama file user, tanpa whitelist** | `app/dashboard/actions.ts:147-152` |
| Data di volume | SQLite + uploads + secret = 1 folder `data/` (1 volume) | komentar `lib/images/storage.ts:5-6`, `Dockerfile:54-60` |

## 5. Privacy Controls (Teramati)

- **Tanpa penyimpanan IP**: negara hanya dari header proxy (`cf-ipcountry`/`x-vercel-ip-country`/`x-country-code`), tanpa GeoIP DB — `lib/analytics-capture.ts:35-42` (komentar `:32-34`).
- Referrer disimpan **hostname saja** — `lib/analytics-capture.ts:13-20`.
- Device = regex UA tanpa library — `lib/analytics-capture.ts:4-9`.

## 6. Env & Config (§12 — Teramati)

| Item | Teramati | Evidence |
|------|----------|----------|
| `.env.example` | **TIDAK ADA** — glob `.env*` = 0 file | scan negatif |
| Env keys (nama saja) | `DATABASE_PATH`, `UPLOAD_PATH`, `SECRET_PATH`, `NEXT_RUNTIME`, `NODE_ENV` | `lib/db/index.ts:6`, `lib/images/storage.ts:7`, `lib/auth/master-secret.ts:11`, `instrumentation.ts:3` |
| docker-compose env | **tanpa definisi env** — semua default | `docker-compose.yml:14-23` |
| Validasi config (zod) | **TIDAK aktif** — zod terpasang 0 import | `package.json:36` vs grep 0 |
| Hardcoded secret value | **0 match** scan (`sk_live_`, `AKIA`, `BEGIN.*KEY`, `password = "..."`) | scan pola seluruh source |
| Hardcoded non-secret | IP LAN dev `192.168.x.x` di `allowedDevOrigins` | `next.config.ts:14` |

> Desain konfigurasi (Teramati): "semua konfigurasi dari dashboard, bukan `.env`" — README.id.md:7; secrets runtime (YouTube key, TOTP) disimpan **ter-encrypt di DB**, bukan env (`lib/db/schema.ts:41-43`).

## 7. Kontrol yang SUDAH BAIK (cross-ref)

Cross-ref ke `docs/analyst/security-report.md` (kontrol #1–12) — materinya konsisten dengan bukti di atas; jangan diduplikasi, cukup rujuk.

## Inference Summary

> ⚠️ INFERENSI — alasan membangun auth custom (bukan next-auth/clerk) **tidak terdokumentasi di kode**; kemungkinan demi zero-dependency self-host + tanpa SMTP. Komentar per-fitur ada (`password.ts:3-4`, `totp.ts:29-36`, `master-secret.ts:15-19`) tapi tidak ada dokumen keputusan menyeluruh.

## Unknown

- Perilaku session saat master secret hilang/ganti (session invalid + TOTP/YouTube key tak terdekripsi) — disebut komentar `master-secret.ts:41-44` sebagai log warning, tapi recovery flow tidak teramati.
- Apakah cookie `secure` benar-benar aktif di deployment target — tergantung header `x-forwarded-proto` dari reverse proxy di depan (`secure-cookie.ts:8-11`), TIDAK teramati di repo (Caddy di-install `install.sh`, config-nya di luar repo).

## References

- `lib/auth/*` — 15 file mekanisme auth
- `proxy.ts` — host & admin path control
- `app/uploads/[...path]/route.ts` — file serving safety
- `docs/analyst/security-report.md` — audit menyeluruh (sumber lain)
