# ADR-001: SQLite + better-sqlite3 + Drizzle sebagai Data Layer Tunggal (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.
> Lihat Evidence untuk fakta, Inference untuk dugaan berlabel.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-001 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-002, ADR-004, ADR-005 |
| **Related Requirements** | — (proyek tanpa PRD; ERD-MASTER analyst = 15 entitas) |
| **Scope** | Global |
| **Chapter(s) Terdampak** | Global (semua BAB) |
| **Confidence** | High |

## Context (Teramati)

Seluruh data aplikasi (15 tabel) berada di satu file SQLite yang dibuka oleh satu koneksi sync tanpa pool. Tidak ada ORM kedua, tidak ada database server terpisah.

## Evidence (Fakta — WAJIB)

- Driver: `lib/db/index.ts:8` (`new Database(dbPath)`), path dari env `DATABASE_PATH` (`:6`, default `data/kitab-link.db`)
- Pragma: `lib/db/index.ts:9` (`journal_mode = WAL`), `:10` (`foreign_keys = ON`)
- ORM: `lib/db/index.ts:12` (`drizzle(sqlite, { schema })`); dialect `sqlite` di `drizzle.config.ts:5`
- Versi: `better-sqlite3@^13.0.3`, `drizzle-orm@^0.45.2` (dari `package.json:20,23`, terkunci di `package-lock.json`)
- Native compile jadi bagian build image: `Dockerfile:16-19` (`python3 make g++ ... node-gyp`)
- Migrasi: 36 file SQL `drizzle/0000–0035`, dijalankan saat boot (`instrumentation.ts:15-19`)
- Satu-satunya ORM — tidak ada Prisma/Mongoose/etc di `package.json:14-37`

## Inference (Dugaan — WAJIB dilabeli jika ada)

> ⚠️ INFERENSI — tidak ada ADR/dokumen di kode yang mengonfirmasi alasan.
> Kemungkinan dipilih demi self-host tanpa service database terpisah (konsisten dengan `docker-compose.yml` yang cuma 1 service + 1 volume). Alasan tidak terdokumentasi di kode.

## Alternatives (Hanya Jika Teramati di Kode)

Tidak ada alternatif teramati di kode (tanpa folder Prisma/ORM lain; tidak ada sisa file migrasi non-Drizzle).

## Consequences (Teramati Saja)

### Observed Positive
- 1 proses bisa melayani web + cron + queue tanpa koneksi antar-service (`instrumentation.ts:50-89`)
- Migrasi otomatis saat boot, server gagal start kalau schema rusak (`instrumentation.ts:5-11`)
- Native binding sync bikin `proxy.ts` bisa query DB langsung per-request (`proxy.ts:33-37`, komentar `:6-8`)

### Observed Negative / Risk Pointer
- FK tanpa index + agregasi analytics tanpa LIMIT → R-05 (`TECH-RISK-MASTER.md`)
- `domainValue` tanpa UNIQUE → R-08
- Driver sync + file DB = mengikat app ke Node runtime → kontradiksi deployment (report § Contradictions)

### Unknown
- Perilaku concurrent multi-instance (2 proses, 1 file WAL) — tidak teramati/diuji di kode.

## References (Kode Saja)

- `lib/db/index.ts` — koneksi
- `lib/db/schema.ts` — 15 tabel
- `drizzle.config.ts`, `instrumentation.ts:15-19` — migrasi
- `Dockerfile:16-19` — kompilasi native
