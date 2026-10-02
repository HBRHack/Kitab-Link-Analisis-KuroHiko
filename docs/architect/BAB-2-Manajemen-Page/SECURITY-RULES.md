# SECURITY-RULES — BAB-2 Manajemen Page (Recovered)

> ⚠️ RECOVERED dari kode — aturan host/access per page. ADR terkait: 005 (proxy), 006 (session).

## 1. Host Resolution Rules (Teramati — `proxy.ts`)

| # | Kondisi | Perilaku | Evidence |
|---|---------|----------|----------|
| 1 | Host cocok `pages.domainValue` (custom_domain) + path admin (`/dashboard`,`/login`,`/setup`,`/api`) | **404 Not Found** (bukan redirect) | `proxy.ts:14-18,42-44` |
| 2 | Custom domain belum `domainVerified` | auto-verify sekali saat request pertama | `proxy.ts:49-51` |
| 3 | Path `/` di custom domain | rewrite → `/{slug}` | `proxy.ts:53-58`, `:23-27` |
| 4 | Path `/privacy` | rewrite → `/{slug}/privacy` | idem |
| 5 | Path lain (`/uploads/*`, `/r/*`, `/_next/*`) | diteruskan apa adanya (TIDAK diprefix slug) | komentar `proxy.ts:20-22` |
| 6 | Host bukan custom domain | lanjut tanpa rewrite | `proxy.ts:39` |
| 7 | Path dikecualikan dari hook | `_next/static`, `_next/image`, `favicon.ico`, `robots.txt`, `uploads`, `r/` | `proxy.ts:61-66` |

> Tujuan (komentar `proxy.ts:10-13`): admin/auth hanya lewat host utama — kompromi 1 custom domain tidak otomatis membuka dashboard.

## 2. Page Access Control (Teramati)

| # | Fitur | Aturan | Evidence |
|---|-------|--------|----------|
| 1 | Page tanpa password | publik langsung | default; tidak ada guard |
| 2 | Page ber-password | verifikasi argon2id `page.passwordHash` | `app/[slug]/password-actions.ts:37-41` |
| 3 | Kode akses sementara | fallback: cari `pageAccessCodes` yang belum expired, linear compare hash argon2id | `password-actions.ts:43-56` (komentar `:44-45`: linear compare wajar untuk jumlah kecil) |
| 4 | Rate-limit unlock | key `page:{pageId}`, 8 attempt/5 menit | `password-actions.ts:30-33` |
| 5 | Grant | cookie `kitab_page_{id}` HMAC, TTL 30 hari (batas atas "biar gak kepake sebagai password permanen kedua" — komentar `:68`) | `lib/auth/page-session.ts:5,13-26`; `password-actions.ts:64-65` |
| 6 | Buat kode akses | wajib `requireOwnedPage`; min 4 char; hash argon2id; durasi 1..720 jam | `password-actions.ts:72-91` |
| 7 | Cabut kode | delete `WHERE id AND pageId` | `password-actions.ts:93-97` |

## 3. Ownership (Teramati)

- `requireOwnedPage(pageId)` sebelum semua mutasi page — `lib/db/pages.ts:59-71` (throw kalau bukan milik).
- **Known gap**: operasi baris anak tidak menyertakan `pageId` di WHERE (deleteLink dll) → R-06; detail `docs/analyst/ISSUE.md` #2.

## 4. Anti-Findability (Teramati)

- Admin path di custom domain → 404 (bukan 403, tidak membocorkan keberadaan) — `proxy.ts:42-44`.
- `noIndex` toggle per page (disimpan `pages.profileJson` — `settings-actions.ts:78-102`).
- Sitemap: `app/sitemap.ts` ada (hanya untuk host utama — tidak teramati include custom domain).

## Tidak Teramati di Kode

- Rate-limit brute-force DNS/host probing: tidak teramati.
- Validasi `domainValue` saat input (anti duplikat) — guard di `changeSlugAction` ada untuk slug, untuk domain TIDAK ada (ISSUE.md #5) → R-08.
- Auto-expire `domainVerified` saat DNS berubah: tidak teramati (sekali verify, tetap).
