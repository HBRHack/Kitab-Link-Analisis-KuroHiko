# ADR-002: Multi-stage Docker, Satu Container Non-root, Satu Named Volume (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-002 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-001, ADR-003 |
| **Related Requirements** | GLOSSARY "Self-hosted — Deploy Docker 1 command" |
| **Scope** | Global |
| **Chapter(s) Terdampak** | Global |
| **Confidence** | High |

## Context (Teramati)

Aplikasi didistribusikan sebagai container Docker tunggal; SQLite + uploads + master secret berbagi satu folder `data/` yang di-mount sebagai named volume.

## Evidence (Fakta — WAJIB)

- 3 stage: `Dockerfile:13` (deps), `:25` (builder), `:45` (runner); base `node:22-slim`
- Alasan base glibc vs Alpine **dikomentari eksplisit** `Dockerfile:6-12` (better-sqlite3 compile dari source; mirror unofficial-builds.nodejs.org di musl "lebih gampang lemot/timeout")
- Build limit: `Dockerfile:41` (`--max-old-space-size=768`, komentar `:36-40` anti starve container tetangga), `next.config.ts:32` (`cpus: 1`)
- Runner cuma bawa standalone: `Dockerfile:62-64`
- Non-root dari awal: `Dockerfile:68` (`USER node`), data dir di-chown saat build `:60` (komentar `:54-59`: gak butuh privilege-drop di entrypoint)
- Healthcheck built-in: `Dockerfile:74-75` → `GET /api/health`
- Compose: 1 service + 1 named volume `kitab-link-data:/app/data` (`docker-compose.yml:16,20-21`)
- Satu folder `data/` untuk DB + gambar + secret: komentar `lib/images/storage.ts:5-6`, `lib/auth/master-secret.ts:11`

## Inference (Dugaan — WAJIB dilabeli jika ada)

- Komentar `Dockerfile:54-59` menjelaskan mekanisme (ownership volume), tapi **alasan bisnis "kenapa Docker" tidak terdokumentasi di kode** — INFERENSI: kemungkinan demi deploy 1-perintah untuk user non-dev (konsisten dengan `README.id.md:7` "Deploy 1 command lewat Docker").

## Alternatives (Hanya Jika Teramati di Kode)

- `README.id.md:86-90` menunjukkan jalur dev manual (`npm install` + `npm run dev`) — jalur dev, bukan alternatif deploy produksi.
- Tidak ada systemd unit / PM2 / non-Docker production path yang teramati.

## Consequences (Teramati Saja)

### Observed Positive
- Artefak runner kecil (hanya standalone) — security-report.md:155 menilai ✅
- Update = `compose pull && up -d`, migrasi jalan otomatis (`docker-compose.yml:6-7`)

### Observed Negative / Risk Pointer
- Base image + native toolchain = image lebih besar dari alpine (diterima, alasan di komentar `Dockerfile:6-12`)
- Container = eksposur `docker logs` berisi token (ADR-008) → akses log = akses root

### Unknown
- Ukuran image final tidak terukur dalam sesi ini.

## References (Kode Saja)

- `Dockerfile` — seluruh stage
- `docker-compose.yml` — service/volume
- `docker-entrypoint.sh` — ENTRYPOINT
