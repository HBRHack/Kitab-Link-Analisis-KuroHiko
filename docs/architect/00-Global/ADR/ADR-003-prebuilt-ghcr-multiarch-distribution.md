# ADR-003: Distribusi via Prebuilt GHCR Image Multi-arch + Compose 1-Service (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-003 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-002 |
| **Related Requirements** | — |
| **Scope** | Global |
| **Chapter(s) Terdampak** | Global |
| **Confidence** | High |

## Context (Teramati)

User tidak build dari source — compose menarik image prebuilt dari GHCR; build-from-source tersedia sebagai opsi yang dikomentari. Rilis dibangun otomatis oleh GitHub Actions saat tag versi di-push.

## Evidence (Fakta — WAJIB)

- `docker-compose.yml:16` — `image: ghcr.io/kurohiko-id/kitab-link-self-hosted:latest`
- `docker-compose.yml:17` — `# build: .` (dikomentari); instruksi switch di komentar `:9-12`
- Update = `docker compose pull && docker compose up -d` (`README.id.md:58`; `docker-compose.yml:6-7`)
- CI: `.github/workflows/release.yml:5-9` trigger tag `v*` / manual dispatch
- Multi-arch: `release.yml:51` (`platforms: linux/amd64,linux/arm64`), QEMU+Buildx `:32-36`; alasan dikomentari `:3-4` ("VPS x86 maupun ARM (Raspberry Pi, Oracle ARM free tier)")
- Tag: semver + `latest` (`release.yml:43-45`)
- Installer alternatif: `install.sh` (curl|bash, auto Docker+Caddy) — komentar `install.sh:6-10`

## Inference (Dugaan — WAJIB dilabeli jika ada)

- Pilihan GHCR (gratis untuk public repo) + arm64 → INFERENSI: target user Indonesia dengan VPS murah/ARM gratis; **tidak ada comment yang mengonfirmasi target pasar**. Alasan pemilihan GHCR tidak terdokumentasi.

## Alternatives (Hanya Jika Teramati di Kode)

- Build-from-source via compose (`docker-compose.yml:17`) — opsional, bukan jalur default.
- Docker Hub / registry lain: tidak teramati.

## Consequences (Teramati Saja)

### Observed Positive
- User cukup `compose up -d` tanpa toolchain build (`README.id.md:32`)
- Release artefak identik untuk semua user

### Observed Negative / Risk Pointer
- Workflow **tanpa step test/lint/audit** dan tanpa layer cache → R-03 (report juga mencatat build QEMU dua arsitektur tiap tag)

### Unknown
- Frekuensi rilis / praktik versioning tag — tergantung aktivitas repo, tidak teramati dari kode.

## References (Kode Saja)

- `.github/workflows/release.yml`
- `docker-compose.yml`
- `install.sh`, `README.id.md:32-58`
