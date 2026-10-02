# ISSUE.md — Laporan untuk Kurohiko (Kitab-Link-Self-Hosted)

> **Dari:** reviewer komunitas · **Tanggal:** 2026-10-02 · **Versi diperiksa:** `0.9.0`
> **Metode:** code review statis (tanpa menjalankan aplikasi)
> **Repo:** https://github.com/Kurohiko-id/Kitab-Link-Self-Hosted
>
> ⚠️ **Catatan sebelum kirim:** semua temuan di bawah **hanya didokumentasikan, tidak ada kode yang diubah**. Silakan sesuaikan nada/format sebelum dibuka sebagai issue di GitHub.

---

## 📋 Daftar Isu

| # | Judul (opsional) | Severity | Tipe | Confidence |
|---|---|---|---|---|
| 1 | Scope `discord:read`/`discord:write` tidak pernah bisa dibuat | 🔴 High | Bug fungsional | High |
| 2 | `linkId`/`groupId` tidak divalidasi terhadap `pageId` | 🟠 Medium | Bug konsistensi data | High |
| 3 | Rate-limit key memakai input bebas (bertentangan dengan komentar di file) | 🟡 Low | Potensi bug | High |
| 4 | Dependensi `zod` terpasang tapi tidak pernah dipakai | 🟡 Low | Housekeeping | High |
| 5 | `pages.domainValue` tanpa UNIQUE + `proxy.ts` pakai `.limit(1)` | 🟡 Low | Potensi bug | Medium |

---

# ISSUE 1 — Scope `discord:read` / `discord:write` tidak pernah bisa dibuat

**Suggested title (EN):** `Discord API scopes are silently stripped on token creation, making /api/v1/discord-widgets always return 403`

**Severity:** 🔴 High (fungsional) · **Confidence:** High

## Masalah

UI pembuat API token menawarkan **4 scope**, tapi whitelist saat pembuatan hanya memuat **2**. Scope `discord:*` dibuang **diam-diam** tanpa pesan error, padahal route API-nya mensyaratkan scope tersebut.

Hasilnya: `GET /api/v1/discord-widgets/:id` dan `PATCH /api/v1/discord-widgets/:id` **selalu 403** untuk token yang dibuat lewat UI mana pun. Fitur "kontrol widget Discord dari Stream Deck" yang disebut di komentar kode tidak akan pernah bisa jalan.

## Lokasi kode

**1. UI menawarkan 4 scope** — `app/dashboard/token-creator.tsx:17-20`

```ts
const SCOPE_OPTIONS = [
  { value: "links:read",    label: t.integrations.scopeRead },
  { value: "links:write",   label: t.integrations.scopeWrite },
  { value: "discord:read",  label: t.integrations.scopeDiscordRead },
  { value: "discord:write", label: t.integrations.scopeDiscordWrite },
];
```

**2. Tapi whitelist create hanya 2** — `app/dashboard/integrations-actions.ts:11-17`

```ts
const VALID_SCOPES = ["links:read", "links:write"] as const;  // ← discord:*
type Scope = (typeof VALID_SCOPES)[number];                    //   gak ada

export async function createApiTokenAction(name: string, scopes: string[]) {
  const session = await requireSession();
  const validScopes = scopes.filter((s): s is Scope =>
    (VALID_SCOPES as readonly string[]).includes(s));  // ← discord:* dibuang DIAM-DIAM
  const result = await createApiToken(session.userId, name, validScopes);
```

**3. Route API tetap mensyaratkan scope itu** — `app/api/v1/discord-widgets/[id]/route.ts:16-17, 42-43`

```ts
if (!identity.scopes.includes("discord:read")) {
  return NextResponse.json({ error: "Token tidak punya scope discord:read" }, { status: 403 });
}
...
if (!identity.scopes.includes("discord:write")) {
  return NextResponse.json({ error: "Token tidak punya scope discord:write" }, { status: 403 });
}
```

## Dampak

- Endpoint widget Discord tidak bisa dipakai sama sekali oleh token mana pun.
- UI terlihat seolah-olah fitur tersedia (deskripsi scope ikut ditampilkan), padahal tidak akan pernah jalan → membingungkan.
- User tidak dapat pesan apa pun bahwa scope-nya dibuang.

## Saran (silakan dipilih, bukan perintah)

- Tambahkan `"discord:read"` dan `"discord:write"` ke `VALID_SCOPES`, **atau**
- Sembunyikan opsi scope Discord di `token-creator.tsx` bila memang belum didukung, **atau**
- Lemparkan error eksplisit saat user memilih scope yang tidak valid, alih-alih membuangnya diam-diam.

---

# ISSUE 2 — `linkId` / `groupId` tidak divalidasi terhadap `pageId`

**Suggested title (EN):** `Dashboard actions verify pageId but not that linkId/groupId belongs to that page`

**Severity:** 🟠 Medium · **Confidence:** High

## Masalah

Setiap action memang memanggil `requireOwnedPage(pageId)`, tapi setelah itu menembak `linkId`/`groupId` **mentah** tanpa menyertakan `eq(x.pageId, pageId)` di klausa WHERE. Artinya pemanggil dengan `(pageId milik A, linkId milik B)` tetap berhasil.

## Lokasi kode — `app/dashboard/actions.ts`

```ts
// deleteLink (baris 92)
export async function deleteLink(pageId: number, linkId: number) {
  await requireOwnedPage(pageId);                                              // ✅ page dicek
  const [existing] = await db.select().from(links).where(eq(links.id, linkId)); // ❌ pageId gak ikut
  await db.delete(links).where(eq(links.id, linkId));                          // ❌ pageId gak ikut
```

Pola sama ditemukan di:

| Fungsi | Baris | Kolom hilang di WHERE |
|---|---|---|
| `renameGroup` | 61 | `linkGroups.pageId` |
| `deleteGroup` | 73 | `linkGroups.pageId` |
| `toggleGroupVisibility` | 81 | `linkGroups.pageId` |
| `deleteLink` | 92 | `links.pageId` |
| `toggleLinkActive` | 105 | `links.pageId` |
| `toggleLinkFeatured` | 116 | `links.pageId` |
| `saveLinkAction` (jalur update) | 223 | `links.pageId` |

`persistBoard` (baris 300) bahkan tidak memvalidasi ID apa pun:

```ts
export async function persistBoard(input: PersistBoardInput) {
  await requireOwnedPage(input.pageId);
  db.transaction((tx) => {
    input.groupOrder.forEach((groupId, index) => {          // groupId bebas dari client
      tx.update(linkGroups).set({ orderIndex: index }).where(eq(linkGroups.id, groupId)).run();
```

Dan `saveLinkAction` memakai `groupId` hasil parse form apa adanya:

```ts
const groupId = target.startsWith("group:") ? Number(target.slice("group:".length)) : null;
// ... tidak ada cek eq(linkGroups.pageId, pageId) sebelum dipakai di values
```

## Dampak

- Link/grup page B bisa terhapus/diedit sementara activity log tertulis ke page A → **log jadi menyesatkan**.
- Via `persistBoard`, urutan atau keanggotaan link page lain bisa berubah.
- `saveLinkAction` bisa menempelkan link ke `groupId` milik page lain.

> **Catatan konteks:** karena aplikasi ini single-user, semua page milik akun yang sama → **tidak ada eskalasi ke akun lain**. Dampaknya konsistensi data antar-page + activity log yang tidak akurat. Karena itu severity dinilai Medium, bukan High.

## Saran

Tambahkan `pageId` ke klausa WHERE pada semua operasi baris anak, misalnya:

```ts
const [existing] = await db.select().from(links)
  .where(and(eq(links.id, linkId), eq(links.pageId, pageId)))
  .limit(1);
if (!existing) return; // atau throw, konsisten dengan requireOwnedPage
```

Untuk `persistBoard`, validasi seluruh `groupOrder` / `linkIds` berasal dari `input.pageId` sebelum transaksi jalan.

---

# ISSUE 3 — Rate-limit key memakai input bebas

**Suggested title (EN):** `Rate-limit map key uses attacker-controlled username, contradicting the guard comment in the same file`

**Severity:** 🟡 Low · **Confidence:** High

## Masalah

Komentar di `lib/auth/unlock-rate-limit.ts:8-11` menetapkan aturan keras:

> *"PENTING soal ukuran Map: key di sini WAJIB dari domain terbatas (id numerik page/user) atau dibatasi eviction-nya... tanpa itu Map ini bisa tumbuh gak terbatas (memory leak beneran)."*

Tapi dua pemakai memakai **username sebagai key** — dan username adalah input bebas dari request:

```ts
// app/login/actions.ts:31
const rlKey = `login:${username}`;

// app/login/actions.ts:129
const rlKey = `reset-request:${username}`;
```

## Dampak

1. Aturan yang ditulis sendiri di kode dilanggar oleh kode itu sendiri.
2. Dalam window 5 menit, pemanggil bisa membuat entri sebanyak mungkin (username berbeda = entri baru).
3. Pembatasan "8 percobaan" bisa di-reset dengan berganti username, karena **tidak ada rate-limit berbasis IP**.

**Mitigasi yang sudah ada:** `cleanupExpired()` dipanggil di tiap `recordFailedAttempt`, jadi entri lewat 5 menit ikut terhapus → pertumbuhan terbatas pada window, bukan selamanya. Karena itu severity Low.

> ⚠️ Komentar "bisa tumbuh gak terbatas" sendiri agak berlebihan untuk kasus ini — cleanup memang membatasi. Mungkin komentarnya yang perlu diluruskan, bukan kodenya.

## Saran

- Tambahkan rate-limit berbasis IP di lapisan proxy/`proxy.ts`, atau
- Batasi jumlah key yang di-track (mis. LRU / cap maksimum), atau
- Luruskan komentar agar sesuai dengan perilaku sebenarnya.

---

# ISSUE 4 — Dependensi `zod` terpasang tapi tidak pernah dipakai

**Suggested title (EN):** `zod is listed as a dependency (and mentioned in CLAUDE.md) but has zero imports`

**Severity:** 🟡 Low (housekeeping) · **Confidence:** High

## Masalah

- `package.json:36` → `"zod": "^4.6.2"` ada di `dependencies`
- `CLAUDE.md` bagian Tech Stack menulis: **"Validation: Zod"**
- Hasil grep seluruh repo (`app/`, `lib/`, `components/`): **0 import**

```bash
$ grep -rn 'from "zod"' app lib components --include="*.ts" --include="*.tsx" | wc -l
0
```

Realitanya seluruh validasi memakai pola manual:

```ts
const title = String(formData.get("title") ?? "").trim();
const linkType = VALID_LINK_TYPES.includes(linkTypeRaw as LinkType) ? (linkTypeRaw as LinkType) : "url";
```

## Dampak

- Klaim di `CLAUDE.md` **tidak akurat** terhadap kode.
- Satu dependensi (plus transitive-nya) ikut ter-bundle tanpa guna → biaya audit CVE & upgrade percuma.
- **Tidak ada bug fungsional** dari ini.

## Saran

Hapus `zod` dari `dependencies`, **atau** perbarui `CLAUDE.md` agar sesuai (validasi manual). Kalau rencananya mau pakai Zod nanti, cukup catat di CLAUDE.md bahwa "belum diadopsi".

---

# ISSUE 5 — `pages.domainValue` tanpa UNIQUE + `proxy.ts` pakai `.limit(1)`

**Suggested title (EN):** `pages.domainValue has no UNIQUE constraint while proxy.ts resolves host with .limit(1)`

**Severity:** 🟡 Low · **Confidence:** Medium ⚠️ *(disimpulkan dari pembacaan kode, belum diuji runtime)*

## Masalah

`domainValue` tidak dideklarasikan UNIQUE di skema:

```ts
// lib/db/schema.ts:77
domainValue: text("domain_value"),   // ← tanpa .unique()
```

Sedangkan `proxy.ts:22-27` mencari page berdasarkan host dengan `.limit(1)`:

```ts
const [page] = await db
  .select({ id: pages.id, slug: pages.slug, domainVerified: pages.domainVerified })
  .from(pages)
  .where(and(eq(pages.domainType, "custom_domain"), eq(pages.domainValue, host)))
  .limit(1);                 // ← ambigu bila ada duplikat
```

`changeSlugAction` memang menjaga keunikan `slug`, tapi **tidak ada guard setara** di `setCustomDomainAction` untuk `domainValue`.

## Dampak

- Dua page bisa sama-sama mengklaim host yang sama → page yang menang ditentukan urutan baca SQLite, **tidak deterministik**.
- `domainVerified` bisa diset pada page yang salah.
- Praktis jarang terjadi (butuh operator memasukkan host sama dua kali), makanya Low.

## Saran

- Tambahkan `.unique()` pada `domainValue`, **atau**
- Validasi keunikan di `setCustomDomainAction` (pola sama seperti `changeSlugAction`), **atau**
- Bila memang dibiarkan, pastikan `.limit(1)` diganti resolusi yang deterministik + catat perilakunya.

> ⚠️ **Confidence Medium** karena review ini statis (aplikasi tidak dijalankan). Sebaiknya direproduksi dulu: buat 2 page dengan `domainValue` sama, lalu akses host tersebut.

---

# 📎 Lampiran A — Temuan keamanan yang layak dicatat

*Ini bukan bug, lebih ke observasi untuk dokumentasi. Silakan dinilai sendiri apakah perlu issue terpisah.*

### A1. SSRF pada OG image / webhook / content feed

`fetchOgImageBuffer(url)`, `webhooks.targetUrl`, dan `contentFeeds.feedUrl` melakukan `fetch` ke URL bebas tanpa validasi skema/host/internal-range (bisa diarahkan ke `http://169.254.169.254` / `http://localhost`).

**Severity rendah untuk self-hosted** karena ketiganya hanya bisa diisi oleh admin yang sudah login — penyerang yang sudah punya akses dashboard pada dasarnya sudah menang. Tapi layak didokumentasikan di runbook operasional.

### A2. `getSocialLinksForPage` tanpa ownership check

`app/dashboard/social-links-actions.ts:17` — server action ini tidak memanggil `requireOwnedPage`. Data yang dibaca memang tampil publik, jadi eksposurnya minimal, tapi polanya berbeda dari action lain.

### A3. Ekstensi file upload diambil dari nama file user

```ts
// app/dashboard/actions.ts (saveLinkAction, tipe "file")
const ext = file.name.split(".").pop()?.toLowerCase() || "bin";
newLinkFilePath = await saveFile(buffer, "link-files", ext);
```

Tanpa whitelist ekstensi. **Praktis aman** karena `CONTENT_TYPE_BY_EXT` di `app/uploads/[...path]/route.ts` tidak mengenal `.html` → fallback `application/octet-stream` (browser mengunduh, bukan me-render). Tapi whitelist ekstensi eksplisit akan lebih defensif.

### A4. Dependency audit (SBOM) belum bisa dilakukan

`npm audit` tidak dijalankan selama review ini (butuh instalasi yang mengubah direktori kerja). 7 package prioritas yang layak diaudit:

`next`, `better-sqlite3`, `sharp`, `@node-rs/argon2`, `drizzle-orm`, `otplib`, `rss-parser`

Sarankan menambahkan **Dependabot / Renovate** ke `.github/workflows` supaya CVE terpantau otomatis.

---

# ✅ Kontrol keamanan yang SUDAH BAIK

Supaya laporannya seimbang — ini yang **tidak perlu diubah**:

| # | Kontrol | Detail |
|---|---|---|
| 1 | Password hashing | argon2id, param OWASP dieksplisitkan anti-drift (19 MiB, t=2, p=1) |
| 2 | Timing-safe compare | `crypto.timingSafeEqual` di session, page access, setup token |
| 3 | Anti path-traversal | Validasi per-segmen di `/uploads` + nama file `randomUUID()` |
| 4 | Enkripsi secret | TOTP & YouTube key di-encrypt AES-256-GCM, key turunan scrypt per-purpose |
| 5 | Master secret | Atomic create (`wx`), mode 0600, lazy-load (tak ke-bake ke image) |
| 6 | Setup token | One-time, hanya stdout, timing-safe compare → anti-hijack akun pertama |
| 7 | Admin path di custom domain | Diblokir 404 oleh `proxy.ts` |
| 8 | Anti user-enumeration | `requestPasswordReset` selalu balas `{sent:true}` |
| 9 | IDOR lintas-akun di API | `app/api/v1/**` pakai `innerJoin(pages)` + `eq(pages.userId, ...)` |
| 10 | Migration safety | `migrate()` di-await sebelum server menerima request → tak ada window schema-rusak |
| 11 | Cron error handling | Semua `cron.schedule` dibungkus `.catch` |
| 12 | Privasi pengunjung | Tidak ada IP yang disimpan; hanya referrer host, device class, kode negara dari header proxy |

---

# 🔍 Metode & batasan review

- **Static analysis saja** — aplikasi tidak pernah dijalankan.
- Tidak ada dependency yang diinstal → **`npm audit` / SCA belum dilakukan**.
- Confidence severity: **High** untuk ISSUE 1–4 · **Medium** untuk ISSUE 5 (butuh runtime).
- Tidak ditelusuri penuh (di luar cakupan): 2 dari 4 `dangerouslySetInnerHTML` (`app/layout.tsx:40` dan `components/firefly-overlay.tsx:101`).
- **Tidak ada kode yang diubah** — seluruh temuan murni dokumentasi.

---

### Format siap-tempel per issue

Buka GitHub → New issue → New issue → tempel salah satu bagian `# ISSUE N` di atas. Judul alternatif ada di baris **Suggested title** masing-masing (EN) atau tulis versi ID-nya sendiri.

**Suggested labels** (kalau repo-nya pakai label): `bug`, `good first issue` (untuk ISSUE 4), `needs-triage`
