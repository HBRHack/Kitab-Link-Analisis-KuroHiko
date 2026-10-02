# ADR-009: API v1 Bearer Token `klk_` + Scope SHA-256 Hashed (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-009 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-006, ADR-011 |
| **Related Requirements** | BAB-6 (UC-043–045: API token), scope token |
| **Scope** | BAB-6-Integrasi-Automasi |
| **Chapter(s) Terdampak** | BAB-6, BAB-3 (resource links) |
| **Confidence** | High |

## Context (Teramati)

API publik diproteksi Bearer token personal (dibuat via dashboard), dengan scope per endpoint; token mentah tidak pernah disimpan.

## Evidence (Fakta — WAJIB)

- Prefix + format: `klk_` + `randomBytes(24)` (48 hex) — `lib/auth/api-tokens.ts:6,15`
- Disimpan sebagai SHA-256 hash; komentar `:8-9`: "Token API bukan password (entropi tinggi), ... cukup di-hash pakai SHA-256 biasa (fast lookup by hash) — bukan bcrypt yang sengaja lambat" — `:10-12`
- Raw token cuma tampil sekali saat dibuat (komentar `:26-27`)
- Verify: prefix check `:33` → lookup by hash `:38` → return `{userId, scopes, name}` (`:41-42`)
- Scope storage: `apiTokens.scopesJson` (`lib/db/schema.ts:192`)
- Enforcement per endpoint: `links/route.ts:14` (`links:read`), `links/[id]/route.ts:17,44` (`links:read`/`links:write`), `discord-widgets/[id]/route.ts:16,42` (`discord:read`/`discord:write`)
- Ownership: `innerJoin pages + pages.userId` — `app/api/v1/links/route.ts:23-28`
- Activity log source `"api"` — `links/[id]/route.ts:67-68`
- UI scope options: `token-creator.tsx:17-20` (4 opsi)

## Inference (Dugaan — WAJIB dilabeli jika ada)

- Alasan SHA-256 (bukan bcrypt) **dikomentari** — fakta, bukan inferensi.
- Kenapa API v1 hanya GET/PATCH (tanpa create/delete): alasan tidak terdokumentasi; INFERENSI: dicatat sebagai keterbatasan implementasi (komentar `links/[id]/route.ts:8` menyebut use-case "polling status dari luar, mis. Stream Deck" — itu fakta untuk 2 endpoint).

## Alternatives (Hanya Jika Teramati di Kode)

- Tidak ada (OAuth/segmentasi scope lain tidak teramati).

## Consequences (Teramati Saja)

### Observed Positive
- DB breach tidak membocorkan token mentah (hanya hash)
- Scope kecil per endpoint, ownership terjamin lewat join

### Observed Negative / Risk Pointer
- **Kontradiksi scope discord:*** — UI menawarkan, whitelist create tidak mengizinkan → route selalu 403 (R-07, Contradictions #2)
- API v1 tanpa rate limit per-token (API-STANDARD §5)

### Unknown
- Masa berlaku token (expiry/revoke) — revoke flow tidak dibaca detail dalam sesi ini; tidak teramati kolom expiry di `schema.ts:185-196`.

## References (Kode Saja)

- `lib/auth/api-tokens.ts`
- `app/api/v1/**`, `app/dashboard/token-creator.tsx`, `app/dashboard/integrations-actions.ts`
