# Process Flows — BAB-5 Halaman Publik & Analytics

Chapter ini memuat dua proses jalur publik: **render halaman `/{slug}` + catat page view** dan **klik link lewat redirect `/r/{linkId}`** — keduanya sumber data analytics.

> PF-009–PF-010 nomor global sequential. Format mengikuti `00-Global/NOTATION.md` (GLOBAL-00).

## PF-009: Klik Link — Redirect + Analytics

Proses setiap kali pengunjung mengklik kartu link: rekam event dulu, baru redirect 302 ke tujuan.

**Trigger:** Pengunjung klik kartu link di halaman publik (UC-038, BR-025).
**Goal:** `analytics_events` terisi `link_click` **sebelum** pengunjung berpindah ke tujuan.

**Actors:**
- Pengunjung: mengklik kartu link
- System: route `/r/{linkId}` (lookup + record + redirect)

### Flow (Flowchart — ASCII)

```
START
  |
  v
[1] Pengunjung klik kartu (href = /r/{linkId}, BUKAN URL tujuan)
  |
  v
[2] Parse linkId numerik? ---- TIDAK --> 404 --> END
  |
  YA
  v
[3] Lookup link ada? ---- TIDAK --> 404 --> END
  |
  ADA
  v
[4] recordLinkClick(...) — INSERT analytics TANPA await
    (fire-and-forget; error hanya dicetak ke log)
  |
  v
[5] Hitung href final = getLinkHref(publicLink)
    (termasuk UTM, tipe khusus: email/phone/copy/dst)
  |
  v
[6] Resolve absolute dengan getBaseUrl()
    (BUKAN req.url — di belakang reverse proxy req.url
     bisa balik ke host container)
  |
  v
[7] NextResponse.redirect(..., 302) --> END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Pengunjung | Klik kartu | `app/r/[linkId]/route.ts` |
| 2–3 | System | — | Validasi numerik + lookup; tak ada → 404 |
| 4 | System | — | `recordLinkClick` tanpa await (`route.ts:26`) — penerimaan by design (BR-046 mindset: analitik tak boleh memperlambat redirect) |
| 5–6 | System | — | `getLinkHref` → `getBaseUrl()` (`route.ts`) |
| 7 | System | — | 302 ke tujuan |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | `linkId` numerik | Step 3 | 404 |
| 3 | link ditemukan | Step 4 | 404 |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 4 | Insert analytics gagal (DB penuh dll) | Hanya `console.error`; redirect **tetap jalan** | Pengunjung tak terpengaruh; cek log |
| 5 | Link type `copy`/`accordion` dst. | href dihitung sesuai tipe (bukan selalu URL) | — |

---

## PF-010: Render Halaman Publik + Catat Page View

Proses render `/{slug}` (Server Component, `force-dynamic`) beserta pencatatan view yang terjadi di saat yang sama.

**Trigger:** Pengunjung membuka `/{slug}` (UC-035 + UC-041).
**Goal:** HTML terkirim + `analytics_events` `page_view` tercatat.

**Actors:**
- Pengunjung / crawler: membuka halaman
- System: route page + `PublicPageBody` + `recordPageView`

### Flow (Flowchart — ASCII)

```
START
  |
  v
[1] GET /{slug}
  |
  v
[2] Query page by slug (LIMIT 1)
    ---- tidak ada --> notFound() --> END (404)
  |
  ADA
  v
[3] page.passwordHash ada DAN cookie grant tak valid?
  +--- YA ---> [3a] tampilkan form unlock
  |            (proses unlock = PF-004 BAB-2) --> END (belum render konten)
  +--- TIDAK -+
              v
[4] generateMetadata (canonical; konsisten ke / bila primary)
  |
  v
[5] PublicPageBody — 4 query PARALEL (Promise.all):
      a. board (groups + links aktif)
      b. theme tokens aktif
      c. live status (badges)
      d. discord widgets aktif
  |
  v
[6] recordPageView(...) — INSERT fire-and-forget
    (referrer = hostname saja, device = regex UA,
     country = HANYA dari header proxy, null bila tak ada)
  |
  v
[7] Render HTML + <style> custom font/CSS milik owner
    (force-dynamic — tak pernah di-cache statis) --> END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 2 | System | — | Select by slug `.limit(1)` (`app/[slug]/page.tsx:44`); `notFound()` bila kosong |
| 3a | System | — | Form unlock → rujuk PF-004 (BAB-2) |
| 5 | System | — | `Promise.all` 4 query (`public-page-body.tsx:74-79`) |
| 6 | System | — | Insert `page_view` tanpa await (`public-page-body.tsx:68-72` → `lib/db/analytics.ts:10-14`) |
| 7 | System | — | `dynamic = "force-dynamic"` (`page.tsx:12`) — data bisa berubah kapan saja |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | slug ditemukan | Step 3 | `notFound()` (404) |
| 3 | berpassword + belum grant | 3a (form unlock) | Step 4 (render) |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 6 | Insert view gagal | Fire-and-forget — tak mempengaruhi render | — |
| 5 | Widget Discord / YouTube down | Query tertentu kosong/null → tampil tanpa elemen itu | Fallback bawaan per komponen |

### Process Relationship Map

```
PF-010 (Render) --bila berpassword--> PF-004 (Unlock, BAB-2)
PF-010 (Render kartu) --triggers--> PF-009 (Klik link /r)
PF-009 + PF-010 --menulis--> analytics_events (UC-041 export CSV)
```
