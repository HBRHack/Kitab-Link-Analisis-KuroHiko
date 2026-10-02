# ADR-010: Rate-Limit In-Memory Map (Tanpa Redis) (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-010 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-006, ADR-004 |
| **Related Requirements** | BAB-1 (UC brute-force protection) |
| **Scope** | BAB-1-Otentikasi-Akun |
| **Chapter(s) Terdampak** | BAB-1 (utama), BAB-2 (page unlock) |
| **Confidence** | High |

## Context (Teramati)

Perlambatan percobaan login/unlock dilakukan dengan Map JavaScript di memori proses — tanpa tabel DB, tanpa Redis.

## Evidence (Fakta — WAJIB)

- Parameter: `MAX_ATTEMPTS = 8` (`lib/auth/unlock-rate-limit.ts:12`), `WINDOW_MS = 5 menit` (`:13`)
- Store: in-memory `Map<string, {count, windowStart}>` (`:15`)
- Komentar eksplisit `:1-5`: "sengaja gak butuh tabel/Redis ... reset kalau server restart ... upgrade ke store persisten kalau nanti jalan multi-instance"
- Aturan key (komentar `:8-11`): key "WAJIB dari domain terbatas (id numerik page/user) atau dibatasi eviction-nya... tanpa itu Map ini bisa tumbuh gak terbatas (memory leak beneran)"
- Cleanup per record: `cleanupExpired()` di `recordFailedAttempt` (`:17-22, :35`)
- Pemakai: `app/login/actions.ts:31` (`login:{username}`), `:129` (`reset-request:{username}`), `app/[slug]/password-actions.ts:30` (`page:{id}`)
- Tanpa rate-limit berbasis IP (scan: tidak teramati di `proxy.ts` / route mana pun)

## Inference (Dugaan — WAJIB dilabeli jika ada)

- Alasan tanpa Redis **dikomentari** (`unlock-rate-limit.ts:1-5`) — fakta: kesadaran single-instance + rencana upgrade bila multi-instance.

## Alternatives (Hanya Jika Teramati di Kode)

- Redis/tabel DB: disebut di komentar (`:1-5`) sebagai sesuatu yang sengaja TIDAK dipakai — alternatif teramati tapi ditolak secara eksplisit.

## Consequences (Teramati Saja)

### Observed Positive
- Tanpa infra tambahan; cleanup menjaga ukuran Map proporsional ke attacker aktif (`:17-22`)

### Observed Negative / Risk Pointer
- **Key `login:{username}` melanggar aturan file-nya sendiri** (username = input bebas) → R-09 (Contradictions terkait: komentar vs kode)
- Reset per restart server → jendela proteksi ter-reset
- Tanpa rate-limit IP → bypass berganti username (ISSUE.md #3)

### Unknown
- Efektivitas nyata terhadap distribusi brute-force — tidak diuji.

## References (Kode Saja)

- `lib/auth/unlock-rate-limit.ts`
- `app/login/actions.ts:31,129`, `app/[slug]/password-actions.ts:30`
