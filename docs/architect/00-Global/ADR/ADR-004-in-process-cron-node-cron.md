# ADR-004: 6 Cron Job In-Process via node-cron di instrumentation.ts (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-004 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-001, ADR-002, ADR-012 |
| **Related Requirements** | BAB-6 (UC-043–UC-054) |
| **Scope** | Global |
| **Chapter(s) Terdampak** | BAB-6-Integrasi-Automasi (utama), BAB-5-Publik-Analytics |
| **Confidence** | High |

## Context (Teramati)

Semua job berkala berjalan di proses Node yang sama dengan web server, dijadwalkan node-cron saat Next.js instrumentation hook berjalan.

## Evidence (Fakta — WAJIB)

- `instrumentation.ts:40` (`import("node-cron")`), guard runtime `:2-3` (`NEXT_RUNTIME !== "nodejs"` → return)
- Jadwal:
  - `*/10 * * * * *` webhook queue → `:50-54`
  - `*/2 * * * *` scheduled rules → `:58-62`
  - `*/2 * * * *` live badges → `:67-71`
  - `0 */6 * * *` update check (+ sekali saat startup `:74`) → `:75-77`
  - `*/15 * * * *` content feeds → `:81-83`
  - `0 3 * * *` dead links → `:87-89`
- Semua callback dibungkus `.catch` (evidence: tiap blok di atas; cross-ref kontrol #11 security-report)
- Komentar `instrumentation.ts:48-49`: "cron proses webhook_queue tiap ~10 detik, jalan di proses Next.js yang sama **(bukan service terpisah)**" — dan rujukan `docs/arsitektur-link-in-bio.md`
- Versi: `node-cron@^4.6.0` (`package.json:26`)
- Setup token dicetak saat boot juga (`instrumentation.ts:21-35`)

## Inference (Dugaan — WAJIB dilabeli jika ada)

- Komentar `:48-49` menyatakan DESAIN-nya (bukan service terpisah) — fakta. Alasan bisnisnya (mis. menghindari worker terpisah di self-host) **tidak terdokumentasi**; INFERENSI: konsisten dengan target 1-container (ADR-002).

## Alternatives (Hanya Jika Teramati di Kode)

- Tidak ada alternatif teramati (tanpa bullmq/pg-boss/cron terpisah; `package.json` tanpa dep queue/cron selain node-cron).

## Consequences (Teramati Saja)

### Observed Positive
- Zero infra tambahan — cukup 1 container
- Job gagal tidak merusak proses (`.catch` di semua job)

### Observed Negative / Risk Pointer
- Single-instance only: 2 container → job dobel (R-12; `bug-report.md:257` mengakui perilaku multi-instance di luar cakupan)
- Idle RAM permanen: cron + web dalam 1 proses (konteks diskusi deployment — report § Contradictions #3)
- Job webhook 10 detik & live-check 2 menit = frekuensi tinggi untuk platform serverless (relevan bila migrasi dipertimbangkan; angka: ±10.181 trigger/hari)

### Unknown
- Dampak restart container terhadap job yang sedang berjalan (in-memory schedule hilang) — tidak teramati handling antrian yang tertunda.

## References (Kode Saja)

- `instrumentation.ts:40-89`
- `lib/webhooks.ts:53`, `lib/scheduled-rules.ts:82`, `lib/live-badge-check.ts`, `lib/content-feeds.ts:124-125`, `lib/dead-links.ts:26-29`, `lib/update-check.ts`
