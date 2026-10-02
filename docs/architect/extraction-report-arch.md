# Extraction Report (Arch) — Kitab Link Self-Hosted

Date: 2026-10-02 | Source: `Kitab-Link-Self-Hosted/` (clone `https://github.com/Kurohiko-id/Kitab-Link-Self-Hosted` v0.9.0) | Depth: **deep** | Scope: **functional** (13 kategori di-scan, output dipangkas ke yang fungsional sesuai permintaan pemilik — tanpa `NFR-MAPPING-INDEX`, tanpa `architecture-spec`, per-chapter dibuat lazy hanya yang berisi)

## Summary

- Stack terdeteksi: Node 22 (Docker `node:22-slim`) · Next.js 16.3.5 App Router · React 19.2.8 · TypeScript 5 strict · SQLite (better-sqlite3 13, WAL) + Drizzle 0.45 · tanpa cache layer · custom auth (argon2id + HMAC cookie + TOTP) · node-cron in-process · Docker multi-stage → GHCR
- ADR Recovered: **12** (Contradictory: **1** — ADR-011/zod)
- Endpoints: **8 route API** (5 Bearer v1 + 3 publik) · Integrations outbound: **6** · Risiko: **18** (R-01…R-18 di TECH-RISK-MASTER)
- Dokumen yang dihasilkan: 8 file `00-Global/` + 12 ADR + 7 file per-chapter + report ini = **28 file**

## Chapter Mapping

| Chapter | File dibuat | Endpoints/Tabel utama | Source Hint |
|---------|-------------|------------------------|-------------|
| BAB-1-Otentikasi-Akun | `SECURITY-RULES.md` | users, apiTokens; 10 aturan jalur auth | `lib/auth/*` (15 file), `app/login/`, `app/setup/` |
| BAB-2-Manajemen-Page | `SECURITY-RULES.md` | pages, pageAccessCodes; 7 host rules + 7 access rules | `proxy.ts`, `app/[slug]/password-actions.ts` |
| BAB-3-Link-Grup | `SECURITY-RULES.md` | links, linkGroups; ownership pattern + known gap | `app/dashboard/actions.ts`, `backup-actions.ts` |
| BAB-4-Tema-Profil | `SECURITY-RULES.md` | themes; injection surface + upload limits | `theme-actions.ts`, `settings-actions.ts` |
| BAB-5-Publik-Analytics | `PERFORMANCE-NOTES.md` | analyticsEvents, activityLogs; force-dynamic + aggregation cost | `app/[slug]/`, `lib/db/analytics.ts` |
| BAB-6-Integrasi-Automasi | `API-ENDPOINTS.md` + `INTEGRATION-DETAIL.md` | 8 endpoint, 6 integrasi, 6 cron | `app/api/**`, `lib/{webhooks,youtube-live,content-feeds,update-check}` |
| 00-Global | 8 master docs + 12 ADR | semua 15 tabel, stack, security, infra, risk | `lib/db/schema.ts`, `Dockerfile`, `.github/` |

> Catatan scope: BAB-3/BAB-4 diminta eksplisit masuk setelah rencana awal membuangnya; keduanya diisi SECURITY-RULES (konten nyata dari kode, bukan template kosong).

## Confidence

### High (langsung dari kode)
- Seluruh stack & versi (lockfile + `Dockerfile`)
- 15 tabel + enum + unique constraint (grep `.index(`/`.unique(` menyeluruh — hasil: 0 index non-unique, 4 unique)
- 8 endpoint + scope enforcement (kutipan per route)
- 6 cron jadwal + handler (`instrumentation.ts:50-89`)
- 10 aturan auth BAB-1 + 7 host rules `proxy.ts`
- 12 ADR evidence (semua `file:baris`)
- Scan negatif: tanpa Redis, tanpa test framework, tanpa Sentry, tanpa `.env*`, tanpa Dependabot, tanpa hardcoded secret, tanpa state mgmt library

### Medium (inferred — dilabeli di ADR)
- Alasan bisnis di balik sebagian besar ADR (custom auth, SQLite, GHCR, in-process queue) — ditulis `⚠️ INFERENSI` + "alasan tidak terdokumentasi di kode" (kecuali ADR-002/005/007/008/010 yang punya **komentar eksplisit** = fakta)
- License per-package belum diekstrak per entry (TECH-STACK: klaim Medium, sengaja)
- ISSUE.md #5 (domainValue duplikat) — tetap Medium, butuh runtime (statis review)

### Low / bukan hasil ekstraksi
- Seluruh § Appendix A (Migration Feasibility) — **riset internet eksternal**, bukan observasi kode

## Contradictions

| # | Lokasi A | Lokasi B | Konflik | Dampak | Status ADR |
|---|----------|----------|---------|--------|------------|
| 1 | `CLAUDE.md:14` ("Validation: Zod") + `package.json:36` (zod ^4.6.2) | 0 import zod di seluruh source; validasi manual (`integrations-actions.ts:12`) | Klaim vs kenyataan | Dep mati; onboarding menyesatkan | **ADR-011 Recovered-Contradictory** |
| 2 | `token-creator.tsx:17-20` (UI tawarkan 4 scope) | `integrations-actions.ts:12` (whitelist 2 scope) vs route minta `discord:*` (`discord-widgets/[id]/route.ts:16,42`) | Scope tak bisa dibuat | Endpoint discord-widgets selalu 403 — fitur mati (R-07) | bukan ADR (bug fungsional — ISSUE.md #1) |
| 3 | Arsitektur kode: native module (`Dockerfile:16-19`) + cron in-process (`instrumentation.ts:50-89`) + upload ke disk (`lib/images/storage.ts:7`) + master secret file (`master-secret.ts:11`) | Premis produk "ringan / deploy 1 command" & pertanyaan migrasi serverless (diskusi) | App terkunci ke Node runtime + filesystem persistent — mustahil ke Vercel/Edge tanpa redesign 4-6 subsistem | Menutup jalur "Rp0 serverless" untuk arsitektur saat ini; bukan bug, tapi fakta deployment-context | ADR-001/004/005/007 terkait (lihat Appendix A) |
| 4 | Komentar `unlock-rate-limit.ts:8-11` (key "WAJIB dari domain terbatas") | `app/login/actions.ts:31,129` (key = username input bebas) | Aturan file dilanggar file yang sama | Memory growth terbatas window (ada cleanup) → Low (R-09) | ADR-010 |

> **Status:** dokumen final sebagai representasi kode. Konteks manusia (jika diberikan) dicatat di ADR terkait sebagai `Human Context (optional)` — bukan approval gate. Promosi `Recovered` → `Accepted` hanya lewat `architect-grill-with-docs`.

## What Was NOT Found

Yang dicari tapi TIDAK ADA di kode (biar grill tahu apa yang perlu ditanya ke manusia):

| Item | Bukti scan negatif | Implikasi |
|------|--------------------|-----------|
| Test framework + test signifikan | 1 file test; `CLAUDE.md:33` mengakui "nanti" | R-01 |
| Step test/lint/audit di CI | `release.yml` = build+push saja | R-03 |
| Dependabot/Renovate | `.github/` = 1 workflow | R-04 |
| `.env.example` / validasi config | glob `.env*` = 0; zod 0 import | R-14 |
| Backup DB otomatis / RPO/RTO / DR plan | tidak teramati | keputusan operasional manusia |
| Cache layer (Redis dkk) | 0 dep | by-design? → grill |
| Rate limit per API token / IP-based | scan 6 route + proxy = nihil | ADR-010 |
| CSP / sanitasi HTML untuk customCss & privacyPolicyContent | header CSP tidak teramati | perlu penelusuran lanjutan |
| `docs/arsitektur-link-in-bio.md` (dikutip komentar `instrumentation.ts:48`) | **tidak ada di repo** (yang ada `docs/arsitektur-link-in-bio.md` disebut CLAUDE.md tapi path berbeda — sudah jadi temuan analyst, extraction-report.md §5.4) | dokumen desain di luar repo? |
| npm audit / CVE status | tidak dijalankan (butuh instalasi) | R-16 |
| Webhook inbound, email/SMTP, payment | 0 dep terkait | by-design tanpa layanan eksternal |
| Akun expiry, session list per device, audit trail login | tidak teramati | fitur belum ada |

## Appendix A — Migration Feasibility (Riset EKSTERNAL — bukan observasi kode)

> ⚠️ **LABEL PENTING:** appendix ini bukan bagian dari ekstraksi skill (skill melarang riset internet selain CVE). Ini hasil diskusi/riset web terpisah, dicatat agar tidak hilang. Sumber: dokumentasi Vercel/Cloudflare/Upstash/Turso + studi kasus publik (dicek 2026-10-02).

**Pertanyaan:** bisa-tidak app ini di-host serverless (Vercel/Edge) dengan hanya mengganti DB + cron?

**Temuan (angka dari dokumentasi resmi saat dicek):**

1. **DB → Turso/D1**: jalur paling bersih karena sudah Drizzle (driver libSQL/D1 resmi). Tapi D1 hanya hidup di Cloudflare — di Vercel pilihannya Turso/Neon. Blocker turunan: `better-sqlite3` native → driver async (review `db.transaction` sync-callback di `persistBoard`), `instrumentation.ts` boot-migrate harus pindah ke deploy step.
2. **Cron**: Vercel Hobby = **maks 1× sehari** (resmi Vercel) → 4 dari 6 cron mati; Pro ($20/bln) = per-menit → webhook 10 detik tetap tak muat. QStash free = **1.000 pesan/hari** vs kebutuhan ±**10.181/hari** (hitungan dari 6 jadwal `instrumentation.ts`) → PAYG ≈$1–3/bln.
3. **Di luar "2 hal"**: upload ke disk (`storage.ts:7`) butuh Blob/R2; master secret file (`master-secret.ts:11`) butuh env/KV (multi-instance = 2FA rusak); rate-limit in-memory butuh Redis; setup token via `docker logs` (`setup-token.ts:9-11`) tak punya padanan.
4. **Kontradiksi identitas**: produk bernama "Self-Hosted" (README.id.md:7: "datanya 100% punya kamu sendiri") — pindah ke Vercel/D1 = data di cloud pihak ketiga → bukan refactor, tapi perubahan value proposition.
5. **Konteks pasar (studi kasus)**: LinkBreeze (referensi arsitektur project ini — `proxy.ts:12`), LinkStack, LittleLink-Server semuanya Docker+VPS; Linktree (1M+ views) tetap pakai container (Fargate) + Lambda hybrid. Belum ada link-in-bio open-source berfitur setara yang berjalan pure-edge.

**Kesimpulan appendix (opini berlabel, bukan fakta kode):** stack saat ini sudah sesuai kategori; opsi "edge murni" menuntut rewrite separuh backend + kehilangan janji self-host; opsi hemat realistis = irisan bertahap (halaman publik via CDN, dashboard tetap container) bila suatu saat ada data bahwa VPS jadi bottleneck.

---

**Audit lanjutan:** disarankan `/analyst-design-reviewer` Phase 0 di sesi terpisah untuk cek akurasi observasi (bukan mengubah status ADR).
