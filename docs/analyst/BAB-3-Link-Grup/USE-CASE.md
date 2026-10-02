# BAB-3 — Link & Grup

Chapter: `BAB-3-Link-Grup` · UC: 018–026

> Dokumen ringkas (Opsi A). Cakupan: CRUD link (10 tipe), grup, urutan board, icon sosial.

## Use Cases

| UC | Judul | Actor | Trigger | Ringkasan Alur | Entities | BR |
|----|-------|-------|---------|----------------|----------|----|
| UC-018 | Simpan link (tambah/edit) | Operator | Modal Link | `requireOwnedPage` → baca semua field form → whitelist `linkType` (10 nilai valid, selain itu fallback `url`) → tipe `file`: validasi ≤20MB, simpan ke `link-files/` → insert/update → `maybeFetchOgImage` bila `rich` tanpa thumbnail → `logActivity` | links | BR-022, BR-023, BR-026 |
| UC-019 | Hapus link | Operator | Aksi baris link | `requireOwnedPage` → select by `linkId` → delete → `deleteImage` thumbnail & content image → hapus file lokal bila tipe `file` → log | links | — |
| UC-020 | Tampilkan/sembunyikan link | Operator | Toggle | `requireOwnedPage` → update `isActive` → log `link_shown`/`link_hidden` | links | — |
| UC-021 | Tandai featured | Operator | Toggle | `requireOwnedPage` → update `featured` (tanpa log) | links | — |
| UC-022 | Kelola grup (buat/ubah nama/hapus) | Operator | Dashboard | `requireOwnedPage` → create: auto `orderIndex = count`; rename/delete: select by `groupId` → mutate → log | linkGroups | — |
| UC-023 | Toggle visibilitas grup | Operator | Toggle | `requireOwnedPage` → update `isVisible` → log `group_shown`/`group_hidden` | linkGroups | BR-021 |
| UC-024 | Simpan urutan board (drag & drop) | Operator | Selesai drag | `requireOwnedPage` → transaksi: tulis `orderIndex` grup → tulis `groupId`+`orderIndex` per link → tulis ungrouped (`groupId=null`) | links, linkGroups | — |
| UC-025 | Kelola icon sosial (baris atas/bawah) | Operator | Settings → social links | `requireOwnedPage` → baca link `displayStyle=icon` → `reconcilePosition` per posisi (keep/update/create/delete) → update flag `socialIconsShowTop/Bottom` di `profileJson` | links, pages | — |
| UC-026 | Backup & restore link+grup | Operator | Settings → backup | Export: `requireOwnedPage` → serialize (id/groupId/thumbnailPath **dipisahkan**) → JSON v3. Import: parse → cek `version===3` → **restore destruktif** (delete semua links+grup dulu) → re-insert dari JSON | links, linkGroups | — |

### Alur kritis (ditulis lengkap)

#### UC-018: Simpan Link — Main Flow

1. **Operator** isi form modal (title, url, tipe, grup, display style, dsb)
2. **System** `requireOwnedPage(pageId)` — verifikasi page milik sesi
3. **System** normalisasi `linkType` terhadap `VALID_LINK_TYPES`; nilai asing → fallback `url`
4. **System** bila tipe `file` dan ada upload → cek ≤ 20MB → `saveFile` ke `link-files/`
5. **System** validasi `title` dan `url` non-empty (bila gagal, return error)
6. **System** upload gambar opsional (thumbnail / content image) → `processImage` → WebP
7. **System** insert (link baru, `orderIndex = count`) ATAU update (link existing)
8. **System** bila `displayStyle=rich` dan belum ada thumbnail → `fetchOgImageBuffer(url)` → proses → simpan
9. **System** `logActivity(pageId, "link_created"/"link_updated", …)` → `revalidatePath("/dashboard")`

#### UC-018: Exception Flows

- **EF-1 File > 20MB** → Step 4 gagal → return `{error}` sebelum ada tulis ke DB
- **EF-2 Upload gambar bukan `image/*`** → `uploadLinkImage` throw → return `{error}` (belum ada partial write ke DB)
- **EF-3 Fetch OG image gagal** → di-catch → link tetap tersimpan tanpa thumbnail (rich card tampil tanpa gambar)

### Catatan pengamatan (dokumentasi saja)

> `requireOwnedPage(pageId)` memverifikasi **page**, tetapi `linkId`/`groupId` yang diterima tidak pernah dicek apakah benar milik page tersebut. Lihat `bug-report.md` **BUG-002**.
