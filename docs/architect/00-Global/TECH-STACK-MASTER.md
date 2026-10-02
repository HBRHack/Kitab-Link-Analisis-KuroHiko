# Tech Stack Master (Recovered from Code): Kitab Link Self-Hosted

> ⚠️ RECOVERED — hasil observasi kode pada 2026-10-02, sumber `Kitab-Link-Self-Hosted/` (clone `https://github.com/Kurohiko-id/Kitab-Link-Self-Hosted`, versi `0.9.0`).
> Bukan rekomendasi. Alasan pilihan hanya ditulis jika ada bukti di kode; selebihnya dilabeli INFERENSI atau "alasan tidak terdokumentasi di kode".

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 0.9.0 (`package.json:3`) |
| **Last Updated** | 2026-10-02 |
| **Status** | Recovered |
| **Source** | `Kitab-Link-Self-Hosted/` (kerjaan lokal, bukan git checkout analisis) |
| **Depth** | deep |

## Stack Overview (Factual)

| Layer | Technology | Version (lockfile) | Evidence | Confidence |
|-------|-----------|---------------------|----------|------------|
| Runtime | Node.js | 22 (image `node:22-slim`) | `Dockerfile:13` | High |
| Package manager | npm | lockfile `package-lock.json` | lockfile ada; `Dockerfile:22` (`npm ci`) | High |
| Language | TypeScript | ^5 (`typescript` devDep) | `package.json:51`, `tsconfig.json:7` (`strict: true`) | High |
| Framework | Next.js (App Router) | 16.3.5 (exact pin) | `package.json:25`, `proxy.ts:6-8` (Next 16 rename middleware→proxy) | High |
| UI | React | 19.2.8 (exact pin) | `package.json:29-30` | High |
| Styling | Tailwind CSS 4 + shadcn/ui | ^4 / style `base-nova` | `package.json:40,50`, `components.json:2-4` | High |
| Database | SQLite via better-sqlite3 | ^13.0.3 | `package.json:20`, `lib/db/index.ts:8` | High |
| ORM | Drizzle ORM | ^0.45.2 | `package.json:23`, `lib/db/index.ts:12`, `drizzle.config.ts:5` | High |
| Cache | — (tidak ada cache layer) | — | tidak ada dep redis/ioredis di `package.json:14-37` | High |
| Auth | Custom (argon2id + HMAC cookie) | `@node-rs/argon2` ^2.2.1 | `package.json:19`, `lib/auth/password.ts:8-13`, `lib/auth/session.ts:10-11` | High |
| Cron | node-cron (in-process) | ^4.6.0 | `package.json:26`, `instrumentation.ts:40,50` | High |
| Deploy | Docker multi-stage → GHCR | `node:22-slim` | `Dockerfile:13,25,45`, `.github/workflows/release.yml:47-53` | High |

## Layer-by-Layer Evidence

### 1. Runtime & Language

**Observed:** Node.js 22, npm, TypeScript 5 strict.

- Evidence: `Dockerfile:13` (`FROM node:22-slim AS deps`); `package-lock.json` (npm); `tsconfig.json:7` (`"strict": true`), `:3` (`target: ES2017`), `:11` (`moduleResolution: bundler`), `:14` (`jsx: react-jsx`), `:21-23` (paths `@/* → ./*`).
- Factual Note: tidak ada `engines` di `package.json` — versi Node dipatok lewat base image Docker, bukan constraint npm.
- Inference: ⚠️ INFERENSI — pemilihan `node:22-slim` (Debian/glibc) bukan Alpine **dieksplisitkan di komentar** `Dockerfile:6-12`: better-sqlite3 selalu compile dari source, dan di Alpine (musl) node-gyp ambil header dari mirror tidak resmi yang "lebih gampang lemot/timeout". Ini fakta komentar, bukan inferensi.

### 2. Framework / Router

**Observed:** Next.js 16 App Router, `output: "standalone"`.

- Evidence: folder `app/` dengan layout/page/route.ts (bukan `pages/`); `next.config.ts:6` (`output: "standalone"`); `proxy.ts:6-8` (komentar: "Next.js 16 ganti nama middleware.ts -> proxy.ts"); Server Actions `"use server"` di `app/dashboard/theme-actions.ts:1` dll.
- Factual Note: `experimental.serverActions.bodySizeLimit: "8mb"` (`next.config.ts:22`), `experimental.cpus: 1` (`next.config.ts:32`), `outputFileTracingIncludes` untuk `./drizzle/**/*` (`next.config.ts:39-41`).
- Inference: alasan `cpus: 1` & body limit **ada di komentar config** (`next.config.ts:16-19, 25-31`): batasi puncak RAM build di VPS kecil + izinkan upload foto HP >1MB.

### 3. Database / ORM

**Observed:** SQLite (better-sqlite3) + Drizzle, WAL, foreign keys ON.

- Evidence: `lib/db/index.ts:8` (`new Database(dbPath)`), `:9` (`journal_mode = WAL`), `:10` (`foreign_keys = ON`), `:12` (`drizzle(sqlite, { schema })`); env key `DATABASE_PATH` (`lib/db/index.ts:6`) — nama key saja, tanpa value.
- Factual Note: koneksi dibuka sekali saat module load, **sync, tanpa pool**. Detail di `DATA-ARCH-MASTER.md`.
- Inference: alasan pilih SQLite (bukan Postgres) **tidak terdokumentasi di kode** — INFERENSI: kemungkinan demi self-host tanpa service DB terpisah (konsisten dengan `docker-compose.yml` yang cuma 1 service + 1 volume).

### 4. Cache Layer

**Observed:** TIDAK ADA cache layer terstruktur (tanpa Redis/Memcached/`unstable_cache`).

- Evidence: `package.json:14-37` tanpa dep cache; grep `unstable_cache` = 0 match di source.
- Yang dipakai sebagai pengganti: cache in-process per module — `lib/youtube-live.ts:145` (`channelIdCache = new Map<string,string>()`) dan Next fetch cache `next: { revalidate: 60 }` di `lib/discord-widget.ts:34`.
- Factual Note: rate-limit juga in-memory Map (`lib/auth/unlock-rate-limit.ts:15`) — bukan cache, tapi pola state serupa.

### 5. Auth

**Observed:** Custom auth — argon2id + stateless HMAC cookie, tanpa library auth eksternal.

- Evidence: `lib/auth/password.ts:8-13` (Argon2id, 19456 KiB, t=2, p=1); `lib/auth/session.ts:5` (cookie `kitab_session`), `:10-11` (payload `userId.expires` + HMAC sign), `:6` (TTL 7 hari); `lib/auth/hmac.ts:5` (`createHmac("sha256", getMasterSecret())`).
- Factual Note: **0 import** `next-auth`/`@clerk`/`@supabase`/`jsonwebtoken` di source — seluruh alur di `lib/auth/*` (15 file).
- Inference: alasan bangun custom auth **tidak terdokumentasi sebagai keputusan**; komentar `lib/auth/password.ts:3-4` hanya menjelaskan pemilihan parameter OWASP.

### 6. Cron / Background

**Observed:** node-cron in-process di `instrumentation.ts`, 6 job, tanpa queue external.

- Evidence: `instrumentation.ts:40` (`import("node-cron")`), jadwal di `:50, :58, :67, :75, :81, :87`; guard runtime `:3` (`NEXT_RUNTIME !== "nodejs"` → return).
- Factual Note: auto-migrate di-await saat boot (`instrumentation.ts:15-19`) sebelum server menerima request.
- Detail jadwal & dampak → `INFRA-CLOUD-MASTER.md` + `TECH-RISK-MASTER.md`.

### 7. Frontend State & UI

**Observed:** Tanpa state library eksternal — Server Actions + React 19 `useActionState`/`useTransition`.

- Evidence: grep `zustand|redux|jotai|react-query|createContext` di source = 0 (hanya match transitive di `package-lock.json`); `useActionState` di `app/login/login-form.tsx:3,13-14`, `app/dashboard/slug-form.tsx:3,13` dll; 54 file `"use client"` (baris 1, semua `*.tsx`).
- UI kit: shadcn/ui 10 komponen di `components/ui/*` (`components.json:15-21`), icon `lucide-react` (`package.json:24`), drag-drop `@dnd-kit` (`package.json:16-18`, dipakai `app/dashboard/board.tsx:16`).
- Fonts: 10 Google Fonts via `next/font/google` (`lib/fonts.ts:1-12`, semua `preload: false` di `:14-21`).

### 8. Testing (§11)

**Observed:** 1 file test, runner bawaan Node, TANPA framework test.

- Evidence: `package.json:10` (`"test": "node --import tsx --test lib/**/*.test.ts"`); file test **hanya** `lib/youtube-live.test.ts` (glob `**/*.test.ts` = 1 hasil); tanpa jest/vitest/playwright config.
- Factual Note: `tsx` **bukan** devDependency langsung — dia masuk `node_modules` sebagai transitive `drizzle-kit` (`package-lock.json` drizzle-kit → `tsx: ^4.21.0`). `CLAUDE.md:33` menulis `npm run test # (isi sesuai testing framework yang dipakai nanti)`.
- → dicatat sebagai RISIKO di `TECH-RISK-MASTER.md` (bukan info netral).

### 9. CI/CD & SBOM (§8, §13 pointer)

**Observed:** Satu workflow: build+push image GHCR multi-arch. Tanpa step test/lint/audit.

- Evidence: `.github/workflows/release.yml:5-9` (trigger tag `v*`), `:47-53` (build-push `linux/amd64,linux/arm64`), tanpa step `npm test`/`npm audit` (tidak ada di file).
- SBOM: `npm audit` **tidak dijalankan** selama ekstraksi ini (butuh instalasi dependensi) → status CVE = "belum dicek", lihat `TECH-RISK-MASTER.md`.

## Compatibility (Teramati Saja)

| Component A | Component B | Evidence Kompatibel Jalan | Notes |
|-------------|-------------|---------------------------|-------|
| Next.js 16.3.5 | React 19.2.8 | `package.json:25,29` (pin versi berdampingan), build di `Dockerfile:42` | Teramati jalan bersama (image build punya langkah `npx next build`) |
| better-sqlite3 13 | Drizzle 0.45 | import `drizzle-orm/better-sqlite3` di `instrumentation.ts:15`, driver `lib/db/index.ts:8,12` | Teramati dipakai bersama |
| proxy.ts (Next 16) | better-sqlite3 native | `proxy.ts:6-8` komentar: runtime Node (bukan Edge) → native binding aman | Keputusan eksplisit di kode |
| node-cron | standalone output | `instrumentation.ts:2-3` guard `NEXT_RUNTIME` | Cron hanya hidup di Node runtime |

DILARANG mengisi compatibility dari pengetahuan umum — hanya dari bukti kode.

## License (dari manifest, bukan opini)

| Component | License | Source |
|-----------|---------|--------|
| Seluruh direct dependency (22 package) | Tabel per-package TIDAK diekstrak manual dalam sesi ini | Field `license` per entry ada di `package-lock.json` |
| Proyek sendiri | LICENSE ada di repo root (isi tidak dibaca dalam sesi ini) | `Kitab-Link-Self-Hosted/LICENSE` |

> Catatan: tidak ada komponen dengan lisensi copyleft yang teridentifikasi dari `package.json` (semua nama paket adalah paket npm umum), **tapi ini klaim Medium confidence** — verifikasi penuh butuh pembacaan `package-lock.json` per entry. GAGAL klaim High di sini sengaja, supaya grill bisa minta verifikasi.

## Risks Pointer

> Detail di `00-Global/TECH-RISK-MASTER.md`. Pointer factual:

- Test hampir nol (1 file) + `tsx` phantom dependency → risiko regresi.
- CI tanpa test/lint/audit → release ship tanpa pemeriksaan.
- `zod` ada di dependencies tapi 0 import (§ KNOWN-004 di report).
- Tanpa Dependabot/Renovate → CVE monitoring manual.
