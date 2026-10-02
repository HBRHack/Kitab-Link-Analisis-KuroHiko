# Process Flows — BAB-4 Tema & Profil

Chapter ini memuat dua proses simpan utama: **edit tema** (library lintas page) dan **simpan profil page + custom CSS** (input yang disuntik ke halaman publik).

> PF-007–PF-008 nomor global sequential. Format mengikuti `00-Global/NOTATION.md` (GLOBAL-00).

## PF-007: Simpan Perubahan Theme

Proses merge token tema dari form editor ke `themes.tokensJson` lalu me-refresh tampilan publik.

**Trigger:** Operator submit editor tema (UC-029).
**Goal:** `tokensJson` ter-update tanpa menghilangkan token yang tidak disentuh form; halaman publik re-render.

**Actors:**
- Operator: mengubah warna/font/tombol di editor
- System: `saveThemeAction` (requireSession + requireOwnedTheme)

### Flow (Flowchart — ASCII)

```
START
  |
  v
[1] Operator ubah token di editor, klik simpan
  |
  v
[2] requireSession()? ---- TIDAK --> redirect /login --> END
  |
  YA
  v
[3] requireOwnedTheme(session.userId, themeId)?
    (theme milik akun INI, bukan cuma page) ---- TIDAK --> throw --> END
  |
  YA
  v
[4] Baca token lama dari DB
  |
  v
[5] buildThemeTokensFromForm(current, formData)
    --> merge: HANYA field yang ada di form yang berubah,
        sisa token dipertahankan
  |
  v
[6] Update name (bila berubah) + tokensJson
  |
  v
[7] logActivity(pageId, "theme_edited", name)
    (pageId di sini HANYA untuk logging) --> revalidate dashboard + /[slug] --> END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 2 | System | — | `requireSession()` (`app/dashboard/theme-actions.ts:39` dst.) |
| 3 | System | — | `requireOwnedTheme(userId, themeId)` — guard owner theme |
| 5 | System | — | `buildThemeTokensFromForm(current, formData)` — **merge**, bukan replace (`theme-actions.ts` UC-029) |
| 7 | System | — | `logActivity("theme_edited")` + `revalidatePath` dashboard & `/{slug}` |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | session valid | Step 3 | redirect `/login` |
| 3 | theme milik user | Step 4 | throw (berhenti) |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 5 | Field form tak dikenal | Diabaikan (hanya field valid yang merge) | — |
| — | `pageId` hanya untuk logging (tak lewat `requireOwnedPage`) | Konsisten BUG-002 | Lihat `bug-report.md` |

---

## PF-008: Simpan Profil Page + Custom CSS

Proses update `pages.profileJson` — termasuk `customCss` yang nantinya disuntik `<style dangerouslySetInnerHTML>` ke halaman publik.

**Trigger:** Operator submit Settings → Profil (UC-032).
**Goal:** Profil ter-update; `customCss` dibatasi 20.000 karakter; halaman publik segar.

**Actors:**
- Operator: mengisi bio, avatar/banner, custom CSS
- System: `updateProfileAction` (requireOwnedPage + merge profileJson)

### Flow (Flowchart — ASCII)

```
START
  |
  v
[1] Operator ubah field profil / customCss, klik simpan
  |
  v
[2] requireOwnedPage(pageId)? ---- TIDAK --> throw --> END
  |
  YA
  v
[3] formData punya field "customCss"?
  +--- YA ---> [3a] customCss = slice(0, 20000)  ← batas keras
  |            [3b] tandai aksi = "css_updated"
  +--- TIDAK -+  (tanpa field customCss: CSS lama TIDAK disentuh)
              v
[4] Merge field profil baru ke state profileJson lama
    (field kosong tak menghapus isi lama kecuali by design)
  |
  v
[5] UPDATE pages SET profileJson = ...
  |
  v
[6] logActivity("profile_updated" ATAU "css_updated")
    (bergantung formData.has("customCss"))
  |
  v
[7] revalidatePath dashboard + /[slug] --> END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 2 | System | — | `requireOwnedPage` (`app/dashboard/settings-actions.ts:78-102`) |
| 3a | System | — | `customCss.slice(0, 20000)` — dipotong, bukan error |
| 6 | System | — | Pilihan log berdasarkan `formData.has("customCss")` (`settings-actions.ts`) |
| 7 | System | — | Revalidate — `public-page-body.tsx` menyuntik `<style dangerouslySetInnerHTML>` saat render berikutnya |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | page milik sesi | Step 3 | throw |
| 3 | `formData.has("customCss")` | 3a–3b (potong + log `css_updated`) | Step 4 (log `profile_updated`) |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 3a | CSS > 20.000 char | Diam-diam dipotong ke 20.000 (tanpa error) | Operator potong sendiri dulu |
| — | Upload avatar/banner/OG/favicon | Proses terpisah UC-033 (`processImage`, batas OG 5MB / favicon 2MB) | — |

### Process Relationship Map

```
PF-007 (Theme tersimpan) --triggers--> render publik BAB-5 (pakai tokensJson)
PF-008 (Profil tersimpan) --triggers--> render publik BAB-5 (profileJson + customCss disuntik)
PF-007 / PF-008 --ditulis oleh--> UC-026 backup (BAB-3, scope theme+profile)
```
