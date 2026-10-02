# ADR Master (Recovered): Kitab Link Self-Hosted

> ⚠️ Semua ADR di sini berstatus **Recovered** — hasil observasi kode (2026-10-02), BUKAN keputusan yang divalidasi manusia.
> Promosi ke `Accepted` hanya boleh lewat `architect-grill-with-docs` dengan manusia berwenang.
> Nomor global sequential — jangan dipakai ulang.

## Index

| ADR | Judul | Status | Scope | Confidence |
|-----|-------|--------|-------|------------|
| [ADR-001](ADR/ADR-001-sqlite-better-sqlite3-drizzle.md) | SQLite + better-sqlite3 + Drizzle sebagai data layer tunggal | Recovered | Global | High |
| [ADR-002](ADR/ADR-002-multi-stage-docker-single-container.md) | Multi-stage Docker, satu container non-root, satu named volume | Recovered | Global | High |
| [ADR-003](ADR/ADR-003-prebuilt-ghcr-multiarch-distribution.md) | Distribusi via prebuilt GHCR image multi-arch + compose 1-service | Recovered | Global | High |
| [ADR-004](ADR/ADR-004-in-process-cron-node-cron.md) | 6 cron job in-process via node-cron di instrumentation.ts | Recovered | Global | High |
| [ADR-005](ADR/ADR-005-proxy-node-runtime-hook.md) | proxy.ts (Next 16 request hook) di Node runtime untuk host resolution | Recovered | Global | High |
| [ADR-006](ADR/ADR-006-custom-stateless-auth.md) | Custom auth: argon2id + stateless HMAC cookie + TOTP (tanpa library auth) | Recovered | BAB-1 | High |
| [ADR-007](ADR/ADR-007-master-secret-encryption.md) | Master secret file + scrypt per-purpose + AES-256-GCM untuk secret di DB | Recovered | BAB-1 | High |
| [ADR-008](ADR/ADR-008-token-via-stdout-no-smtp.md) | Setup/reset token dicetak ke stdout — tanpa SMTP | Recovered | BAB-1 | High |
| [ADR-009](ADR/ADR-009-bearer-scope-api-v1.md) | API v1 Bearer token `klk_` + scope SHA-256 hashed | Recovered | BAB-6 | High |
| [ADR-010](ADR/ADR-010-in-memory-rate-limit.md) | Rate-limit in-memory Map (tanpa Redis) | Recovered | BAB-1 | High |
| [ADR-011](ADR/ADR-011-manual-validation-zod-unused.md) | Validasi manual — zod terpasang tapi 0 import | **Recovered-Contradictory** | Global | High |
| [ADR-012](ADR/ADR-012-in-process-webhook-queue.md) | Webhook queue di tabel DB + worker in-process (tanpa queue external) | Recovered | BAB-6 | High |

## Statistik

- Total: **12 ADR** (11 `Recovered`, 1 `Recovered-Contradictory`)
- Contradictory: **ADR-011** (zod) — klaim `CLAUDE.md:14` vs bukti 0 import
- Kontradiksi non-ADR (deployment-context, scope UI, ownership gap) → `extraction-report-arch.md` § Contradictions
- Format: `ADR-RECOVERED-FORMAT.md` (skill architect-extract-code-with-docs)

## Nomor berikutnya

Gunakan **ADR-013** untuk ekstraksi/keputusan berikutnya. Jangan mengulang nomor.
