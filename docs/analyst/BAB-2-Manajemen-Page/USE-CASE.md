# BAB-2 — Manajemen Page

Chapter: `BAB-2-Manajemen-Page` · Blok BR: 051–100 (pakai 017–020, 027–029 dari block global awal) · UC: 010–017

> Dokumen ringkas (Opsi A).

## Use Cases

| UC | Judul | Actor | Trigger | Ringkasan Alur | Entities | BR |
|----|-------|-------|---------|----------------|----------|----|
| UC-010 | Buat page baru | Operator | Dashboard → tambah page | `requireSession` → `createPageForUser` → auto-slugify nama → `findAvailableSlug` (hindari reserved + duplikat, suffix `-2`, `-3`…) → insert → opsional set `displayName` profil | pages | BR-017 |
| UC-011 | Ganti slug | Operator | Settings → slug | `requireOwnedPage` → validasi `SLUG_PATTERN` → cek reserved → cek unik (kecuali milik sendiri) → update → revalidate `/[slug]` | pages | BR-017 |
| UC-012 | Set / hapus password page | Operator | Settings → proteksi | `requireOwnedPage` → kosong = `passwordHash=null` (publik), terisi = hash argon2id | pages | — |
| UC-013 | Buat kode akses sementara | Operator | Settings → temp access | `requireOwnedPage` → kode min 4 char → `durationHours` di-clamp 1–720 (30 hari) → hash argon2id → insert | pageAccessCodes | BR-027 |
| UC-014 | Cabut kode akses | Operator | Settings | `requireOwnedPage` → delete by `codeId` + `pageId` | pageAccessCodes | BR-028 |
| UC-015 | Hapus page | Operator | Settings → hapus | `requireSession` + `requireOwnedPage` → tolak bila sisa page ≤ 1 → null-kan `users.primaryPageId` bila menunjuk page ini → delete (cascade) | pages, users | BR-018 |
| UC-016 | Jadikan primary page | Operator | Dashboard | `requireSession` (+`requireOwnedPage` bila non-null) → update `users.primaryPageId` → revalidate `/` | users, pages | — |
| UC-017 | Set custom domain | Operator | Settings → domain | `requireOwnedPage` → validasi hostname regex (tanpa skema/path/spasi) → set `domainType=custom_domain` + `domainValue` → `domainVerified` menyala otomatis saat pertama ke-hit | pages | BR-019, BR-020 |

### Alur kritis (ditulis lengkap)

#### UC-015: Hapus page — Main Flow

1. **Operator** konfirmasi hapus page
2. **System** `requireSession()` — bawa `/login` bila belum login
3. **System** `requireOwnedPage(pageId)` — throw bila bukan milik sendiri
4. **System** hitung jumlah page milik user; bila ≤ 1 → return error `deletePageLastError` dan **berhenti**
5. **System** null-kan `users.primaryPageId` bila menunjuk page ini (FK `set null` tidak terpasang di SQLite karena keterbatasan drizzle-kit)
6. **System** `delete(pages)` → cascade ke `links`, `link_groups`, `analytics_events`, `scheduled_rules`, `activity_logs`, `content_feeds`, `discord_widgets`, `live_badges`, `page_access_codes`
7. **System** `revalidatePath("/dashboard")` + `revalidatePath("/")`

#### UC-017: Alternative Flows

- **AF-1: Domain sudah dipakai page lain** → tidak ada guard unik eksplisit di kolom `domainValue`; `proxy.ts` memakai `.limit(1)` sehingga hanya satu page yang akan cocok. *Dicatat sebagai pengamatan, lihat `bug-report.md` BUG-005.*

## Notes

- Route admin (`/dashboard`, `/login`, `/setup`, `/api`) **diblokir 404** bila diakses lewat host custom domain — implementasi di `proxy.ts`.
- `resolveSlugPath` hanya me-rewrite `/` → `/{slug}` dan `/privacy` → `/{slug}/privacy`; path lain (`/uploads/*`, `/r/*`, `/_next/*`) dilewatkan apa adanya.
