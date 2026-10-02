# Extraction Report — Kitab Link

**Tanggal:** 2026-10-02
**Sumber:** clone `https://github.com/Kurohiko-id/Kitab-Link-Self-Hosted`
**Mode:** Opsi A — ringkas-esensial, functional, untuk konteks **self-hosted**
**Peran:** Reviewer saja · **Kode tidak diubah, tidak ada rencana perbaikan** (perbaikan REJECT)

---

## 1. Deteksi Framework & Language

```
Language   : TypeScript
Framework  : Next.js 16.3.5 (App Router, Server Components) + React 19.2.8
Database   : SQLite (better-sqlite3, WAL mode) + Drizzle ORM 0.45
Styling    : Tailwind CSS 4 + shadcn/ui
Auth       : Cookie-based HMAC session + argon2id + TOTP 2FA
Scheduler  : node-cron (in-process)
Validation : Manual (zod terpasang tapi 0 import — lihat BUG-004)
Project    : Fullstack (dashboard admin + halaman publik)
Depth      : Quick / esensial
```

Pembacaan utama: `lib/db/schema.ts`, `app/**/actions.ts` (64 server action), `app/api/**` (10 route file), `instrumentation.ts` (6 cron), `proxy.ts`, `lib/auth/*`, `lib/db/*`, `README.id.md`, `CLAUDE.md`.

---

## 2. Ringkasan Ekstraksi

| Aspek | Hasil |
|---|---|
| Entities (tabel) | **15** |
| Server Actions | **64** |
| API Routes | **10 file / 13 handler** (GET·PATCH·POST) |
| Public Routes | `/{slug}`, `/{slug}/og`, `/{slug}/privacy`, `/r/{linkId}`, `/uploads/*` |
| Roles / Actors | **3**: Operator (admin), Pengunjung, Sistem eksternal (API token) |
| Business Rules terdokumentasi | **48** (BR-014 – BR-048 + reserve block) |
| Use Cases | **54** (UC-001 – UC-054) |
| Cron jobs | **6** |
| Integrasi keluar | Discord widget, YouTube live, RSS/Atom, Webhook (HMAC), GitHub Releases, OG-image fetch |
| Temuan bug | **5** (1 High, 1 Medium, 3 Low) |
| Security issues | **0 Critical / 0 High** · 4 Low · 2 info-by-design · 1 unknown (SCA) |
| Dependencies | 22 direct, transitive belum dihitung, CVE belum dicek |
| Gate #6 | ✅ **PASS** (8 pemeriksaan value consistency) |

---

## 3. Chapter Structure (diinfer dari kode)

| Chapter | Nama | Use Cases | Entities | Source Hint |
|---------|------|-----------|----------|-------------|
| BAB-1-Otentikasi-Akun | Otentikasi & Akun | UC-001 – UC-009 | users, apiTokens | `app/login/`, `app/setup/`, `lib/auth/`, `app/dashboard/totp-actions.ts` |
| BAB-2-Manajemen-Page | Manajemen Page | UC-010 – UC-017 | pages, pageAccessCodes | `app/dashboard/page-actions.ts`, `app/[slug]/password-actions.ts`, `proxy.ts` |
| BAB-3-Link-Grup | Link & Grup | UC-018 – UC-026 | links, linkGroups | `app/dashboard/actions.ts`, `app/dashboard/social-links-actions.ts`, `app/dashboard/backup-actions.ts` |
| BAB-4-Tema-Profil | Tema & Profil | UC-027 – UC-034 | themes | `app/dashboard/theme-actions.ts`, `settings-actions.ts` |
| BAB-5-Publik-Analytics | Halaman Publik & Analytics | UC-035 – UC-042 | analyticsEvents, activityLogs | `app/[slug]/`, `app/r/[linkId]/`, `app/uploads/`, `lib/db/analytics.ts` |
| BAB-6-Integrasi-Automasi | Integrasi & Automasi | UC-043 – UC-054 | webhooks, webhookQueue, scheduledRules, liveBadges, discordWidgets, contentFeeds | `app/api/v1/`, `instrumentation.ts`, `lib/webhooks.ts`, `lib/scheduled-rules.ts` |
| 00-Global | Lintas modul | — | semua 15 entitas | `lib/db/schema.ts` |

> Struktur ini **belum dikonfirmasi ulang oleh stakeholder** (sesi berjalan tanpa tanya-balik). Bila ada chapter yang mau digabung/dipisah, cukup pindahkan baris di tabel ini — penomoran UC/BR sudah diblok per chapter di `NUMBERING-LOG.md` sehingga aman ditata ulang.

---

## 4. Confidence Levels

### 🟢 High — langsung dari kode

- Seluruh struktur 15 tabel, kolom, enum, dan FK/relasi (`lib/db/schema.ts`)
- 64 server action + pola otorisasinya (`requireSession` / `requireOwnedPage`)
- 10 API route + mekanisme scope token
- 6 cron schedule + job masing-masing (`instrumentation.ts`)
- BUG-001 (scope discord tak bisa dibuat) — whitelist vs kebutuhan route, hitam di atas putih
- BUG-002 (ownership `linkId`/`groupId` tak divalidasi) — klausa `where(eq(x.id, ...))` tanpa `pageId`
- BUG-003 (rate-limit key dari input bebas) — bertentangan dengan komentar di file yang sama
- BUG-004 (`zod` 0 import) — hasil grep seluruh repo
- Gate #6 seluruh 8 pemeriksaan — literal value vs Value List
- Kualitas kontrol keamanan: argon2id param, `timingSafeEqual`, guard traversal, enkripsi TOTP, atomic secret write

### 🟡 Medium — inferensi, mungkin perlu konfirmasi

- **BUG-005** (`domainValue` tanpa UNIQUE) — disimpulkan dari skema + `.limit(1)` di `proxy.ts`, **belum diverifikasi runtime**
- Penomoran BR/FR/UC — diinferensi dari grouping kode, bukan dari dokumen kebutuhan asli (proyek tidak punya PRD)
- Asumsi "single-user" — dari `CLAUDE.md` + `setupFirstAdmin` yang hanya jalan saat DB kosong; skema sendiri mendukung multi-user
- Severity self-hosted (Low/Medium) — penilaian saya terhadap konteks deployment yang kamu berikan
- License tiap dependency di SBOM — berdasarkan pengetahuan umum, belum dibaca dari lockfile
- Beberapa FR diklasifikasi Must/Should/Could — prioritas disusun dari ketergantungan kode, bukan dari stakeholder

### 🔴 Low — perkiraan, mohon diverifikasi

- Pola "duplicated logic" di `app/api/v1/**` (blok Bearer+scope diulang 5×) — mungkin disengaja demi keterbacaan
- Eksposur `THEME_INIT_SCRIPT` (`app/layout.tsx:40`) dan `firefly-overlay.tsx:101` (dua `dangerouslySetInnerHTML` sisanya) — **belum ditelusuri penuh**, di luar cakupan Opsi A
- Jumlah transitive dependency & CVE status — `package-lock.json` belum diparse, `npm audit` tidak dijalankan

---

## 5. Items Needing Confirmation

1. **Struktur chapter** di §3 — sudah sesuai, atau mau digabung jadi lebih sedikit?
2. **BUG-005** — perlu dijalankan (2 page, domain sama) untuk memastikan perilaku duplikat.
3. **Prioritas FR** (Must/Should/Could) — apakah urutannya mencerminkan keinginan stakeholder?
4. **`docs/arsitektur.md`** disebut di `CLAUDE.md` tapi tidak ada di repo (yang ada `docs/arsitektur-link-in-bio.md`) — apakah ada dokumen terpisah di luar repo?
5. **Scope audit keamanan** — apakah dua `dangerouslySetInnerHTML` yang tersisa mau ditelusuri (di luar Opsi A)?

---

## 6. Hasil Validation Gate

| # | Pemeriksaan | Status | Catatan |
|---|---|---|---|
| 1 | Redundancy (kolom duplikat FK + denormalisasi) | ✅ PASS | Tidak ada kolom denormalisasi; semua FK murni |
| 2 | BR vs Dict (aturan inferensi vs validasi kode) | ✅ PASS | BR dengan confidence Medium/Low ditandai di `bug-report.md`; tidak ada yang ditulis sebagai kepastian |
| 3 | Matrix completeness (BR ↔ REQUIREMENTS-MATRIX) | ✅ PASS | Seluruh BR-014–BR-048 muncul di `00-Global/REQUIREMENTS-MATRIX.md` |
| 4 | UC overlap (endpoint masuk 2 UC) | ✅ PASS | `/api/v1/links` hanya di UC-043/044/045; tak ada endpoint ganda. `discord-widget-config` dicatat di BAB-5 catatan, bukan UC terpisah |
| 5 | Numbering (blok di NUMBERING-LOG) | ✅ PASS | `NUMBERING-LOG.md` dibuat di sesi ini (sebelumnya tidak ada — bukan menggantikan legacy) |
| 6 | **Value Consistency (Gate #6)** | ✅ **PASS** | 8 pemeriksaan — rincian di `bug-report.md` §Gate #6 |

> **Status akhir gate: ✅ PASS — semua hijau, tidak ada WARNING yang perlu persetujuan.**

---

## 7. Deliverables

```
docs/analyst/
├── 00-Global/
│   ├── NOTATION.md                    ← GLOBAL-00 matriks pemilihan notasi
│   ├── ERD-MASTER.md                  ← 15 entitas, box art ASCII + relationship summary
│   ├── DATA-DICT-MASTER.md            ← atribut + Value Lists + data flow
│   ├── REQUIREMENTS-MATRIX.md         ← FR-001–FR-090 traceability
│   └── GLOSSARY.md                    ← 54 istilah
├── BAB-1-Otentikasi-Akun/USE-CASE.md      ← UC-001–009
├── BAB-2-Manajemen-Page/USE-CASE.md       ← UC-010–017
├── BAB-3-Link-Grup/USE-CASE.md            ← UC-018–026
├── BAB-4-Tema-Profil/USE-CASE.md          ← UC-027–034
├── BAB-5-Publik-Analytics/USE-CASE.md     ← UC-035–042
├── BAB-6-Integrasi-Automasi/USE-CASE.md   ← UC-043–054
├── NUMBERING-LOG.md                 ← register nomor UC/FR/BR
├── bug-report.md                    ← 5 temuan + Gate #6
├── security-report.md               ← penilaian posture self-hosted + manifest SCA
├── extraction-report.md             ← file ini
```

**Yang sengaja TIDAK dibuat** (di luar cakupan Opsi A):
- `SRS-MASTER.md` terpisah (aktor & integrasi sudah tercakup di REQUIREMENTS-MATRIX + per-chapter USE-CASE)

> **Pembaruan (sesi lanjutan):** tiga item di atas yang tadinya dilewati kini **sudah dibuat** — `PROCESS-FLOW.md` per chapter (PF-001–012) + `00-Global/PROCESS-FLOW-MASTER.md`, `ERD-EXCERPT.md` + `DATA-DICT-EXCERPT.md` per chapter, dan `business-rules.md` (BR-014–048). Seluruh isi diverifikasi dari kode. Yang tetap tidak dibuat: `SRS-MASTER.md` terpisah.

---

## 8. Tahap Berikutnya (belum dikerjakan)

Menurut `analyst-extract-code` Phase 7b, dokumen extraction masih perlu **audit independen** via `/analyst-design-reviewer` Phase 0 di sesi terpisah sebelum dianggap final.

**Status saat ini:** gate ✅ · **audit Phase 0 ⏳ belum dijalankan** → dokumen ini belum "final" dalam arti skill.

Selain itu, sesuai kesepakatan:
- ❌ Tidak ada perubahan kode
- ❌ Tidak ada rencana/perbaikan — seluruh temuan **dokumentasi saja**
- ❌ `npm audit` tidak dijalankan (akan mengubah isi direktori kerja)
