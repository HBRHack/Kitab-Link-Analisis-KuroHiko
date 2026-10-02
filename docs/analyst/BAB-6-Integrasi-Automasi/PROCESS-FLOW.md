# Process Flows — BAB-6 Integrasi & Automasi

Chapter ini memuat dua proses kron berjalan di dalam proses Next.js yang sama (`instrumentation.ts`): **pemrosesan antrean webhook** (dengan retry/backoff) dan **automasi show/hide** (YouTube live / jadwal mingguan).

> PF-011–PF-012 nomor global sequential. Format mengikuti `00-Global/NOTATION.md` (GLOBAL-00): urutan langkah proses internal = Activity Mermaid; flow linear = ASCII.
>
> Semua cron di sini **in-process** (bukan service terpisah) — lihat `docs/architect/00-Global/ADR-004-in-process-cron-node-cron.md`.

## PF-011: Proses Antrean Webhook

Pengiriman event webhook keluar secara asinkron lewat tabel `webhook_queue` + retry berjenjang.

**Trigger:** Cron `*/10 * * * * *` memanggil `processWebhookQueue()` (UC-050, BR-035/BR-036).
**Goal:** Setiap event terkirim (status `sent`) atau menempuh 5 percobaan lalu `failed`.

**Actors:**
- Cron in-process (`instrumentation.ts:50-54`)
- System: `processWebhookQueue` + HTTP client
- Server tujuan (`targetUrl` milik user)

### Activity Diagram (Mermaid) — langkah + percabangan retry

```mermaid
activityDiagram-v2
    start
    :Cron */10 detik panggil processWebhookQueue();
    :Select antrean status=pending<br/>DAN nextRetryAt <= now;
    if (ada baris?) then (tidak)
        :selesai untuk tick ini;
        stop
    else (ya)
    endif
    :per baris: ambil row webhooks terkait;
    if (webhook row hilang?) then (ya)
        :set status=failed;
        stop
    else (tidak)
    endif
    :POST targetUrl, timeout 10 detik,<br/>header X-KitabLink-Signature (HMAC-SHA256)<br/>+ X-KitabLink-Event;
    if (res.ok?) then (ya)
        :set status=sent;
    else (tidak)
        :attempts = attempts + 1;
        if (attempts >= 5?) then (ya)
            :set status=failed (permanen);
        else (tidak)
            :nextRetryAt = now + backoff<br/>[10s, 30s, 1m, 5m, 30m];
        endif
    endif
    stop
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Cron | Tick tiap ~10 detik | `instrumentation.ts:50-54` (BR-036) |
| 2 | System | — | Select `pending` + `nextRetryAt <= now` (`lib/webhooks.ts`) |
| 4 | System | Fetch | POST + header HMAC (BR-033), timeout 10 detik (`webhooks.ts:26-35`) |
| 5–6 | System | — | `res.ok` → `sent`; gagal → backoff (`webhooks.ts:37-49`) |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 3 | ada baris jatuh tempo | proses per baris | selesai (tick kosong) |
| 4 | `webhooks` row hilang (FK cascade tak jalan) | `failed` | lanjut fetch |
| 6 | `res.ok` | `sent` | `attempts++` |
| 6b | `attempts >= 5` | `failed` permanen (BR-035) | jadwalkan `nextRetryAt` |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 4 | Server tujuan timeout / 5xx | attempts++ + backoff → setelah 5× = `failed` | Operator kirim ulang event / cek URL |
| 4 | Server tujuan 4xx (URL salah permanen) | Diperlakukan sama — tetap 5× lalu `failed` | Perbaiki `targetUrl`, buat event baru |
| — | Server app restart | Antrean aman di SQLite (bukan memori) | Lanjut dari `pending` berikutnya |

---

## PF-012: Automasi Show/Hide (Scheduled Rules)

Proses polling status (live YouTube / jadwal mingguan) lalu menampilkan-menyembunyikan target grup/link.

**Trigger:** Cron `*/2 * * * *` memanggil `processScheduledRules()` (UC-051, BR-037–BR-039).
**Goal:** `isVisible` target selalu mencerminkan status trigger terakhir; log `source:"automation"`.

**Actors:**
- Cron in-process (`instrumentation.ts:58-62`)
- System: `processScheduledRules`
- Data eksternal: YouTube (scraping / Data API) — lihat `docs/architect/BAB-6-Integrasi-Automasi/INTEGRATION-DETAIL.md` §1

### Flow (Flowchart — ASCII)

```
START
  |
  v
[1] Cron tiap 2 menit: processScheduledRules()
  |
  v
[2] Loop tiap scheduled_rules
  |
  v
[3] Baca status trigger:
    - youtube_live --> cek channel (live / offline / unknown)
    - weekly_schedule --> hari ini termasuk jendela? (mode show/hide, untilDate)
    - manual --> state statis dari configJson
  |
  v
[4] Status UNKNOWN (gagal fetch / fallback)?
  +--- YA ---> [4a] JANGAN ubah apa pun
  |            (rule diam; lastState lama dipertahankan) --> lanjut rule berikut
  |
  +--- TIDAK -+
              v
[5] newState == lastState?
  +--- YA ---> [5a] update lastCheckedAt SAJA
  |            (lastTriggeredAt TIDAK disentuh — AF-2) --> lanjut rule berikut
  |
  +--- BERUBAH -+
                v
[6] Update lastState = newState
    + lastTriggeredAt = now
  |
  v
[7] Set isVisible TARGET (group/link sesuai targetType)
    (live→offline = sembunyikan — AF-1; offline→live = tampilkan)
  |
  v
[8] logActivity(targetPage, ..., source:"automation",
    sourceLabel:nama rule) --> lanjut rule berikut --> END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Cron | Tick tiap 2 menit | `instrumentation.ts:58-62` |
| 3 | System | — | `configJson` per `triggerType` (BR-037): `{channelUrl}` / `{days, mode, untilDate}` |
| 4a | System | — | `unknown` → `null` → rule tak mengubah state (konsisten `scheduled-rules.ts:58`) |
| 5a | System | — | Hanya `lastCheckedAt` (BR-038, AF-2) |
| 6–8 | System | — | `lastState` + `lastTriggeredAt` + tulis `isVisible` + log `source:"automation"` |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 4 | status = `unknown` | 4a (diamkan) | Step 5 |
| 5 | `newState == lastState` | 5a (hanya `lastCheckedAt`) | Step 6 (transisi) |
| 7 | transisi `live → offline` | sembunyikan target | tampilkan target (offline→live) |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 3 | YouTube fetch gagal / kuota habis | Status `unknown` → rule diam (tak ada toggle palsu) | Cek `users.youtubeApiKey` / jaringan |
| 7 | Target `targetId` menunjuk objek yang sudah dihapus | `targetId` **bukan FK** (BR-039) — update jadi no-op tanpa error dari DB | Bersihkan rule mati dari dashboard |
| — | 2 instance cron berjalan (multi-container) | Eksekusi dobel — di luar cakupan desain (ADR-004) | Deploy 1 instance saja |

### Process Relationship Map

```
PF-011 (Webhook queue) --dipicu oleh--> event perubahan yang sama dengan PF-012
PF-012 (Show/hide) --menulis--> links/linkGroups.isVisible --> terlihat di PF-010 (render BAB-5)
PF-012 --log--> activity_logs source:"automation" (dashboard BAB-3)
```
