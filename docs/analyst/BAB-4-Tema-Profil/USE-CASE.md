# BAB-4 — Tema & Profil

Chapter: `BAB-4-Tema-Profil` · UC: 027–034

> Dokumen ringkas (Opsi A). Cakupan: theme library, preset, profil page, custom CSS, settings akun, streamer mode.

## Use Cases

| UC | Judul | Actor | Trigger | Ringkasan Alur | Entities | BR |
|----|-------|-------|---------|----------------|----------|----|
| UC-027 | Pakai preset theme | Operator | Galeri preset | `requireOwnedPage` → duplicate `sourceTokens` jadi entry library baru (`uniqueThemeName`) → set `pages.themeId` → log `theme_changed` | themes, pages | — |
| UC-028 | Aktifkan theme dari library | Operator | Pilih theme | `requireOwnedPage` + `requireOwnedTheme(userId, themeId)` → update `pages.themeId` → log | themes, pages | — |
| UC-029 | Simpan perubahan theme | Operator | Editor theme | `requireSession` + `requireOwnedTheme` → `buildThemeTokensFromForm(current, formData)` (merge ke state lama) → update `tokensJson` → log `theme_edited` → revalidate publik | themes | — |
| UC-030 | Hapus theme dari library | Operator | Library | `requireSession` + `requireOwnedTheme` → delete → `pages.themeId` otomatis `set null` | themes | — |
| UC-031 | Import theme JSON | Operator | Settings → import | `requireOwnedPage` → baca `json` field ATAU file → `JSON.parse` → tolak bila bukan object → `parseThemeTokens` (filter token tak dikenal) → insert sebagai theme baru | themes | — |
| UC-032 | Simpan profil page + custom CSS | Operator | Settings → profil | `requireOwnedPage` → merge field → `customCss` di-`slice(0, 20000)` → update `profileJson` → log `profile_updated` / `css_updated` (berdasarkan `formData.has("customCss")`) | pages | — |
| UC-033 | Upload avatar / banner / OG / favicon | Operator | Settings → gambar | `requireOwnedPage` → validasi ukuran (OG ≤5MB, favicon ≤2MB) → `processImage`/`processFavicon` → WebP → simpan path ke `profileJson` | pages | BR-022 |
| UC-034 | Kelola akun & streamer mode | Operator | Sidebar → akun | `updateAccountAction` (displayName/avatar akun, terpisah dari profil page) / `toggleStreamerModeAction` / `changePasswordAction` (verify password lama dulu) | users | BR-014 |

### Alur kritis (ditulis lengkap)

#### UC-029: Simpan Theme — Main Flow

1. **Operator** ubah warna/font/tombol di editor
2. **System** `requireSession()` — verifikasi login
3. **System** `requireOwnedTheme(session.userId, themeId)` — verifikasi theme milik akun ini (bukan cuma page)
4. **System** baca token lama → `buildThemeTokensFromForm` mengisi hanya field yang ada di form (sisa token dipertahankan)
5. **System** update `name` (bila berubah) dan `tokensJson`
6. **System** `logActivity(pageId, "theme_edited", name)` → revalidate dashboard + `/[slug]`

> **Catatan:** UC-029 memakai `pageId` hanya untuk **logging** — tidak divalidasi lewat `requireOwnedPage`. Konsisten dengan pengamatan BUG-002.

## Notes

- Theme bersifat **library lintas page** (milik `userId`), bukan milik satu page. Pemakaian per page dicatat di `pages.themeId`.
- `customCss` disuntik ke halaman publik memakai `<style dangerouslySetInnerHTML>` — hanya bisa diisi oleh operator sendiri.
