# Tech Risk Master (Recovered from Code): Kitab Link Self-Hosted

> ⚠️ RECOVERED — risiko yang TERAMATI dari kode pada 2026-10-02. Severity dinilai dalam konteks **self-hosted single-user** (konsisten dengan `docs/analyst/security-report.md`). Bukan pengganti audit.

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 0.9.0 |
| **Last Updated** | 2026-10-02 |
| **Status** | Recovered |
| **Depth** | deep |

---

## Risk Register

| ID | Risiko | Severity | Kategori | Evidence | Confidence |
|----|--------|----------|----------|----------|------------|
| R-01 | **Testing hampir nol** — 1 file test (`lib/youtube-live.test.ts`) untuk 64 server action + 6 route; runner `node --test` tanpa framework | 🟠 High (regresi) | §11 Testing | `package.json:10`; glob `**/*.test.ts` = 1 | High |
| R-02 | **`tsx` phantom dependency** — script `test` & `db:seed` andalkan `tsx` yang bukan dep langsung (masuk via `drizzle-kit`) → pecah kalau drizzle-kit berubah | 🟠 Medium | §11 | `package.json:10-11` vs `package-lock.json` (drizzle-kit → `tsx: ^4.21.0`), 0 dep `tsx` di `package.json` | High |
| R-03 | **CI/CD tanpa test/lint/audit** — release workflow cuma build+push image; `CLAUDE.md:33` bahkan menulis test "isi sesuai ... nanti" | 🟠 Medium | §8 DevOps | `.github/workflows/release.yml` (tanpa step test); `CLAUDE.md:33` | High |
| R-04 | **Tanpa Dependabot/Renovate** — CVE monitoring manual; `npm audit` belum perlu dijalankan | 🟠 Medium | §13 SBOM | `.github/` glob: hanya `workflows/release.yml` + banner | High |
| R-05 | **FK tanpa index + agregasi analytics tanpa LIMIT** — `analyticsEvents`/`activityLogs`/`links` FK full-scan; query 366 hari dimuat semua ke memori, bucketing di JS | 🟠 Medium (scale) | §4 Data | `schema.ts:91,101,104,367,391` (`.references()` saja); `lib/db/analytics.ts:196-208,65`; grep `.index(` = 0 | High |
| R-06 | **Ownership gap lintas-page** — `deleteLink`/`renameGroup`/`deleteGroup`/`toggle*`/`persistBoard` tidak menyertakan `pageId` di WHERE (guard hanya `requireOwnedPage(pageId)`) | 🟠 Medium (konsistensi data) | §7 Auth | `app/dashboard/actions.ts:92-95,65-66,75-76,301`; detail `docs/analyst/ISSUE.md` #2 | High |
| R-07 | **Scope `discord:*` tak bisa dibuat** — UI tawarkan 4 scope, whitelist create 2 → route discord-widgets selalu 403 | 🔴 High (fungsional) | §6 API | `token-creator.tsx:17-20` vs `integrations-actions.ts:12` vs `app/api/v1/discord-widgets/[id]/route.ts:16,42` | High |
| R-08 | **`pages.domainValue` tanpa UNIQUE** + `proxy.ts` resolusi `.limit(1)` → resolusi host tidak deterministik kalau duplikat | 🟡 Low | §4 Data | `schema.ts:77` (tanpa `.unique()`), `proxy.ts:33-37`; `slug` dan `tokenHash` punya unique, `domainValue` tidak | Medium (statis) |
| R-09 | **Rate-limit key = input bebas** — `login:{username}` / `reset-request:{username}` melanggar aturan komentar file-nya sendiri (key harus domain terbatas) | 🟡 Low (ada cleanup) | §7 Auth | `app/login/actions.ts:31,129` vs komentar `unlock-rate-limit.ts:8-11`; cleanup di `:17-22` | High |
| R-10 | **`zod` dead dependency** — `^4.6.2` di dependencies, 0 import; CLAUDE.md klaim "Validation: Zod" | 🟡 Low (housekeeping) | §12 Env | `package.json:36`, `CLAUDE.md:14`; grep `from "zod"` = 0 | High |
| R-11 | **Console-only logging** — tanpa structured logger/error tracking; secret (setup/reset token) sengaja dicetak ke stdout | 🟡 Low (self-host by design) | §10 Monitoring | pola `console.*` (`instrumentation.ts:19,37,91-96`); 0 dep logging; `setup-token.ts:16-17` | High |
| R-12 | **Cron in-process = single-instance only** — 6 job di proses Next; 2 container → dobel eksekusi; restart = kehilangan jadwal sementara | 🟡 Low (deploy saat ini 1 container) | §3 Backend | `instrumentation.ts:50-89`; `docker-compose.yml` 1 service; `bug-report.md:257` (perilaku multi-instance di luar cakupan) | High |
| R-13 | **Upload link-file tanpa whitelist ekstensi** — ext dari nama file user; mitigasi: MIME fallback `octet-stream` | 🟡 Low | §7 Auth | `app/dashboard/actions.ts:147-152`; mitigasi `app/uploads/[...path]/route.ts:42` | High |
| R-14 | **Tanpa `.env.example` / validasi config** — konfigurasi via dashboard + default hardcoded; env key tanpa validasi schema | 🟡 Low | §12 Env | glob `.env*` = 0; zod 0 import (R-10) | High |
| R-15 | **Versi pinning inkonsisten** — `next`/`react` exact, `better-sqlite3`/`sharp`/`zod` caret | 🟡 Low | §13 SBOM | `package.json:25,29` vs `:20,34,36` | High |
| R-16 | **Dependency audit (SCA) belum dilakukan** — `npm audit` tidak dijalankan selama ekstraksi (butuh instalasi); 7 dep prioritas: `next`, `better-sqlite3`, `sharp`, `@node-rs/argon2`, `drizzle-orm`, `otplib`, `rss-parser` | 🟠 Medium (unknown) | §13 SBOM | scan negatif; cross-ref `docs/analyst/ISSUE.md` A4 | High (fakta "belum dicek") |
| R-17 | **IP LAN dev hardcoded** di `allowedDevOrigins` — config personal ke-commit | ⚪ Info | §12 Env | `next.config.ts:14` | High |
| R-18 | **DangerouslySetInnerHTML** di 3 lokasi — theme init script (`app/layout.tsx:40`), custom font (`public-page-body.tsx:97`), custom CSS milik pemilik (`:100`) | ⚪ Info (admin-only input) | §7 Auth | `app/layout.tsx:40`; `public-page-body.tsx:97,100` | High |

## SBOM Snapshot (§13)

| Item | Nilai | Evidence |
|------|-------|----------|
| Direct dependencies | 22 (dependencies) + 13 (devDependencies) | `package.json:14-37,38-52` |
| Lockfile | `package-lock.json` (npm, terkunci) | repo root |
| `npm audit` / CVE check | **TIDAK DIJALANKAN** — butuh instalasi dependensi (mengubah working dir) | — |
| License per package | Tabel per-package belum diekstrak dari lockfile | `package-lock.json` field `license` |
| Transitive count | belum dihitung | — |

> Aturan skill: CVE Critical/High **reachable** di path auth/payment/data = High. Status sekarang: **belum diketahui** (audit belum jalan) → R-16. Jangan panik untuk dev-only transitive.

## Prioritas (factual, bukan rekomendasi urutan kerja)

1. **R-07** — fitur API yang dikirim tapi tidak bisa dipakai (fungsional rusak sekarang).
2. **R-01/R-03/R-04** — cluster testing/CI/SBOM: tanpa ini, perubahan apapun tak terverifikasi.
3. **R-05** — meledak saat data analytics tumbuh (single-user mungkin bertahun-tahun aman).
4. Sisanya — dokumentasikan; sebagian besar sudah punya mitigasi eksplisit di kode.

## Cross-Reference

- Bug fungsional lengkap (dengan saran perbaikan): `docs/analyst/ISSUE.md` (5 issue) + `docs/analyst/bug-report.md`.
- Kontrol keamanan yang sudah baik: `docs/analyst/security-report.md` §162 dst.
- Kontradiksi terkait stack/deployment: `extraction-report-arch.md` § Contradictions.
