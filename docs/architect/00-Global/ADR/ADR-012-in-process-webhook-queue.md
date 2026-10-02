# ADR-012: Webhook Queue di Tabel DB + Worker In-Process (Recovered)

> ⚠️ RECOVERED — hasil observasi kode, bukan keputusan yang divalidasi manusia.

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-012 |
| **Status** | Recovered |
| **Date Observed** | 2026-10-02 |
| **Observer** | architect-extract-code-with-docs |
| **Related ADRs** | ADR-004 (cron 10 detik), ADR-001 (SQLite) |
| **Related Requirements** | BAB-6 (webhook outbound, UC integrasi) |
| **Scope** | BAB-6-Integrasi-Automasi |
| **Chapter(s) Terdampak** | BAB-6 |
| **Confidence** | High |

## Context (Teramati)

Event webhook diantre ke tabel `webhook_queue` lalu dikirim oleh worker yang berjalan sebagai cron in-process tiap ±10 detik, dengan retry backoff eksponensial — tanpa Redis/bullmq.

## Evidence (Fakta — WAJIB)

- Queue table: `webhookQueue` (`lib/db/schema.ts:215`)
- Enqueue: `lib/webhooks.ts:10-15` (insert `payloadJson`)
- Signature: HMAC-SHA256 — `lib/webhooks.ts:17-19` (`createHmac("sha256", secret)`)
- Delivery: POST `targetUrl` + header `X-KitabLink-Signature` & `X-KitabLink-Event`, timeout 10 detik — `lib/webhooks.ts:26-35`
- Retry: `MAX_ATTEMPTS = 5`, backoff `[10s, 30s, 60s, 5m, 30m]` — `lib/webhooks.ts:6-8,37-49`
- Worker: cron `*/10 * * * * *` → `instrumentation.ts:50-54`; komentar `:53` "Dipanggil cron tiap ~10 detik"
- Comment `instrumentation.ts:48-49` merujuk `docs/arsitektur-link-in-bio.md` (dokumen desain di repo)
- Penggunaan saat ini: `enqueueWebhookEvent` dipanggil dari `app/dashboard/integrations-actions.ts:66-70` (event `test.ping`); enum event lain (`subscriber.new`, `link.clicked`, `page.live_status_changed` — `schema.ts:205-207`) **belum dipanggil di kode** (fakta dari scan; i18n `lib/i18n.ts:373` menyebut "baru terpicu otomatis kalau fitur sumbernya sudah ada")

## Inference (Dugaan — WAJIB dilabeli jika ada)

- Kenapa tabel DB (bukan antrian eksternal): alasan tidak terdokumentasi; INFERENSI: konsisten dengan desain zero-infra (ADR-001/002/004).

## Alternatives (Hanya Jika Teramati di Kode)

- Tidak ada (bullmq/pg-boss/NATS tidak ada di `package.json`).

## Consequences (Teramati Saja)

### Observed Positive
- Antrean tahan restart (data di SQLite, bukan memori)
- Retry dengan backoff + status `sent`/`failed` terlacak di DB

### Observed Negative / Risk Pointer
- Latensi delivery ±10 detik minimum (jadwal cron)
- 3 dari 4 event enum belum ada pemanggil — fitur webhook sebagian belum aktif
- Bergantung pada proses yang sama dengan web (ADR-004 → R-12)

### Unknown
- Perilaku antrean saat ada >1 instance — tidak teramati (same as R-12).

## References (Kode Saja)

- `lib/webhooks.ts`
- `lib/db/schema.ts:198-231` (webhooks + webhook_queue)
- `instrumentation.ts:48-54`
