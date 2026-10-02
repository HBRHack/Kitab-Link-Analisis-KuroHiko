# Security Report — Kitab Link

**Mode:** Ekstraksi ringkas (Opsi A) · **Postur:** *self-hosted, single-user* · **Kode tidak diubah** (reviewer-only)

> Penilaian severity memakai konteks yang kamu berikan: **di-deploy sendiri, dipakai konten kreator biasa**. Karena itu eksposur publik internet tetap dipertimbangkan (VPS + domain publik), tapi asumsi "satu admin, satu server, tanpa multi-tenant" menurunkan beberapa temuan.

---

## Ringkasan Postur Keamanan

| Kategori | Status |
|----------|--------|
| Password hashing | ✅ **Kuat** — argon2id (19 MiB, t=2, p=1) sesuai OWASP |
| Session | ✅ **Kuat** — HMAC-SHA256 stateless, `timingSafeEqual`, cookie HttpOnly + SameSite=Lax |
| 2FA | ✅ **Kuat** — TOTP secret di-encrypt AES-256-GCM (bukan plaintext), backup code hash argon2id |
| Secret management | ✅ **Kuat** — auto-generate 32 byte, tulis atomik `wx`, mode 0600, lazy (tak ikut ke-bake ke image) |
| Anti path-traversal | ✅ **Kuat** — validasi per-segmen di `/uploads` |
| Rate limiting | ⚠️ **Cukup** — in-memory, tanpa IP-based (lihat BUG-003) |
| Ownership check | ⚠️ **Sebagian** — page divalidasi, baris anak (`linkId`/`groupId`) tidak (BUG-002) |
| Scope API | ❌ **Rusak** — scope discord tak pernah bisa dibuat (BUG-001) |
| Validasi input | ⚠️ Manual, `zod` terpasang tapi tak dipakai (BUG-004) |
| Dependency audit (SCA) | ⛔ **Tidak dijalankan** — `node_modules` absen (lihat § Dependencies) |

**Temuan: 0 Critical · 0 High · 0 Medium (lintas-akun) · 5 Low/Info · 2 Bug fungsional terkait keamanan**

---

## 1. Authentication

| Pemeriksaan | Status | Bukti |
|---|---|---|
| Password hashing | ✅ | `@node-rs/argon2`, `algorithm:2` (Argon2id), `memoryCost:19456`, `timeCost:2`, `parallelism:1` — param dieksplisitkan agar tak berubah diam-diam oleh default upstream |
| Hash korup ditangani | ✅ | `verifyPassword` catch → `false`, bukan melempar 500 mentah |
| Session cookie | ✅ | `kitab_session`, payload `{userId}.{expires}.{signature}`, TTL 7 hari, `httpOnly`, `sameSite:lax`, `secure` dinamis via `isSecureRequest()` |
| Signature timing-safe | ✅ | `crypto.timingSafeEqual` setelah cek panjang |
| Session stateless | ✅ | Tidak ada tabel session — sesuai skala proyek |
| Expiry dicek | ✅ | `Date.now() <= Number(expiresRaw)` |
| Anti user-enumeration | ✅ | `requestPasswordReset` selalu balas `{sent:true}` walau username tak ada |
| 2FA pending terbatas | ✅ | State TOTP sementara 5 menit, session penuh baru terbentuk setelah lolos |
| TOTP secret | ✅ | Di-encrypt AES-256-GCM, key di-derive scrypt per-purpose |
| Setup token one-time | ✅ | `randomBytes(16)`, hanya di stdout (tidak persist ke disk), timing-safe compare |
| Rate limit brute-force | ⚠️ | 8 percobaan / 5 menit per key; **in-memory** (reset saat restart), **tanpa IP-based** → lihat BUG-003 |
| Reset token | ⚠️ *Info-by-design* | Dicetak ke log container (15 menit). **Aman selama akses log = akses server**, yang berarti penyerang sudah menang |

**Perlawanan terhadap serangan umum:**
- Brute-force password → dibatasi 8/5 menit per username, tapi bisa di-reset dengan ganti username (tanpa IP limit).
- Credential stuffing → sama.
- Session fixation → tidak relevan (session dibuat baru setelah auth sukses).
- CSRF → Server Action Next.js + SameSite=Lax cookie; risiko rendah untuk app satu-user.

---

## 2. Authorization

| Pemeriksaan | Status | Bukti |
|---|---|---|
| Gerbang session | ✅ | `requireSession()` di semua action dashboard (kecuali `login`/`setup` — memang publik) |
| Ownership page | ✅ | `requireOwnedPage(pageId)` → query `pages` + `pages.userId = session.userId` |
| Ownership baris anak (link/grup) | ❌ **BUG-002** | `deleteLink`/`renameGroup`/`deleteGroup`/`toggle*`/`saveLinkAction`/`persistBoard` tidak menyertakan `eq(x.pageId, pageId)` |
| IDOR lintas-akun di API | ✅ | `app/api/v1/**` pakai `innerJoin(pages)` + `eq(pages.userId, identity.userId)` |
| Scope check API | ❌ **BUG-001** | Scope `discord:*` diperlukan route tapi tak pernah bisa dibuat |
| Blocking admin di custom domain | ✅ | `proxy.ts` → 404 untuk `/dashboard`, `/login`, `/setup`, `/api` |
| `getSocialLinksForPage` tanpa ownership | ⚠️ **Low** | Server action tanpa `requireOwnedPage`; data yang diembek memang tampil publik → eksposur minim |
| Token API hash | ✅ | SHA-256 (sah, karena token entropi tinggi & dihasilkan sistem) |

> **Catatan severity:** BUG-002 tidak memungkinkan akses ke akun lain karena aplikasi single-user. Konsekuensinya konsistensi data antar-page + activity log menyesatkan — bukan kebocoran antar-pengguna.

---

## 3. Input Validation

| Pemeriksaan | Status | Catatan |
|---|---|---|
| Validasi framework (zod) | ⚠️ **BUG-004** | Terpasang, **0 import** — semua validasi manual `String(formData.get(...))` |
| Whitelist enum | ✅ | `VALID_LINK_TYPES` mencocokkan enum schema (Gate #6 PASS) |
| Batas ukuran upload | ✅ | OG ≤5MB, favicon ≤2MB, file link ≤20MB; `serverActions.bodySizeLimit: "8mb"` |
| Tipe file upload | ✅ | `uploadLinkImage` tolak non-`image/*` |
| Panjang teks | ✅ | `customCss` di-`slice(0,20000)`, password 8–200, `displayName` ≤100 |
| Slug | ✅ | Regex `^[a-z0-9]+(-[a-z0-9]+)*$` + `RESERVED_SLUGS` |
| Custom domain | ✅ | Regex hostname (tanpa skema/path/spasi) |
| JSON import (backup/theme) | ✅ | `JSON.parse` dibungkus try/catch → pesan error; versi diverifikasi (`BACKUP_VERSION === 3`); `parseThemeTokens` buang token tak dikenal |
| SQL injection | ✅ | Semua lewat Drizzle query builder parameterized — **tidak ditemukan string concatenation ke query** |
| Guild ID (endpoint publik) | ✅ | `/api/discord-widget` tolak selain `/^\d+$/` |
| Bug: `imageRadius` | ⚠️ *Info* | `Number(...)` bisa menerima `NaN`/nilai ekstrem → dinormalkan ke `null` bila `!Number.isFinite`, tapi **tanpa clamp rentang** |

---

## 4. Path Traversal & File Handling

| Pemeriksaan | Status | Bukti |
|---|---|---|
| Guard traversal | ✅ | `segments.every(s => s.length>0 && !/[\\/]|\.\./.test(s))` → 404 |
| Path di-join tanpa normalisasi ulang | ⚠️ *Info* | `resolveUploadPath` = `path.join(UPLOAD_ROOT, relative)`. Aman **karena** guard di atas menolak `..` sebelum join; tak ada lapisan kedua |
| Nama file di-random | ✅ | `randomUUID()` → tak bisa ditebak/ditimpa |
| Stream, bukan buffer | ✅ | Komentar eksplisit soal spike RAM di VPS kecil |
| Content-Type whitelist | ✅ | Map ekstensi; fallback `application/octet-stream` |
| File tak lewat sharp | ⚠️ *By design* | `saveFile` menyimpan apa adanya (font, PDF, ZIP) — `caller yang validasi extension`. Catatan: ekstensi file link berasal dari `file.name` user → `saveFile(buffer, "link-files", ext)` **tanpa whitelist ekstensi** |
| Hapus file lama | ✅ | `deleteImage` catch & abaikan (idempoten) |

**Pengamatan tambahan (Low):** ekstensi upload tipe `file` diambil dari nama file (`ext = file.name.split(".").pop()`), hanya dibatasi ukuran + penamaan `randomUUID.ext`. Karena file disajikan dari `/uploads` dengan Content-Type yang diturunkan dari ekstensi, unggahan `.html` akan disajikan sebagai `text/html` (ada di `CONTENT_TYPE_BY_EXT`? **tidak** → `application/octet-stream`, jadi praktis aman karena browser mengunduh, bukan me-render). Tidak ada eksekusi konten.

---

## 5. Data Protection

| Pemeriksaan | Status | Catatan |
|---|---|---|
| Secret hardcoded di kode | ✅ | Pencarian `grep -rniE "(api_key|secret|password|token)\s*[:=]\s*['\"][A-Za-z0-9_-]{16,}"` → **0 temuan** |
| Secret di git | ✅ | `data/.secret` berada di volume data, `.gitignore` |
| Master secret | ✅ | 32 byte acak, `mode:0600`, lazy-load (tak ikut `next build` / image build) |
| Enkripsi at rest (TOTP, YT key) | ✅ | AES-256-GCM, key scrypt per-purpose |
| Password saat lupa | ⚠️ *Info* | Token di log container (by design, tanpa SMTP) |
| Data pribadi pengunjung | ✅ | Analytics hanya: hostname referrer, device class, kode negara dari header. **Tidak ada IP yang disimpan**, tidak ada cookie tracking pengunjung |
| Cookie pengunjung | ✅ | Hanya cookie akses page (`kitab_page_{id}`) bila page diproteksi password |
| HTTPS | ⚠️ *Deployment* | App mengatur `secure` flag otomatis; HTTPS sendiri diserahkan ke Caddy (`install.sh`) |
| Debug mode di produksi | ✅ | Tidak ada; `output:"standalone"` |
| CORS | ⚠️ *Info* | Tidak dikonfigurasi eksplisit → default Same-Origin Next.js. Endpoint API membaca `Authorization` header (bukan cookie) sehingga tetap bisa dipanggil lintas-origin bila tahu token — **wajar untuk API token** |

---

## 6. Server-Side Request Forgery (SSRF)

| Endpoint | Status | Analisis |
|---|---|---|
| `fetchOgImageBuffer(url)` | ⚠️ **Low** | Fetch URL bebas dari server (content-type dicek `image/*` / `text/html`, timeout 5s). **Tapi** URL berasal dari input admin yang sedang membuat link → butuh akses dashboard dulu. Jika penyerang sudah punya akses dashboard, dia sudah menang. |
| `webhooks.targetUrl` | ⚠️ **Low** | `fetch` POST ke URL apa pun, tanpa validasi skema/host/internal-range. Bisa diarahkan ke `http://169.254.169.254` / `http://localhost`. Lagi-lagi **hanya admin** yang bisa menambah webhook. |
| `contentFeeds.feedUrl` | ⚠️ **Low** | Pola sama, `fetch` dengan timeout 8s |
| `checkDeadLinks` (cron harian) | ⚠️ **Low** | Ping HEAD/GET ke semua `linkType:"url"` — URL berasal dari link admin. Cron harian → dampak lambat |
| `fetchDiscordWidgetData` | ✅ | `guildId` divalidasi `^\d+$`, URL dibentuk dari konstanta `WIDGET_ENDPOINT(guildId)` |
| `/api/discord-widget` | ✅ | Validasi `^\d+$` di entry |

**Kesimpulan SSRF:** semua jalur berada di belakang otorisasi admin atau input admin. Untuk **self-hosted single-user**, ini *acceptable* — satu-satunya skenario jadi masalah adalah kombinasi kompromi akun + akses ke metadata cloud, dan deployment yang didokumentasikan adalah VPS/Docker mandiri, bukan cloud multi-tenant.

**Rekomendasi dokumentatif saja** *(tanpa perubahan kode, sesuai kesepakatan)*: catat di runbook operasional bahwa `targetUrl` webhook dan URL link tidak dibatasi ke host tertentu.

---

## 7. XSS (Cross-Site Scripting)

| Lokasi | Status | Analisis |
|---|---|---|
| `<style dangerouslySetInnerHTML>` custom CSS | ⚠️ *By design* | `public-page-body.tsx:100` — CSS disuntik apa adanya, di-`slice(0,20000)`. **Hanya admin** yang bisa mengisi (`saveProfileAction` butuh session). Pengunjung tak bisa menuliskannya. Risiko: CSS exfiltration bila admin menempel CSS dari pihak ketiga. |
| `<style>` custom font | ⚠️ | Pola sama, dari `fontLibrary` milik admin |
| `THEME_INIT_SCRIPT` (`app/layout.tsx:40`) | ⚠️ *Perlu verifikasi* | Script inline di layout — nilai diambil dari konstanta build. **Confidence: Medium**, tidak saya telusuri penuh (di luar cakupan Opsi A) |
| `firefly-overlay.tsx:101` | ⚠️ *Perlu verifikasi* | `dangerouslySetInnerHTML` ke-4 — kemungkinan SVG/animasi statis |
| Konten user → HTML | ✅ | Tidak ditemukan penyisipan `title`/`description`/`url` link ke HTML lewat `dangerouslySetInnerHTML` — dirender sebagai teks React (escape otomatis) |
| `eval` / `new Function` | ✅ | **0 temuan** seluruh repo |

---

## 8. Configuration & Deployment

| Pemeriksaan | Status | Catatan |
|---|---|---|
| Docker | ✅ | Multi-stage, stage runner cuma bawa `.next/standalone` |
| Volume persistensi | ✅ | `kitab-link-data:/app/data` (db + upload + secret) |
| Port terbuka | ⚠️ *Info* | `3000:3000` di-compose; `install.sh` menyiapkan Caddy untuk HTTPS |
| Migration otomatis | ✅ | `instrumentation.ts` menjalankan `migrate()` **sebelum** server menerima request; kalau gagal → server gagal boot (bukan jalan dengan schema rusak) |
| Error handling cron | ✅ | Semua `cron.schedule` dibungkus `.catch` → tak membunuh proses |
| Retry webhook | ✅ | Backoff eksponensial, maks 5 percobaan |
| `allowedDevOrigins` | ⚠️ *Info* | `["192.168.x.x"]` hardcode IP LAN — hanya berlaku di dev, tak berpengaruh di produksi |
| Logging | ⚠️ | Reset token & setup token dicetak ke stdout (by design). Siapa pun yang bisa baca `docker logs` bisa reset password → **akses log = akses root**, konsisten dengan deployment self-hosted |
| Dependency versions | ⛔ **Belum diaudit** | Lihat § Dependencies |

---

## 9. Dependencies (SCA / SBOM)

**Status: ⛔ TIDAK DIJALANKAN**

Alasan:
- `node_modules` tidak ada di clone (repo di-clone saja, tidak di-install).
- Menjalankan `npm install` / `npm audit` akan **mengubah isi lingkungan kerja**, sementara instruksi eksplisit: *jangan rubah apapun programnya, kita hanya reviewer*.

### Manifest (langsung dari `package.json` — tanpa instalasi)

**Direct dependencies (22):**

| Package | Version | Purpose | License* | Known CVE | Status |
|---------|---------|---------|---------|-----------|--------|
| @base-ui/react | ^1.8.0 | Primitif UI | MIT | — | ⚠️ belum dicek |
| @dnd-kit/core | ^6.3.1 | Drag & drop | MIT | — | ⚠️ belum dicek |
| @dnd-kit/sortable | ^10.0.0 | Sortable DnD | MIT | — | ⚠️ belum dicek |
| @dnd-kit/utilities | ^3.2.2 | DnD helper | MIT | — | ⚠️ belum dicek |
| @node-rs/argon2 | ^2.2.1 | **Password hash** | MIT | — | ⚠️ **prioritas audit** |
| better-sqlite3 | ^13.0.3 | **Database driver** | MIT | — | ⚠️ **prioritas audit** (native) |
| class-variance-authority | ^0.7.1 | Styling | MIT | — | ⚠️ belum dicek |
| cn | ^0.2.6 | Class helper | MIT | — | ⚠️ belum dicek |
| drizzle-orm | ^0.45.2 | **ORM** | Apache-2.0 | — | ⚠️ **prioritas audit** |
| lucide-react | ^1.45.0 | Ikon | ISC | — | ⚠️ belum dicek |
| next | 16.3.5 | **Framework** | MIT | — | ⚠️ **prioritas audit** (attack surface terbesar) |
| node-cron | ^4.6.0 | Scheduler | MIT | — | ⚠️ belum dicek |
| otplib | ^13.5.0 | **TOTP 2FA** | MIT | — | ⚠️ **prioritas audit** |
| qrcode | ^1.5.4 | QR generation | MIT | — | ⚠️ belum dicek |
| react | 19.2.8 | UI | MIT | — | ⚠️ belum dicek |
| react-dom | 19.2.8 | UI | MIT | — | ⚠️ belum dicek |
| react-easy-crop | ^6.2.3 | Cropping | MIT | — | ⚠️ belum dicek |
| recharts | ^3.10.1 | Charts | MIT | — | ⚠️ belum dicek |
| rss-parser | ^3.13.0 | **RSS/Atom feed** | MIT | — | ⚠️ **parsing input eksternal** |
| sharp | ^0.35.4 | **Image processing** | Apache-2.0 | — | ⚠️ **prioritas audit** (native, proses file tak terpercaya) |
| tw-animate-css | ^1.4.0 | Animasi CSS | MIT | — | ⚠️ belum dicek |
| zod | ^4.6.2 | Validation — **TIDAK DIPAKAI** (BUG-004) | MIT | — | ⚠️ kandidat dihapus |

\* *License column berdasarkan pengetahuan umum masing-masing package — **belum diverifikasi** dari lockfile.*

**Dev dependencies (11):** tailwindcss 4, typescript 5, eslint 9, drizzle-kit, shadcn — tidak masuk runtime attack surface produksi.

**Transitive dependencies:** `package-lock.json` ada (441 KB) tetapi **belum di-parse** → jumlah pasti transitive belum diketahui.

### SCA Summary

```
Total dependencies  : 22 direct, transitive: BELUM DIHITUNG (lockfile belum diparse)
With known CVE      : TIDAK DIKETAHUI (npm audit belum dijalankan)
Outdated major      : TIDAK DIKETAHUI
License risk        : TIDAK DITEMUKAN pada direct deps (semua MIT/Apache-2.0/ISC)
                     — TAPI belum diperiksa di level transitive
Prioritas audit     : 7 package (next, better-sqlite3, sharp, @node-rs/argon2,
                     drizzle-orm, otplib, rss-parser)
```

### Langkah audit yang disarankan *(dijalankan OLEH OPERATOR, bukan oleh sesi ini)*

```bash
cd "<path folder>"
npm ci                  # butuh node_modules agar audit bisa jalan
npm audit --omit=dev
npm ls --all > /tmp/sbom-tree.txt   # untuk SBOM transitive
```

> Hasilnya silakan ditempel ke bagian ini sebagai pembaruan. Sesi ini sengaja tidak menjalankannya (akan mengubah isi direktori kerja).

---

## Temuan Terurut

| # | Temuan | Severity (self-hosted) | Kategori | Referensi |
|---|--------|------------------------|----------|-----------|
| 1 | Scope `discord:*` tak pernah bisa dibuat → API selalu 403 | **High (fungsional)** | Authorization | `bug-report.md` BUG-001 |
| 2 | `linkId`/`groupId` tak divalidasi milik `pageId` | **Medium** → rendah karena single-user | Authorization | `bug-report.md` BUG-002 |
| 3 | Rate-limit key dari input bebas, tanpa IP limit | **Low** | Authentication | `bug-report.md` BUG-003 |
| 4 | SSRF via OG image / webhook targetUrl / feed URL | **Low** (di belakang auth admin) | SSRF | §6 |
| 5 | `domainValue` tanpa UNIQUE + `.limit(1)` ambigu | **Low** | Config/Data | `bug-report.md` BUG-005 |
| 6 | `getSocialLinksForPage` tanpa ownership check | **Low** (data memang publik) | Authorization | §2 |
| 7 | Ekstensi file link dari nama file user tanpa whitelist | **Low** (Content-Type fallback aman) | File upload | §4 |
| 8 | Dependensi `zod` tak terpakai | **Low** (quality) | SCA | `bug-report.md` BUG-004 |
| 9 | Dependency audit (npm audit / SBOM) belum dijalankan | **Unknown** | SCA | §9 |
| 10 | Reset/setup token dicetak ke log | **Info** (by design) | Logging | §5, §8 |

**Tidak ditemukan:** hardcoded secret · SQL injection · path traversal lolos · password plaintext · `eval` · session tanpa expiry · eksekusi kode dari upload · IDOR lintas-akun di API.

---

## Kontrol yang Sudah Baik (jangan diutak-atik) ✅

1. argon2id dengan param OWASP, dieksplisitkan anti-drift
2. `timingSafeEqual` di semua jalur verifikasi signature (session, page access, setup token)
3. Guard path-traversal per-segmen + nama file `randomUUID`
4. TOTP & YouTube key ter-encrypt AES-256-GCM dengan key turunan scrypt
5. Master secret atomic create (`wx`) + lazy-load (tak ke-bake ke image)
6. Setup token one-time anti-hijack akun pertama
7. Admin path diblokir di custom domain (`proxy.ts`)
8. Anti user-enumeration pada alur reset password
9. Rate limit pada login/TOTP/reset/page unlock (walau in-memory)
10. Migration di-await sebelum server menerima request → tak ada window schema-rusak
11. IDOR lintas-akun tertutup di `app/api/v1/**` (join by `userId`)
12. Tidak ada penyimpanan IP pengunjung

---

## Batasan Review

- **Static analysis saja** — aplikasi tidak dijalankan (tidak ada instalasi dependensi, sesuai instruksi).
- `npm audit` / SCA tidak dijalankan → §9 hanya manifest.
- Confidence severity bug = **High** kecuali BUG-005 (**Medium**, butuh runtime).
- Audit penuh berkas `dangerouslySetInnerHTML` ke-4 (`firefly-overlay.tsx`, `app/layout.tsx`) belum selesai — di luar cakupan Opsi A.
- **Tidak ada perubahan kode maupun rencana perbaikan** — sesuai kesepakatan (perbaikan REJECT).
