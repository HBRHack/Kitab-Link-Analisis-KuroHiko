# BAB-6 — Integrasi & Automasi

Chapter: `BAB-6-Integrasi-Automasi` · UC: 043–054

> Dokumen ringkas (Opsi A). Cakupan: API token + REST API, webhook, cron, YouTube live, RSS feed, Discord widget, live badge, update check.

## Use Cases

| UC | Judul | Actor | Trigger | Ringkasan Alur | Entities | BR |
|----|-------|-------|---------|----------------|----------|----|
| UC-043 | Baca daftar link via API | Sistem eksternal (Stream Deck dsb) | `GET /api/v1/links?pageId=` | Bearer token → `verifyApiToken` → cek scope `links:read` → `pageId` wajib → page harus milik `identity.userId` → balas 5 field ringkas | apiTokens, links, pages | BR-030 |
| UC-044 | Baca 1 link via API | Sistem eksternal | `GET /api/v1/links/:id` | Sama; lookup link dengan `innerJoin pages` + filter `pages.userId` (anti IDOR lintas akun) | links, pages | — |
| UC-045 | Toggle link via API | Sistem eksternal | `PATCH /api/v1/links/:id` | Scope `links:write` → ambil `pageId` dari link → body `{active}` atau kosong = toggle → update → `logActivity(source:"api", sourceLabel:namaToken)` | links | — |
| UC-046 | Baca status widget Discord | Sistem eksternal | `GET /api/v1/discord-widgets/:id` | Scope `discord:read` → `innerJoin pages` filter owner → balas id/name/isEnabled | discordWidgets | — |
| UC-047 | Toggle widget Discord | Sistem eksternal | `PATCH /api/v1/discord-widgets/:id` | Scope `discord:write` → body `{enabled}` atau kosong = toggle → log | discordWidgets | BR-042 |
| UC-048 | Daftar & cabut webhook | Operator | Settings → integrations | `requireSession` → insert `targetUrl` (trim) + `secret=randomBytes(24)`; delete by `webhookId` + `userId` | webhooks | BR-033 |
| UC-049 | Kirim test event | Operator | Settings → test | `requireSession` → `enqueueWebhookEvent(webhookId, {type:"test.ping",...})` → masuk antrean | webhookQueue | BR-034 |
| UC-050 | Proses antrean webhook | Cron `*/10s` | `instrumentation.ts` | Select `status=pending` AND `nextRetryAt <= now` → kirim POST + HMAC header → sukses: `sent`; gagal: `attempts++`, backoff 10s/30s/1m/5m/30m; `attempts>=5` → `failed` | webhookQueue, webhooks | BR-035, BR-036 |
| UC-051 | Automasi show/hide (YouTube live / jadwal mingguan) | Operator + Cron | Dashboard / `*/2m` | Buat rule (`configJson` per trigger) → cron `processScheduledRules` poll status → ubah `isVisible` target → `lastState` di-update tiap tick, `lastTriggeredAt` hanya saat berubah → log `source:"automation"` | scheduledRules, links, linkGroups | BR-037, BR-038, BR-039 |
| UC-052 | Kelola live badge | Operator + Cron | Dashboard / `*/2m` | `saveLiveBadgeAction` / `deleteLiveBadgeAction` (1 row per page) → cron `processLiveBadges` poll → update `isLive` + `videoUrl`, `lastLiveAt` hanya pada edge offline→live | liveBadges | BR-040 |
| UC-053 | Auto-post dari RSS/Atom feed | Operator + Cron | Dashboard / `*/15m` | `createContentFeedAction` (wajib `groupId`) → cron `processContentFeeds` parse feed → item lebih baru dari `lastItemGuid` → buat link `rich`/`pill` di paling atas grup → simpan GUID baru | contentFeeds, links | BR-043, BR-044 |
| UC-054 | Kelola widget Discord & cek update | Operator | Dashboard | `saveDiscordWidgetAction` (clamp width 200–600 / height 150–800) → render publik fetch `widget.json` live (revalidate 60s); `dismissUpdateAction` tandai `users.lastSeenAppVersion` | discordWidgets, users | BR-041, BR-042 |

### Alur kritis (ditulis lengkap)

#### UC-050: Proses Antrean Webhook — Main Flow

1. **Cron** (`*/10 * * * * *`) panggil `processWebhookQueue()`
2. **System** select antrean `pending` yang `nextRetryAt <= now`
3. **System** per baris: ambil row `webhooks` → bila hilang → set `failed`
4. **System** `fetch(targetUrl, POST)` dengan header `X-KitabLink-Signature` (HMAC-SHA256) dan `X-KitabLink-Event`, timeout 10 detik
5. **System** res.ok → set `sent`
6. **System** gagal → `attempts+1`; bila ≥ 5 → set `failed`; selain itu tulis `nextRetryAt = now + backoff[attempts-1]`

#### UC-051: Alternative Flows

- **AF-1: Channel belum live tapi sebelumnya `live`** → `lastState` berubah `live`→`offline` → target disembunyikan, `lastTriggeredAt` di-update
- **AF-2: Status sama dengan tick sebelumnya** → `lastCheckedAt` di-update saja; `lastTriggeredAt` tidak disentuh

## Notes

- Semua cron berjalan **dalam proses Next.js yang sama** (`instrumentation.ts`), bukan service terpisah — sesuai prinsip proyek.
- `discord:read`/`discord:write` **tidak pernah bisa dibuat** pada token saat ini (whitelist create hanya memuat 2 scope pertama) → UC-046/UC-047 praktis selalu 403. Lihat `bug-report.md` **BUG-001**.
