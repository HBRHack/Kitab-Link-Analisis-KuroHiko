# SECURITY-RULES — BAB-3 Link & Grup (Recovered)

> ⚠️ RECOVERED dari kode — otorisasi mutasi link/grup + jalur backup. ADR terkait: 006 (requireSession), 009 (API scope).

## 1. Mutasi via Server Action (Teramati — `app/dashboard/actions.ts`)

Pola wajib (Teramati di semua action):

```
requireSession()           ── lib/auth/require-session.ts:4 (redirect /login)
  └─ requireOwnedPage(pageId) ── lib/db/pages.ts:59-71 (throw kalau bukan milik)
       └─ operasi Drizzle ── logActivity(...) ── revalidatePath("/[slug]")
```

| Action | Guard | WHERE operasi baris anak | Evidence |
|--------|-------|--------------------------|----------|
| `saveLinkAction` | `requireOwnedPage` | insert/update `links` | `actions.ts:245-248` |
| `deleteLink` | `requireOwnedPage` (:93) | **HANYA `eq(links.id, linkId)`** — tanpa `pageId` | `actions.ts:92-95` |
| `renameGroup` / `deleteGroup` / `toggleGroupVisibility` | `requireOwnedPage` | **tanpa `pageId`** | `actions.ts:65-66,75-76` |
| `toggleLinkActive` / `toggleLinkFeatured` | `requireOwnedPage` | **tanpa `pageId`** | `actions.ts:105,116` (analyst ISSUE #2) |
| `persistBoard` (drag & drop) | `requireOwnedPage` (:301) | `groupId`/`orderIndex` dari client **tanpa validasi kepemilikan** | `actions.ts:300-…` (ISSUE #2) |

**Known gap (R-06):** karena app single-user saat ini → tidak ada eskalasi lintas-akun, tapi **lintas-page**: log aktivitas bisa menyesatkan (ISSUE.md #2, severity Medium).

## 2. Social Links & Backup (Teramati)

| Action | Aturan | Evidence |
|--------|--------|----------|
| `getSocialLinksForPage` | **TANPA `requireOwnedPage`** (beda dari action lain) — data memang tampil publik | `app/dashboard/social-links-actions.ts:17` (security-report A2) |
| `exportPageDataAction` | `requireOwnedPage` (:44); scope opsional `{theme, profile, links}`; `id`/`groupId`/`thumbnailPath` di-strip | `app/dashboard/backup-actions.ts:43-61,35-41` |
| `importPageDataAction` | `requireOwnedPage`; versi JSON wajib cocok (`BACKUP_VERSION = 3`); **restore = hapus total** links+groups lama (`WHERE pageId`) lalu insert | `backup-actions.ts:65-143,78-80,117-118` |
| Export route | `app/dashboard/export/route.ts` — `requireOwnedPage` (:27) | idem |

## 3. Upload File Link (Teramati)

| Aturan | Evidence |
|--------|----------|
| Maks 20MB (pesan error eksplisit) | `app/dashboard/actions.ts:147-152` |
| Nama file: `randomUUID().{ext}` — ext dari nama file user, **tanpa whitelist** (R-13) | `lib/images/storage.ts:21-27` |
| Serving: whitelist MIME per ekstensi + fallback `application/octet-stream` (browser download, bukan render) | `app/uploads/[...path]/route.ts:7-15,42` |
| Anti path-traversal per segmen → 404 | `app/uploads/[...path]/route.ts:20-24` |
| Stream (anti RAM spike — komentar `:35-37`) + `Cache-Control: immutable` | `app/uploads/[...path]/route.ts:38-45` |

## 4. API untuk Resource Links (cross-BAB-6)

- `GET/PATCH /api/v1/links` — Bearer + scope + join ownership → `00-Global/API-STANDARD.md` §3.
- **Tidak ada endpoint create/delete link di API** (hanya dashboard action) — `app/api/v1/links/route.ts:7` (GET saja).

## Tidak Teramati di Kode

- Virus/malware scanning upload: tidak teramati.
- Quota jumlah link per page: tidak teramati.
- Soft-delete / undo hapus link: tidak teramati (delete langsung `db.delete`).
