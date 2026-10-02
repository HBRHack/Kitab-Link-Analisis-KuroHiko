# Data Dictionary Excerpt — BAB-5 Halaman Publik & Analytics

> **Excerpt dari [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat definisi atribut dua tabel catatan: `analytics_events` (metrik views/klik halaman publik) dan `activity_logs` (riwayat perubahan dashboard dengan sumber aksi).

> **Scope excerpt ini:** analytics_events, activity_logs — lihat [`ERD-EXCERPT.md`](./ERD-EXCERPT.md) untuk relasi antar entitas.

## Entities

### analytics_events

Metrik views & klik.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id event |
| pageId | integer FK→pages | Yes | — | cascade delete | Page terkait |
| eventType | enum | Yes | — | `page_view`\|`link_click` | Jenis event |
| linkId | integer | No | null | **bukan FK** | Id link (null untuk page_view) |
| referrer | text | No | null | hostname saja, bukan URL penuh | Asal kunjungan |
| deviceType | enum | No | null | `mobile`\|`tablet`\|`desktop` | Deteksi User-Agent |
| country | text | No | null | dari header proxy (`cf-ipcountry`, dst) | Negara |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu event |

**Business Rules:**
- BR-045: `linkId` sengaja bukan FK — link dihapus tidak boleh menghapus riwayat klik.
- BR-046: `country` selalu null bila reverse proxy tidak mengirim header geo — aplikasi **tidak menebak** lokasi.
- BR-047: Pencatatan bersifat fire-and-forget (`.catch`), kegagalan insert tidak menggagalkan request.

---

### activity_logs

Riwayat perubahan per page.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id log |
| pageId | integer FK→pages | Yes | — | cascade delete | Page terkait |
| action | text | Yes | — | key generik | Jenis aksi |
| detail | text | No | null | — | Judul/nama terkait |
| source | enum | Yes | `dashboard` | `dashboard`\|`automation`\|`api` | Pemicu aksi |
| sourceLabel | text | No | null | null bila `dashboard` | Nama rule / token |
| createdAt | timestamp | Yes | `unixepoch()` | — | Waktu aksi |

**Business Rules:**
- BR-048: `detail` menyimpan data mentah (judul/nama), **pesan tampilan** dikomposisi dari kamus i18n agar tetap benar saat user ganti bahasa.

---

## Value Lists (Relevan)

### event_type analytics (analytics_events.event_type)

| Code | Label | Description |
|------|-------|-------------|
| page_view | Page view | Halaman publik dilihat |
| link_click | Link click | Link diklik (lewat `/r/{id}`) |

### device_type (analytics_events.device_type)

| Code | Label | Detection |
|------|-------|-----------|
| mobile | Mobile | `Mobi|iPhone|Android` |
| tablet | Tablet | `iPad|Android(non-Mobile)|Tablet` |
| desktop | Desktop | default / tidak terdeteksi |

### log_source (activity_logs.source)

| Code | Label | sourceLabel |
|------|-------|-------------|
| dashboard | Dashboard | null |
| automation | Automation | nama rule |
| api | API | nama token |

## Full Reference

> Definisi seluruh 15 tabel: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md).
