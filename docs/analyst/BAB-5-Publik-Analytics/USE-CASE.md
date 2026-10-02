# BAB-5 — Halaman Publik & Analytics

Chapter: `BAB-5-Publik-Analytics` · UC: 035–042

> Dokumen ringkas (Opsi A). Cakupan: render publik, proteksi password, redirect klik, OG image, upload, analytics, export CSV.

## Use Cases

| UC | Judul | Actor | Trigger | Ringkasan Alur | Entities | BR |
|----|-------|-------|---------|----------------|----------|----|
| UC-035 | Lihat halaman publik | Pengunjung | Buka `/{slug}` | Query page by slug → `notFound()` bila tidak ada → `generateMetadata` (canonical konsisten ke `/` bila primary) → render `PublicPageBody` (`force-dynamic`, tak di-cache) | pages | — |
| UC-036 | Buka via custom domain | Pengunjung | Buka `namamu.com` | `proxy.ts` cari page by `domainType=custom_domain` + `domainValue=host` → bila path admin → 404 → set `domainVerified=true` bila belum → rewrite `/` → `/{slug}`, `/privacy` → `/{slug}/privacy` | pages | BR-019, BR-020 |
| UC-037 | Unlock page berpassword | Pengunjung | Form password muncul | `unlockPage` → cek rate-limit `page:{pageId}` → verify `pages.passwordHash` → bila gagal dan input kosong-senang, coba kode akses aktif (`expiresAt > now`) satu per satu → sukses: `grantPageAccess` (cookie 30 hari) → redirect | pages, pageAccessCodes | BR-029 |
| UC-038 | Klik link (redirect + analytics) | Pengunjung | Klik kartu link | Buka `/r/{linkId}` → validasi numerik → lookup link → `recordLinkClick` (fire-and-forget) → `getLinkHref` → `NextResponse.redirect(new URL(href, getBaseUrl()), 302)` | links, analyticsEvents | BR-025, BR-045, BR-047 |
| UC-039 | Generate OG image | crawler / pengunjung | `/{slug}/og` | Cari page → fallback banner "Kitab Link" bila tak ada → render `ImageResponse` 1200×630 dengan warna theme → avatar WebP dikonversi PNG dulu (Satori tak support WebP) | pages, themes | — |
| UC-040 | Sajikan file upload | Pengunjung | Akses `/uploads/...` | Validasi tiap segmen: non-empty, tanpa `..`, tanpa slash → `resolveUploadPath` → `stat` → **stream** (bukan buffer) → Content-Type dari whitelist ekstensi | — | — |
| UC-041 | Catat page view | Sistem | Render publik | Insert `analytics_events` `page_view` dengan referrer host, device type, country (dari header proxy saja) | analyticsEvents | BR-046, BR-047 |
| UC-042 | Export analytics CSV | Operator | Dashboard → export | `requireSession` → scope `page`: `requireOwnedPage` → `getPageAnalytics`; scope `overview`: `getCrossPageAnalytics(userId)` → `buildAnalyticsCsv` → response dengan `Content-Disposition: attachment` | analyticsEvents | — |

### Alur kritis (ditulis lengkap)

#### UC-038: Klik Link — Main Flow

1. **Pengunjung** klik kartu link (href mengarah ke `/r/{linkId}`, bukan URL tujuan)
2. **System** parse `linkId` → bila bukan angka → 404
3. **System** cari link → bila tak ada → 404
4. **System** `recordLinkClick(...)` — insert analytics **tanpa await**, error hanya dicetak ke log
5. **System** hitung `href` final dari `getLinkHref(publicLink)` (termasuk UTM, tipe khusus)
6. **System** resolve ke absolute dengan `getBaseUrl()` (bukan `req.url`, karena di belakang reverse proxy `req.url` bisa balik ke host container)
7. **System** `NextResponse.redirect(..., 302)`

#### UC-040: Exception Flows

- **EF-1 Segmen mengandung `..` atau slash** → 404 sebelum menyentuh filesystem (guard anti path-traversal)
- **EF-2 File tidak ada** → `stat` gagal → 404
- **EF-3 Ekstensi tak dikenal** → `Content-Type: application/octet-stream`

## Notes

- Tidak ada tabel `page_views` terpisah — page view dan link click berbagi `analytics_events`.
- `country` null bila reverse proxy tak kirim header geo (`cf-ipcountry`, `x-vercel-ip-country`, `x-country-code`) — aplikasi tidak mengarang lokasi.
