# ADR-005: proxy.ts (Next 16 Request Hook) di Node Runtime untuk Host Resolution (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.
> Ini salah satu ADR dengan alasan **paling eksplisit** di kode (komentar developer).

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-005 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-001 |
| **Related Requirements** | BAB-2 (custom domain, FR/UC manajemen page) |
| **Scope** | Global |
| **Chapter(s) Terdampak** | BAB-2-Manajemen-Page (utama), BAB-1 (admin path protection) |
| **Confidence** | High |

## Context (Teramati)

Sebelum render, setiap request melewati `proxy.ts` yang meresolve Host header ke page custom domain, memblokir path admin, dan me-rewrite path ke slug.

## Evidence (Fakta — WAJIB)

- Komentar eksplisit `proxy.ts:6-8`: "Next.js 16 ganti nama middleware.ts -> proxy.ts ... proxy defaultnya jalan di Node.js runtime (bukan Edge), jadi better-sqlite3 (native binding, sync) aman dipakai langsung di sini."
- Resolve host: `proxy.ts:33-37` (query `pages` WHERE `domainType='custom_domain' AND domainValue=host` LIMIT 1)
- Blokir admin di custom domain → 404: `proxy.ts:42-44`; tujuan komentar `:10-13` (cross-ref arsitektur LinkBreeze: "Admin and auth stay on your main domain")
- Auto-verify DNS: `proxy.ts:49-51`
- Rewrite `"/"→"/{slug}"`, `"/privacy"→"/{slug}/privacy"`: `proxy.ts:53-58` (daftar path yang TIDAK boleh diprefix: komentar `:20-22`)
- Matcher: `proxy.ts:61-66` (kecualikan `_next/static`, `_next/image`, `favicon.ico`, `robots.txt`, `uploads`, `r/`)
- Tanpa cek access-code di proxy (dicek di aplikasi `app/[slug]/password-actions.ts`)

## Inference (Dugaan — WAJIB dilabeli jika ada)

- Alasan runtime Node **TIDAK perlu diinferensikan** — dikutip di Evidence (`proxy.ts:6-8`).
- Kenapa pakai hook tingkat request (bukan resolve host di tiap route): alasan tidak terdokumentasi; INFERENSI: agar semua route publik otomatis kena tanpa duplikasi logic.

## Alternatives (Hanya Jika Teramati di Kode)

- Tidak ada `middleware.ts` lama yang tersisa (sudah rename penuh ke `proxy.ts` — konsisten dengan upgrade Next 16).

## Consequences (Teramati Saja)

### Observed Positive
- Kebijakan admin-vs-custom-domain terpusat di 1 file, berlaku untuk semua path
- Satu query SQLite per request yang melewati matcher (path dikecualikan tidak diquery — komentar `proxy.ts:62-65`)

### Observed Negative / Risk Pointer
- `.limit(1)` tanpa UNIQUE `domainValue` → resolusi tidak deterministik (R-08)
- Ketergantungan ke runtime Node mempersulit migrasi Edge/serverless (kontradiksi deployment — report)

### Unknown
- Latensi query per-request di traffic tinggi — tidak terukur dalam sesi ini.

## References (Kode Saja)

- `proxy.ts:6-66`
- `app/[slug]/password-actions.ts` — access check di aplikasi
