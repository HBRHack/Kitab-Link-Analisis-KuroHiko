# Bug Report — Kitab Link

**Mode:** Ekstraksi ringkas (Opsi A) · **Cakupan:** fungsional / esensial · **Kode tidak diubah** (reviewer-only)
**Sumber:** clone `https://github.com/Kurohiko-id/Kitab-Link-Self-Hosted`, `package.json` v0.9.0

> ⚠️ **Semua temuan di bawah adalah DOKUMENTASI SAJA.** Tidak ada perbaikan yang diusulkan atau dikerjakan, sesuai kesepakatan stakeholder (perbaikan REJECT).

---

## Ringkasan

| Severity | Jumlah |
|----------|--------|
| 🔴 High | 1 |
| 🟠 Medium | 1 |
| 🟡 Low | 3 |
| **Total** | **5** |

---

## 🔴 BUG-001 — Scope Discord tidak pernah bisa dibuat (fitur API mati total)

| Field | Nilai |
|-------|-------|
| Severity | **High** |
| Confidence | **High** (langsung dari kode) |
| File | `app/dashboard/integrations-actions.ts:11` ↔ `app/dashboard/token-creator.tsx:17-20` ↔ `app/api/v1/discord-widgets/[id]/route.ts:16,42` |
| Tipe | Logic error / dead feature |

### Deskripsi
UI pembuat token menawarkan **4 scope**, tapi whitelist saat pembuatan hanya memuat **2**.

### Bukti
`token-creator.tsx` menawarkan:
```ts
{ value: "links:read",   label: t.integrations.scopeRead },
{ value: "links:write",  label: t.integrations.scopeWrite },
{ value: "discord:read", label: t.integrations.scopeDiscordRead },
{ value: "discord:write",label: t.integrations.scopeDiscordWrite },
```

`integrations-actions.ts` memfilter saat create:
```ts
const VALID_SCOPES = ["links:read", "links:write"] as const;
...
const validScopes = scopes.filter((s): s is Scope =>
  (VALID_SCOPES as readonly string[]).includes(s));   // discord:* dibuang DIAM-DIAM
```

Sedangkan route API **mensyaratkan** scope itu:
```ts
// app/api/v1/discord-widgets/[id]/route.ts
if (!identity.scopes.includes("discord:read"))  return 403 "Token tidak punya scope discord:read"
if (!identity.scopes.includes("discord:write")) return 403 "Token tidak punya scope discord:write"
```

### Dampak
- `GET /api/v1/discord-widgets/:id` dan `PATCH /api/v1/discord-widgets/:id` **selalu 403** untuk token yang dibuat lewat UI mana pun.
- UI terlihat seolah-olah fitur tersedia (deskripsi scope ikut tampil), padahal tidak akan pernah jalan.
- Kasus penggunaan "kontrol widget Discord dari Stream Deck" yang disebut di komentar kode tidak tercapai.

### Pola terkait (BUG-PATTERNS)
- *Unreachable code path* — endpoint tak bisa pernah lolos otorisasi.
- *Silent filter* — input valid user dibuang tanpa pesan error.

---

## 🟠 BUG-002 — `linkId` / `groupId` tidak divalidasi milik `pageId`

| Field | Nilai |
|-------|-------|
| Severity | **Medium** (self-hosted, single-user → turun dari High) |
| Confidence | **High** (langsung dari kode) |
| File | `app/dashboard/actions.ts` — `renameGroup` (61), `deleteGroup` (73), `toggleGroupVisibility` (81), `deleteLink` (92), `toggleLinkActive` (105), `toggleLinkFeatured` (116), `saveLinkAction` (127/223), `persistBoard` (300) |
| Tipe | IDOR / missing ownership check pada baris anak |

### Deskripsi
Setiap action **memang** memanggil `requireOwnedPage(pageId)`, tapi setelah itu menembak `linkId`/`groupId` **mentah** tanpa pernah menyertakan `eq(links.pageId, pageId)` di klausa WHERE.

### Bukti
```ts
export async function deleteLink(pageId: number, linkId: number) {
  await requireOwnedPage(pageId);                                        // ✅ page divalidasi
  const [existing] = await db.select().from(links).where(eq(links.id, linkId)); // ❌ pageId tidak ikut
  await db.delete(links).where(eq(links.id, linkId));                    // ❌ pageId tidak ikut
```
Pola sama pada `renameGroup`, `deleteGroup`, `toggleGroupVisibility`, `toggleLinkActive`, `toggleLinkFeatured`.

`persistBoard` bahkan tak memvalidasi ID sama sekali:
```ts
export async function persistBoard(input: PersistBoardInput) {
  await requireOwnedPage(input.pageId);
  db.transaction((tx) => {
    input.groupOrder.forEach((groupId, index) => {          // groupId bebas dari client
      tx.update(linkGroups).set({ orderIndex: index }).where(eq(linkGroups.id, groupId)).run();
```

`saveLinkAction` jalur update:
```ts
if (linkId) {
  const [existing] = await db.select().from(links).where(eq(links.id, linkId)); // tanpa pageId
  await db.update(links).set({ ...values, ... }).where(eq(links.id, linkId));   // tanpa pageId
```
…dan `groupId` hasil parse `target` dipakai apa adanya, tanpa cek milik page mana pun.

### Dampak
- Pemanggilan dengan `(pageId milik A, linkId milik B)` **berhasil**: link page B terhapus/diedit, sementara activity log tertulis ke page A (log jadi tidak akurat).
- Via `persistBoard`, link/grup page lain bisa dipindahkan halaman atau urutannya diubah.
- `saveLinkAction` bisa menempelkan link ke `groupId` milik page lain.

### Konteks self-hosted ⬇️
Aplikasi **single-user** — semua page dimiliki akun yang sama, jadi tidak ada eskalasi ke akun lain. Konsekuensinya bukan kebocoran data lintas-pengguna, melainkan **konsistensi data antar-page** yang tak terjamin dan **activity log yang menyesatkan**. Itulah alasannya dinilai Medium, bukan High.

### Pola terkait (BUG-PATTERNS)
- *Missing condition* — filter ownership hilang di query baris anak.
- *Incorrect data flow* — ID dari client diperlakukan sebagai milik konteks yang sudah divalidasi.

---

## 🟡 BUG-003 — Rate-limit Map bisa tumbuh dari input bebas (pertentangan dengan komentar sendiri)

| Field | Nilai |
|-------|-------|
| Severity | **Low** (self-hosted, ada cleanup) |
| Confidence | **High** |
| File | `lib/auth/unlock-rate-limit.ts:1-12` ↔ `app/login/actions.ts:31,129` |
| Tipe | Resource growth / doc-vs-code mismatch |

### Deskripsi
Komentar di `unlock-rate-limit.ts` menetapkan aturan keras:

> *"PENTING soal ukuran Map: key di sini WAJIB dari domain terbatas (id numerik page/user) atau dibatasi eviction-nya... tanpa itu Map ini bisa tumbuh gak terbatas (memory leak beneran)."*

Tapi dua pemakai memakai **username sebagai key** — dan username adalah input bebas dari request:

```ts
const rlKey = `login:${username}`;          // login/actions.ts:31
const rlKey = `reset-request:${username}`;  // login/actions.ts:129
```

### Dampak
- Aturan yang ditulis sendiri di kode dilanggar oleh kode itu sendiri.
- Dalam window 5 menit, pemanggil bisa membuat entri sebanyak mungkin (setiap username berbeda = entri baru).
- Ada mitigasi parsial: `cleanupExpired()` dipanggil di tiap `recordFailedAttempt`, jadi entri lewat 5 menit ikut terhapus → pertumbuhan **terbatas pada window**, bukan selamanya. Komentar "tumbuh gak terbatas" sendiri berlebihan untuk kasus ini.
- Tidak ada rate-limit berbasis IP, jadi pembatasan "8 percobaan" bisa di-reset dengan berganti username.

### Pola terkait (BUG-PATTERNS)
- *Unhandled edge case* — pertumbuhan memori dari input attacker-controlled.
- *Comment-code mismatch* — dokumen di kode tidak mencerminkan implementasi.

---

## 🟡 BUG-004 — Dependensi `zod` terpasang tapi tidak pernah dipakai

| Field | Nilai |
|-------|-------|
| Severity | **Low** (code quality, bukan fungsional) |
| Confidence | **High** (grep seluruh repo = 0 import) |
| File | `package.json:36` (`"zod": "^4.6.2"`) |
| Tipe | Orphaned dependency |

### Deskripsi
`zod` ada di `dependencies`, padahal **nol berkas mengimpornya**. CLAUDE.md bahkan mencantumkan "Validation: Zod" sebagai bagian tech stack, dan `EXTRACTION-GUIDE` mengharapkan validasi schema-based — namun kenyataannya seluruh validasi memakai pola manual:

```ts
const title = String(formData.get("title") ?? "").trim();
const linkType = VALID_LINK_TYPES.includes(linkTypeRaw as LinkType) ? ... : "url";
```

### Dampak
- Klaim di `CLAUDE.md` ("Validation: Zod") **tidak akurat** terhadap kode.
- Satu dependensi (plus transitive deps-nya) ikut ter-bundle tanpa guna; biaya maintenance/audit CVE percuma.
- Tidak ada bug fungsional dari ini.

### Pola terkait (BUG-PATTERNS)
- *Dead code / unused dependency*.
- *Documentation drift* — README/CLAUDE menyebut hal yang tak ada di kode.

---

## 🟡 BUG-005 — `pages.domainValue` tanpa UNIQUE + `proxy.ts` pakai `.limit(1)`

| Field | Nilai |
|-------|-------|
| Severity | **Low** |
| Confidence | **Medium** (inferensi dari skema; perilaku dua-baris belum diuji runtime) |
| File | `lib/db/schema.ts:77` (`domainValue`), `proxy.ts:22-27` |
| Tipe | Missing constraint / ambiguous result |

### Deskripsi
`domainValue` **tidak dideklarasikan UNIQUE**, sedangkan `proxy.ts` mencari page by host dengan:

```ts
const [page] = await db
  .select({ id: pages.id, slug: pages.slug, domainVerified: pages.domainVerified })
  .from(pages)
  .where(and(eq(pages.domainType, "custom_domain"), eq(pages.domainValue, host)))
  .limit(1);                 // ← hasil ambigu bila ada duplikat
```

`changeSlugAction` memang menjaga keunikan `slug`, tetapi **tidak ada guard setara untuk `domainValue`** di `setCustomDomainAction`.

### Dampak
- Dua page bisa sama-sama mengklaim host yang sama; page yang menang ditentukan urutan baca SQLite → tidak deterministik.
- `domainVerified` bisa diset pada page yang salah.
- Praktis jarang terjadi (butuh operator memasukkan host sama dua kali), makanya Low.

### Confidence note
Medium karena saya tidak menjalankan aplikasinya (tidak ada `node_modules`, dan instruksi = jangan menyentuh program). Kesimpulan diturunkan dari pembacaan skema + query saja.

---

## Pola yang DIPERIKSA dan TIDAK ditemukan ✅

| Pola (BUG-PATTERNS.md) | Hasil pemeriksaan |
|---|---|
| Null/undefined tanpa penjagaan | ✅ Aman — jalur kritis dioptional-chaining / fallback (`existing?.thumbnailPath ?? null`) |
| Empty catch block | ✅ Sebagian besar catch punya komentar+fallback eksplisit; catch kosong hanya di `deleteImage` (file tak ada = memang tak apa) |
| Unhandled promise | ✅ `recordLinkClick` fire-and-forget **disediakan** `.catch(...)` |
| Race condition | ✅ Master secret tulis atomik (`flag:"wx"`); setup dicek ulang saat menulis, bukan saat render |
| `= vs == vs ===` | ✅ Semua pemakaian `===` / `!==` |
| Off-by-one | ✅ Backoff array pakai `Math.min(attempts-1, len-1)` — sudah dijaga |
| Unreachable code | ⚠️ Ada di BUG-001 (endpoint tak bisa lolos auth) |
| Duplicated logic | ⚠️ Minor: blok `Bearer token → verifyApiToken → scope check` diulang 5× di `app/api/v1/**` (bukan bug, catatan) |
| Broken pagination | ✅ Tidak ada pagination (dataset kecil, `limit(1)` saja) |

---

## Gate #6 — Value Consistency (PASS)

Pemeriksaan literal value di kode vs Value List di Data Dictionary:

| # | Field / Value List | Literal di Kode | File | Hasil |
|---|---|---|---|---|
| 1 | `last_check_status` = `ok` \| `dead` | `"dead"` / `"ok"` | `lib/dead-links.ts:34`, `lib/db/dead-links.ts:14` | ✅ **PASS** |
| 2 | `last_state` = `live` \| `offline` | `"live"` / `"offline"` | `lib/scheduled-rules.ts:44,64` | ✅ **PASS** |
| 3 | `link_type` (10 nilai) | whitelist `VALID_LINK_TYPES` identik dengan enum schema | `app/dashboard/actions.ts:17-29` vs `schema.ts:146-150` | ✅ **PASS** |
| 4 | `display_style` = `pill`\|`rich`\|`icon`\|`image` | hanya dibaca sebagai union type, tak ada query literal | `lib/db/board.ts` | ✅ **PASS** (tidak ada pemakaian query) |
| 5 | `device_type` = `mobile`\|`tablet`\|`desktop` | ketiga nilai dikembalikan fungsi deteksi | `lib/analytics-capture.ts:4-8` | ✅ **PASS** |
| 6 | `event_type` analytics | `"page_view"` / `"link_click"` | `lib/db/analytics.ts` | ✅ **PASS** |
| 7 | `webhook status` = `pending`\|`sent`\|`failed` | ketiga nilai dipakai | `lib/webhooks.ts` | ✅ **PASS** |
| 8 | `log source` = `dashboard`\|`automation`\|`api` | `"dashboard"` (default), `"api"` di route v1 | `schema.ts:400`, `app/api/v1/links/[id]/route.ts` | ✅ **PASS** |

**Field-completeness (6b):** Tidak ditemukan INSERT/UPDATE raw SQL yang menyebut kolom eksplisit — semua lewat Drizzle query builder yang menyertakan seluruh `values`/`set`. Klaim FK vs kolom redundan tidak berlaku (skema ini tanpa kolom denormalisasi).

**Cakupan (6c):** Pemeriksaan mencakup seluruh kode, termasuk berkas uji (`lib/youtube-live.test.ts`) — tidak ada temuan.

> **Status gate: ✅ PASS** — tidak ada WARNING yang perlu persetujuan user.

---

## Items yang tidak bisa diverifikasi (butuh runtime)

1. BUG-005 perilaku duplikat domain — perlu 2 page dengan `domainValue` sama untuk memastikan.
2. Efek nyata BUG-003 terhadap memori dalam 5 menit window — perlu beban uji.
3. Hasil `npm audit` untuk CVE dependency — `node_modules` tidak ada dan instalasi akan mengubah isi lingkungan.
4. Perilaku cron multi-instance (bila suatu saat dijalankan >1 container) — di luar cakupan review.
