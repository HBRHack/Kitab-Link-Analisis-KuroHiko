# Data Dictionary Excerpt — BAB-3 Link & Grup

> **Excerpt dari [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md) — sumber kebenaran ada di master.**
> Jangan edit file ini manual. Update master lalu re-generate excerpt.

Chapter ini memuat definisi atribut entitas inti konten: `link_groups` (kontainer urutan) dan `links` (28 atribut — entitas terlebar di skema, mencakup 10 `linkType` yang semuanya berbagi kolom `url`).

> **Scope excerpt ini:** link_groups, links — lihat [`ERD-EXCERPT.md`](./ERD-EXCERPT.md) untuk relasi antar entitas.

## Entities

### link_groups

Grup kontainer untuk link; mendukung show/hide manual dan automasi.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id grup |
| pageId | integer FK→pages | Yes | — | cascade delete | Page pemilik |
| name | text | Yes | — | non-empty setelah trim | Nama grup |
| orderIndex | integer | Yes | 0 | — | Urutan tampil |
| isVisible | boolean | Yes | true | — | Show/hide manual |

**Business Rules:**
- BR-021: `is_visible` manual dan `scheduled_rules` bisa berjalan bersama — automasi menulis state-nya sendiri ke target.

---

### links

Entitas inti. Kolom `url` dipakai ulang untuk semua `linkType`.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | integer | Yes | autoincrement | PK | Id link |
| pageId | integer FK→pages | Yes | — | cascade delete | Page pemilik |
| groupId | integer FK→link_groups | No | null | set null saat grup dihapus | Grup induk |
| orderIndex | integer | Yes | 0 | — | Urutan dalam grup |
| title | text | Yes | — | non-empty | Judul |
| url | text | Yes | — | non-empty; makna bergantung `linkType` | Tujuan / nilai link |
| description | text | No | null | — | Deskripsi opsional |
| displayStyle | enum | Yes | `pill` | `pill`\|`rich`\|`icon`\|`image` | Gaya tampil |
| thumbnailPath | text | No | null | selalu hasil `processImage()` → WebP | Thumbnail |
| imageHideBorder | boolean | Yes | false | — | Override tema (khusus `image`) |
| imageHideBackground | boolean | Yes | false | — | Override tema (khusus `image`) |
| imageShowTitle | boolean | Yes | false | — | Tumpuk judul di gambar |
| imageShowContent | boolean | Yes | false | — | Tumpuk icon/emoji di gambar |
| imageContentPath | text | No | null | — | Gambar kecil tumpukan |
| imageRadius | integer | No | null | — | Radius override (px) |
| imageShadow | enum | Yes | `theme` | `theme`\|`none`\|`sm`\|`md`\|`lg` | Bayangan override |
| icon | text | No | null | `brand:*` / `generic:*` / null=auto | Icon tombol |
| scheduleStart | timestamp | No | null | — | Jendela tampil terjadwal |
| scheduleEnd | timestamp | No | null | — | Jendela tampil terjadwal |
| isActive | boolean | Yes | true | — | Show/hide manual |
| linkType | enum | Yes | `url` | 10 nilai (lihat Value Lists) | Cara render + href |
| featured | boolean | Yes | false | — | Tandai unggulan |
| utmSource | text | No | null | — | UTM param |
| utmMedium | text | No | null | — | UTM param |
| utmCampaign | text | No | null | — | UTM param |
| iconPosition | enum | Yes | `top` | `top`\|`bottom` | Posisi baris icon (khusus `icon`) |
| lastCheckedAt | timestamp | No | null | — | Cek link mati terakhir |
| lastCheckStatus | enum | No | null | `ok`\|`dead` | Hasil cek terakhir |
| consecutiveFailures | integer | Yes | 0 | threshold 2 | Gagal beruntun |

**Business Rules:**
- BR-022: Semua upload gambar wajib lewat `processImage()` (sharp → resize → WebP); file asli tidak pernah disimpan.
- BR-023: Tipe `file` menerima URL eksternal ATAU upload sendiri; upload menang bila keduanya diisi. Maks 20MB.
- BR-024: Link ditandai `dead` hanya setelah **2x gagal berturut-turut**; `consecutiveFailures` di-reset ke 0 tiap cek sukses.
- BR-025: Semua klik diarahkan lewat `/r/{linkId}` dulu (bukan langsung ke tujuan) supaya analytics terekam sebelum redirect 302.
- BR-026: `displayStyle` lama `rounded` dimigrasi ke `pill` — kolom plain text tanpa CHECK constraint DB, jadi perubahan union type tidak butuh migration SQL.

**Relationships:**
- links --N:1-- pages
- links --N:1-- link_groups (opsional, set null)

---

## Value Lists (Relevan)

### display_style (links.display_style)

| Code | Label | Description |
|------|-------|-------------|
| pill | Pill | Tombol rounded polos (default) |
| rich | Rich card | Kartu dengan thumbnail + deskripsi |
| icon | Icon | Ikon kecil, dua baris (`iconPosition` top/bottom) |
| image | Image | Gambar jadi "wajah" tombol |

### link_type (links.link_type)

| Code | Label | isi kolom `url` |
|------|-------|-----------------|
| url | URL biasa | URL tujuan |
| email | Email | Alamat email |
| phone | Telepon | Nomor telepon |
| whatsapp | WhatsApp | Nomor (auto-format) |
| file | File | Path upload lokal ATAU URL eksternal |
| embed | Embed | URL yang di-iframe |
| copy | Copy to clipboard | Teks bebas (kode promo, no. rekening) |
| accordion | Accordion list | JSON array `{label, value}` |
| countdown | Countdown | JSON `{endsAt, url}` |
| discord_widget | Discord widget | Id row `discord_widgets` sebagai string angka |

### last_check_status (links.last_check_status)

| Code | Label | Description |
|------|-------|-------------|
| ok | OK | Terakhir kali berhasil diakses |
| dead | Dead | Gagal 2x berturut-turut |

## Full Reference

> Definisi seluruh 15 tabel: [`00-Global/DATA-DICT-MASTER.md`](../00-Global/DATA-DICT-MASTER.md).
