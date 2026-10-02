# ADR-011: Validasi Manual — zod Terpasang Tapi 0 Import (Recovered-Contradictory)

> ⚠️ RECOVERED-CONTRADICTORY — hasil observasi kode; ADA KONFLIK antara dokumentasi & kode.
> Lihat Evidence untuk fakta, Inference untuk dugaan berlabel.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-011 |
| **Status** | **Recovered-Contradictory** |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-009 |
| **Related Requirements** | — |
| **Scope** | Global |
| **Chapter(s) Terdampak** | Global (semua input path) |
| **Confidence** | High |

## Context (Teramati)

Seluruh input divalidasi manual (String()/trim/allowlist), padahal `zod` terpasang sebagai dependency dan dokumentasi menyebutnya.

## Evidence (Fakta — WAJIB)

**Sisi A — zod di-deklarasikan:**
- `package.json:36` — `"zod": "^4.6.2"` di `dependencies`
- `CLAUDE.md:14` — "Validation: Zod"

**Sisi B — zod tidak dipakai:**
- grep `from "zod"` / `import.*zod` di `app/`, `lib/`, `components/` = **0 import**
- Validasi aktual manual, contoh: `app/dashboard/actions.ts` pola `String(formData.get(...)).trim()`; allowlist `VALID_LINK_TYPES` (lihat ISSUE.md kutipan)
- `VALID_SCOPES` allowlist manual: `app/dashboard/integrations-actions.ts:12`

## Inference (Dugaan — WAJIB dilabeli jika ada)

> ⚠️ INFERENSI — tidak ada comment yang menjelaskan kenapa zod tidak jadi dipakai.
> Kemungkinan: rencana adopsi yang tertunda, atau dependensi sisa dari setup awal. **Alasan tidak terdokumentasi di kode.**

## Alternatives (Hanya Jika Teramati di Kode)

- Kedua "alternatif" justru hidup bersamaan: zod (di package.json) vs validasi manual (di kode) — inilah kontradiksinya.

## Consequences (Teramati Saja)

### Observed Positive
- Validasi manual bekerja tanpa bug fungsional yang dilaporkan dari jalur ini

### Observed Negative / Risk Pointer
- Dependency percuma ikut ter-bundle → biaya audit CVE & upgrade (ISSUE.md #4)
- Klaim `CLAUDE.md` tidak akurat terhadap kode → onboarding menyesatkan
- Inkonsistensi validasi antar-action (tiap action menulis pola sendiri)

### Unknown
- Apakah ada rencana adopsi zod — di luar pengetahuan kode.

## Human Context (Optional, hanya untuk kontradiksi)

> Belum ada konteks manusia yang diberikan untuk kontradiksi ini. Bila diberi, catat di sini sebagai info (bukan approval).

## References (Kode Saja)

- `package.json:36`, `CLAUDE.md:14`
- grep 0 import (hasil scan 2026-10-02)
- `docs/analyst/ISSUE.md` #4
