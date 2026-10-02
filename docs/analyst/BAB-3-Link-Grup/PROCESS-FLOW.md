# Process Flows — BAB-3 Link & Grup

Chapter ini memuat dua proses tulis paling padat: **simpan link** (validasi multi-field + upload) dan **persist board drag & drop** (transaksi multi-update urutan).

> PF-005–PF-006 nomor global sequential. Format mengikuti `00-Global/NOTATION.md` (GLOBAL-00): flowchart linear = ASCII; urutan langkah proses internal = Activity Mermaid.

## PF-005: Simpan Link (Tambah / Edit)

Proses validasi form modal link sampai baris `links` ter-insert/update + activity log.

**Trigger:** Operator submit modal Link (UC-018).
**Goal:** Baris link tersimpan valid (atau error jelas sebelum ada tulis parsial ke DB).

**Actors:**
- Operator: mengisi form (title, url, tipe, grup, display style, upload)
- System: server action `saveLinkAction` + storage gambar

### Flow (Flowchart — ASCII)

```
START
  |
  v
[1] Operator isi form modal link
  |
  v
[2] requireOwnedPage(pageId)? ---- TIDAK --> throw --> END
  |
  YA
  v
[3] Normalisasi linkType vs VALID_LINK_TYPES
    (10 nilai; nilai asing --> fallback "url")
  |
  v
[4] linkType = file DAN ada upload?
  +--- YA ---> [4a] cek ukuran <= 20MB
  |                 +--- LEBIH --> return error --> END (belum ada tulis DB)
  |                 [4b] saveFile ke link-files/ (randomUUID.ext)
  |
  +--- TIDAK -+
              v
[5] title & url non-empty? ---- TIDAK --> return error --> END
  |
  YA
  v
[6] Upload gambar opsional (thumbnail / content image)
    --> processImage (sharp) --> WebP
    (bukan image/* --> throw --> END, belum ada partial write)
  |
  v
[7] INSERT link baru (orderIndex = count)  ATAU  UPDATE existing
  |
  v
[8] displayStyle = rich DAN belum ada thumbnail?
  +--- YA ---> [8a] fetchOgImageBuffer(url) --> proses --> simpan
  |                 (gagal --> di-catch, link tetap tersimpan tanpa thumbnail)
  +--- TIDAK -+
              v
[9] logActivity("link_created"/"link_updated")
    + revalidatePath("/dashboard") --> END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 2 | System | — | `requireOwnedPage(pageId)` — guard level page (`app/dashboard/actions.ts:245-248`) |
| 3 | System | — | Whitelist `VALID_LINK_TYPES`; nilai asing fallback `url` (BR-026 mindset) |
| 4a–b | System | Upload file | Cek ≤ 20MB → `saveFile` `link-files/` (BR-023) |
| 6 | System | Upload gambar | `processImage()` sharp → WebP, file asli tak disimpan (BR-022) |
| 7 | System | — | Insert (`orderIndex = count`) atau update |
| 8a | System | — | `maybeFetchOgImage` — failure di-catch, tak membatalkan simpan |
| 9 | System | — | `logActivity` + revalidate |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | page milik sesi | Step 3 | throw (berhenti) |
| 4 | tipe `file` + ada upload | 4a (validasi ukuran) | Step 5 |
| 5 | title & url terisi | Step 6 | tolak |
| 8 | `rich` tanpa thumbnail | 8a (fetch OG) | Step 9 |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 4a | File > 20MB | Return `{error}` **sebelum** tulis DB | Kompres / upload ulang |
| 6 | Gambar bukan `image/*` | Throw → `{error}`, belum ada partial write | Upload gambar valid |
| 8a | Fetch OG image gagal | Di-catch — link tetap tersimpan tanpa thumbnail | Edit ulang / biarkan |
| — | `linkId`/`groupId` tak dicek kepemilikannya | Guard hanya level page | Lihat BUG-002 (pengamatan) |

---

## PF-006: Persist Board (Drag & Drop Urutan)

Penyimpanan urutan grup + link setelah operasi drag & drop di papan dashboard — **satu transaksi**.

**Trigger:** Operator selesai menyeret kartu/link di board (UC-024).
**Goal:** `orderIndex` grup, `groupId`, dan `orderIndex` link tersimpan konsisten dalam satu transaksi.

**Actors:**
- Operator: menyeret kartu lalu melepas
- System: `persistBoard` (server action, transaksi Drizzle)

### Activity Diagram (Mermaid) — urutan langkah proses internal

```mermaid
activityDiagram-v2
    start
    :Operator selesai drag, kirim peta board
    (grups + posisi link per grup);
    :requireSession() -- gagal --> redirect /login;
    :requireOwnedPage(pageId) -- gagal --> throw;
    :Buka transaksi Drizzle (db.transaction);
    :Tulis orderIndex tiap link_groups;
    :Tulis groupId + orderIndex tiap link
    (ikut grup baru / posisi baru);
    :Tulis link tanpa grup:
    groupId = null + orderIndex urut papan;
    :Commit transaksi;
    :revalidatePath dashboard + /[slug];
    stop
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Operator | Lepas kartu di board | Client kirim struktur urutan lengkap (grup + link per grup + ungrouped) |
| 2–3 | System | — | `requireSession()` → `requireOwnedPage(pageId)` (`app/dashboard/actions.ts:301`) |
| 4 | System | — | `db.transaction` — semua tulis atomic |
| 5–7 | System | — | Update `link_groups.orderIndex` → `links.groupId`+`orderIndex` → ungrouped (`groupId=null`) |
| 8–9 | System | — | Commit + revalidate |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | belum login | redirect `/login` (berhenti) | Step 3 |
| 3 | page bukan milik sesi | throw (berhenti, transaksi tak dibuka) | Step 4 |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 4 | Salah satu update gagal | Transaksi rollback — papan kembali ke keadaan sebelum drag | Coba drag ulang |
| 1 | Payload `groupId`/`linkId` dari client tak divalidasi per baris | Diterima apa adanya selama page cocok | Lihat BUG-002 (pengamatan) |

### Process Relationship Map

```
PF-005 (Simpan link) --triggers--> PF-006 (urutan baru tersimpan) saat link berpindah grup
PF-005 / PF-006 --menulis--> analytics & activity logs (BAB-5, source:"dashboard")
```
