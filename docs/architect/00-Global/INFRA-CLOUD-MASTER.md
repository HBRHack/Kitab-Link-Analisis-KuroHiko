# Infra & Cloud Master (Recovered from Code): Kitab Link Self-Hosted

> ⚠️ RECOVERED — hasil observasi kode pada 2026-10-02. Section heading eksplisit: Infrastructure / Cloud / DevOps / Monitoring.

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 0.9.0 |
| **Last Updated** | 2026-10-02 |
| **Status** | Recovered |
| **Depth** | deep |

---

## Infrastructure

### Containerization (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Dockerfile | 3-stage: deps → builder → runner | `Dockerfile:13,25,45` |
| Base image | `node:22-slim` (Debian/glibc) — **BUKAN alpine**, alasan eksplisit di komentar | `Dockerfile:13,6-12` |
| Deps stage | `npm ci` + toolchain `python3 make g++` (compile better-sqlite3) | `Dockerfile:18-22` |
| Build | `npx next build` dengan `NODE_OPTIONS=--max-old-space-size=768` | `Dockerfile:41-42` (komentar `:36-40`: batasi heap biar container VPS tetangga gak starve) |
| Build DB stub | `DATABASE_PATH=/tmp/build.db` (top-level open saat build) | `Dockerfile:34` (komentar `:31-33`) |
| Runner | hanya `.next/standalone` + `.next/static` + `public` | `Dockerfile:62-64` |
| User | **non-root** `USER node`, data dir di-chown saat build | `Dockerfile:60,68` (komentar `:54-59`) |
| Expose/health | `EXPOSE 3000`; HEALTHCHECK via `node -e fetch('/api/health')` | `Dockerfile:70,74-75` |
| Entrypoint | `docker-entrypoint.sh` (fix HOSTNAME) → `node server.js` | `Dockerfile:79-80` |

### Compose (Teramati)

`docker-compose.yml` — **1 service, 1 volume, 25 baris**:

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Image | `ghcr.io/kurohiko-id/kitab-link-self-hosted:latest` (prebuilt) | `docker-compose.yml:16` |
| Build alternatif | `# build: .` dikomentari (opsi build dari source) | `docker-compose.yml:17` (komentar `:9-12`) |
| Port | `3000:3000` | `docker-compose.yml:18-19` |
| Volume | named volume `kitab-link-data:/app/data` | `docker-compose.yml:20-21,24-25` |
| Restart | `unless-stopped` | `docker-compose.yml:22` |
| Service lain (DB/cache/proxy) | **TIDAK ADA** — tanpa Postgres/Redis/nginx di compose | seluruh file |
| Env | tidak didefinisikan (semua default) | tidak teramati di file |

### Reverse proxy / TLS (Teramati — sebagian di luar repo)

- **Caddy** di-install & dikonfigurasi oleh `install.sh` (bukan file di repo): `install.sh:6-10` (komentar: deteksi VPS dengan app lain di belakang Caddy, idempotent), `README.id.md:46` ("Install & setup Caddy otomatis ... HTTPS, reverse proxy").
- `Caddyfile` / `nginx.conf` / `traefik` config di repo: **tidak teramati** (tidak ada file tersebut).
- App mendeteksi HTTPS via header `x-forwarded-proto` — `lib/auth/secure-cookie.ts:8-11`.

### Hosting target (Teramati)

- **VPS / server Linux mandiri** — deployment Docker (README.id.md:7,53; GLOSSARY "Self-hosted").
- `vercel.json` / `wrangler.toml` / `fly.toml` / `render.yaml`: **tidak ada** → tidak teramati niat deploy ke platform serverless.

## Cloud

### Registry & distribusi (Teramati)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Registry | GHCR (`ghcr.io`) | `.github/workflows/release.yml:16` |
| Tag | semver dari git tag + `latest` | `release.yml:43-45` (`type=semver,pattern={{version}}`) |
| Multi-arch | `linux/amd64` + `linux/arm64` via QEMU + Buildx | `release.yml:32-36,51` (komentar `:3-4`: VPS x86 + Raspberry Pi / Oracle ARM free tier) |
| Auth registry | `GITHUB_TOKEN` via `docker/login-action` | `release.yml:25-30` |

### Layanan cloud pihak ketiga (Teramati dari kode)

| Layanan | Direction | Evidence |
|---------|-----------|----------|
| `api.github.com` (releases) | outbound | `lib/update-check.ts:14-16` |
| `discord.com/api` (widget publik) | outbound | `lib/discord-widget.ts:5` |
| `googleapis.com/youtube/v3` | outbound | `lib/youtube-live.ts:115` |
| CDN font Google (`next/font/google` di-build time) | build-time | `lib/fonts.ts:1-12` |

> Storage/DB/email terkelola (S3, Turso, Resend, dst): **tidak teramati** — semua lokal (`data/`).

## DevOps

### CI/CD (Teramati)

Satu workflow: `.github/workflows/release.yml`

```
on: push tag "v*" | workflow_dispatch   (:5-9)
steps:
  checkout            (:23)
  login GHCR          (:25-30)
  QEMU + Buildx       (:32-36)
  metadata tags       (:38-45)
  docker/build-push   (:47-53)  context: . , platforms: amd64+arm64, push: true
```

**Fakta penting (scan negatif):**
- **Tidak ada step** `npm test` / `npm run lint` / `npm audit` di workflow (tidak ada di file).
- **Tidak ada** layer cache build (`cache-from`/`cache-to` tidak di-set) → build ulang penuh tiap tag, di 2 arch via QEMU.
- Workflow CI terpisah (PR check): **tidak ada** — hanya `release.yml` (glob `.github/workflows/*` = 1 file).
- Dependabot/Renovate config: **tidak ada** (`.github/dependabot.yml` tidak teramati).

### Installer (Teramati)

`install.sh` — one-command `curl | bash`:
- Interaktif dua bahasa ID/EN (`install.sh:46-55`, fungsi `t()`).
- Alasan eksplisit di komentar `install.sh:6-10`: manual butuh ~7 langkah (folder, edit compose, hapus port, tambah network, install Caddy, Caddyfile, reload) → script deteksi kondisi VPS, idempotent.
- Termasuk **install Docker otomatis** kalau belum ada (`README.id.md:45`).

### Run & Update (Teramati)

- Deploy: `docker compose up -d` (`README.id.md:32`).
- Update: `docker compose pull && docker compose up -d` — migrasi jalan otomatis saat container nyala (`docker-compose.yml:6-7`, `instrumentation.ts:15-19`).
- Logs: `docker logs <nama-container>` (`README.id.md:68`) — juga jalur cetak setup/reset token.

### Developer workflow (Teramati)

- `npm run dev` / `build` / `start` / `lint` / `test` / `db:seed` / `yt:check` — `package.json:6-12`.
- `next.config.ts:14` `allowedDevOrigins: ["192.168.x.x"]` — IP LAN dev hardcoded (dev-only effect).

## Monitoring (§10)

| Aspek | Teramati | Evidence |
|-------|----------|----------|
| Health endpoint | `GET /api/health` → `SELECT 1` → `{status:"ok"}` / 503 | `app/api/health/route.ts:7-13` |
| Container healthcheck | tiap 30s, timeout 5s, start-period 10s, retries 3 | `Dockerfile:74-75` |
| Logging | **console.log/console.error saja** — tanpa pino/winston | pola di `instrumentation.ts:19,37,51,91-96` dll; 0 dep logging di `package.json` |
| Log berisi secret? | setup token & reset token dicetak ke stdout (by design) | `lib/auth/setup-token.ts:9-11,16-17`; cross-ref `security-report.md:162` |
| Error tracking (Sentry/dll) | **tidak teramati** — 0 dep `@sentry/*` | `package.json:14-37` |
| Analytics produk (Vercel/PostHog/GA) | **tidak teramati** | idem |
| APM/tracing (OTel) | **tidak teramati** (file `instrumentation.ts` = hook Next.js untuk cron, BUKAN OTel instrumentation) | `instrumentation.ts:1-3` |
| Metrics endpoint | **tidak teramati** | — |

> Kosongnya Sentry/APM = **fakta untuk risk register** (`TECH-RISK-MASTER.md`), bukan vonis — untuk self-host single-user, console log ke container mungkin cukup (keputusan manusia, bukan observasi).

## Inference Summary

> ⚠️ INFERENSI — tanpa cache CI + QEMU arm64 **kemungkinan** ditoleransi karena release jarang (tag manual); tidak ada comment di workflow yang menjelaskan. Alasan tidak terdokumentasi di kode.

## Unknown

- RPO/RTO, strategi backup DB otomatis, disaster recovery: **tidak teramati di kode** (lihat report § What Was NOT Found).
- Monitoring uptime (UptimeRobot dkk): di luar repo.

## References

- `Dockerfile`, `docker-compose.yml`, `docker-entrypoint.sh`
- `.github/workflows/release.yml`
- `install.sh`, `README.id.md`
- `app/api/health/route.ts`
