# SECURITY-RULES — BAB-4 Tema & Profil (Recovered)

> ⚠️ RECOVERED dari kode — aturan keamanan input tema/profil yang dirender ke halaman publik. ADR terkait: 007 (enkripsi secret), 006.

## 1. Server Actions Tema (Teramati — `app/dashboard/theme-actions.ts`)

Semua `"use server"` (baris 1) + `requireOwnedPage` + `logActivity` + `revalidatePath`:

| Action | Aturan | Evidence |
|--------|--------|----------|
| `duplicateThemeAction` | duplikat preset → library + set aktif | `:16` (log `:24`, revalidate `:25-26`) |
| `setActiveThemeAction` | ganti tema aktif | `:30` |
| `saveThemeAction` | update `themes.name` + `tokensJson` | `:39` |
| `deleteThemeLibraryAction` | hapus dari library | `:53` |
| `importThemeAction` | terima JSON string/file, validasi type `ImportThemeResult` | `:63,66-70` |

## 2. Input Kustom yang Dirender ke Publik (Teramati — Injection Surface)

| Input | Batas | Rendering | Evidence |
|-------|-------|-----------|----------|
| `customCss` (milik pemilik page) | 20.000 char | `<style>` via `dangerouslySetInnerHTML` | `settings-actions.ts:78-102`; `components/public-page-body.tsx:100` |
| Custom font `@font-face` (URL dari theme) | — | `<style>` via `dangerouslySetInnerHTML` | `public-page-body.tsx:97` |
| `THEME_INIT_SCRIPT` (kode hardcoded, bukan input) | — | `<script>` di `<head>` | `app/layout.tsx:24-30,40` |
| `privacyPolicyContent` | 20.000 char | rendered (tidak ditelusuri penuh dalam sesi ini) | `settings-actions.ts:78-102` |
| Font family per theme (`customFontUrl`) | — | CSS variable / `@font-face` | `lib/theme.ts:118`; `public-page-body.tsx:87` |

**Fakta:** ketiga `dangerouslySetInnerHTML` utama menerima **input pemilik page yang sudah login** (admin-only) — untuk self-host single-user, eksposur = admin itu sendiri. 2 dari 3 (`layout.tsx:40`, `firefly-overlay.tsx:101`) **belum ditelusuri penuh** (batas review analyst — ISSUE.md §Metode).

## 3. Batas Upload Profil (Teramati — `app/dashboard/settings-actions.ts`)

| Batas | Nilai | Evidence |
|-------|-------|----------|
| `MAX_AVATAR_WIDTH` | 512 | `:17` |
| `MAX_BANNER_WIDTH` | 1600 | `:18` |
| `MAX_OG_IMAGE_WIDTH` | 1200 | `:19` |
| `MAX_OG_IMAGE_BYTES` | 5 MB | `:20` |
| `MAX_FAVICON_BYTES` | 2 MB | `:21` |
| Konversi | WebP via `processImage` + `saveImage` (nama `randomUUID().webp`), hapus file lama | `:30-45`; `lib/images/storage.ts:9-16` |
| Body limit | `serverActions.bodySizeLimit: "8mb"` | `next.config.ts:20-23` |

## 4. Secret di Chapter Ini (Teramati)

- `users.youtubeApiKey` — dienkripsi AES-256-GCM purpose `youtube-api-key` saat disimpan/dihapus via `automation-actions.ts:106,112` (`lib/youtube-api-key.ts:6-10`).
- Fallback tanpa key = scraping HTML (`lib/youtube-api-key.ts:22-27`).
- **Bukan env var** — semua di DB ter-encrypt (desain konfigurasi-dari-dashboard, README.id.md:7).

## 5. Account-Level Actions (Teramati)

- `changePasswordAction` (:139), `updateAccountAction` — username unik divalidasi (:168), `toggleStreamerModeAction` (:220).
- Update account avatar juga lewat `saveImage` (randomUUID) — `settings-actions.ts` blok `updateAccountAction`.

## Tidak Teramati di Kode

- Sanitasi HTML `privacyPolicyContent` (apakah di-escape saat render): **belum ditelusuri** dalam sesi ini.
- Sandbox/CSP untuk `customCss` (apakah CSS bisa exfil data): **tidak teramati** adanya CSP header di route manapun.
- Batas jumlah tema per user: tidak teramati.
