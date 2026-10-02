# Requirements Matrix — Master

Traceability kebutuhan fungsional **Kitab Link** berdasarkan pembacaan kode (mode Opsi A / ekstraksi ringkas). Semua status `Confirmed` bila langsung terlihat di kode; `Pending` bila hanya tersirat.

> Sumber kebenaran: `lib/db/schema.ts`, `app/**/actions.ts`, `app/api/**`, `instrumentation.ts`.

## Requirements by Priority

### Must Have

| ID | Requirement | Chapter | Use Case | Business Rules | Status | Owner |
|----|-------------|---------|----------|----------------|--------|-------|
| FR-001 | Buat akun admin pertama secara aman (setup token one-time) | BAB-1-Otentikasi-Akun | UC-001 | BR-014 | Confirmed | — |
| FR-002 | Login dengan username + password (argon2id) | BAB-1-Otentikasi-Akun | UC-002 | BR-014 | Confirmed | — |
| FR-003 | Autentikasi 2FA TOTP dengan backup code sekali pakai | BAB-1-Otentikasi-Akun | UC-003, UC-007, UC-008 | BR-015 | Confirmed | — |
| FR-004 | Rate limit percobaan login/TOTP/reset | BAB-1-Otentikasi-Akun | UC-002, UC-003, UC-005 | — | Confirmed | — |
| FR-005 | Reset password via token di log container (tanpa SMTP) | BAB-1-Otentikasi-Akun | UC-005, UC-006 | BR-016 | Confirmed | — |
| FR-006 | Session cookie HMAC stateless (7 hari) | BAB-1-Otentikasi-Akun | UC-002, UC-004 | — | Confirmed | — |
| FR-007 | Buat/cabut API token dengan scope izin | BAB-1-Otentikasi-Akun | UC-009 | BR-030, BR-031, BR-032 | Confirmed | — |
| FR-008 | Ganti password dari dashboard (verify password lama) | BAB-1-Otentikasi-Akun | UC-034 | BR-014 | Confirmed | — |
| FR-013 | CRUD multi-page dalam satu akun | BAB-2-Manajemen-Page | UC-010, UC-015 | BR-018 | Confirmed | — |
| FR-014 | Slug unik + valid + hindari reserved | BAB-2-Manajemen-Page | UC-010, UC-011 | BR-017 | Confirmed | — |
| FR-015 | Password-protect halaman publik | BAB-2-Manajemen-Page | UC-012 | — | Confirmed | — |
| FR-016 | Kode akses sementara (expiry maks 30 hari) | BAB-2-Manajemen-Page | UC-013, UC-014 | BR-027, BR-028, BR-029 | Confirmed | — |
| FR-017 | Primary page di root `/` | BAB-2-Manajemen-Page | UC-016 | — | Confirmed | — |
| FR-018 | Custom domain per page + verifikasi otomatis | BAB-2-Manajemen-Page | UC-017, UC-036 | BR-019, BR-020 | Confirmed | — |
| FR-025 | CRUD link dengan 10 tipe link | BAB-3-Link-Grup | UC-018 | BR-023, BR-026 | Confirmed | — |
| FR-026 | Upload gambar → selalu WebP (sharp) | BAB-3-Link-Grup | UC-018, UC-033 | BR-022 | Confirmed | — |
| FR-027 | Grup link + show/hide manual | BAB-3-Link-Grup | UC-022, UC-023 | BR-021 | Confirmed | — |
| FR-028 | Drag & drop urutan link dan grup | BAB-3-Link-Grup | UC-024 | — | Confirmed | — |
| FR-029 | Icon sosial dua baris (atas/bawah) | BAB-3-Link-Grup | UC-025 | — | Confirmed | — |
| FR-030 | Auto-fetch thumbnail OG image untuk rich card | BAB-3-Link-Grup | UC-018 | — | Confirmed | — |
| FR-057 | Render halaman publik tanpa cache | BAB-5-Publik-Analytics | UC-035 | — | Confirmed | — |
| FR-058 | Redirect semua klik lewat `/r/{id}` + catat analytics | BAB-5-Publik-Analytics | UC-038 | BR-025, BR-045, BR-047 | Confirmed | — |
| FR-059 | Proteksi path traversal pada `/uploads` | BAB-5-Publik-Analytics | UC-040 | — | Confirmed | — |
| FR-060 | OG image auto-generate mengikuti theme | BAB-5-Publik-Analytics | UC-039 | — | Confirmed | — |
| FR-061 | Analytics views/klik + referrer, device, negara | BAB-5-Publik-Analytics | UC-041 | BR-046 | Confirmed | — |
| FR-069 | API token auth (Bearer) + scope check | BAB-6-Integrasi-Automasi | UC-043–UC-047 | BR-030 | Confirmed | — |
| FR-070 | Webhook dengan signature HMAC + retry queue | BAB-6-Integrasi-Automasi | UC-048–UC-050 | BR-033, BR-034, BR-035, BR-036 | Confirmed | — |
| FR-071 | Automasi show/hide berbasis YouTube live / jadwal | BAB-6-Integrasi-Automasi | UC-051 | BR-037, BR-038, BR-039 | Confirmed | — |
| FR-072 | Badge "sedang live" terpisah dari automasi | BAB-6-Integrasi-Automasi | UC-052 | BR-040 | Confirmed | — |
| FR-073 | Auto-post item RSS/Atom menjadi link | BAB-6-Integrasi-Automasi | UC-053 | BR-043, BR-044 | Confirmed | — |
| FR-074 | Widget Discord (4 style, floating/inline) | BAB-6-Integrasi-Automasi | UC-054 | BR-041, BR-042 | Confirmed | — |

### Should Have

| ID | Requirement | Chapter | Use Case | Business Rules | Status | Owner |
|----|-------------|---------|----------|----------------|--------|-------|
| FR-043 | Theme library lintas page + preset galeri | BAB-4-Tema-Profil | UC-027–UC-031 | — | Confirmed | — |
| FR-044 | Custom CSS per page (maks 20.000 char) | BAB-4-Tema-Profil | UC-032 | — | Confirmed | — |
| FR-045 | Backup/restore JSON (theme + profile + links) | BAB-3-Link-Grup, BAB-4-Tema-Profil | UC-026 | — | Confirmed | — |
| FR-046 | Activity log per page dengan sumber (dashboard/automation/api) | 00-Global | UC-018, UC-045, UC-051 | BR-048 | Confirmed | — |
| FR-062 | Export analytics CSV | BAB-5-Publik-Analytics | UC-042 | — | Confirmed | — |
| FR-063 | Deteksi link mati (2x gagal berturut) | BAB-6-Integrasi-Automasi | — | BR-024 | Confirmed | — |
| FR-064 | Cek update rilis GitHub + notif bell | BAB-6-Integrasi-Automasi | UC-054 | — | Confirmed | — |
| FR-065 | i18n dashboard (ID/EN) + halaman publik | 00-Global | UC-005, UC-037 | BR-048 | Confirmed | — |
| FR-066 | Streamer mode (sembunyikan info sensitif) | BAB-4-Tema-Profil | UC-034 | — | Confirmed | — |
| FR-067 | i18n halaman publik (ID/EN) | BAB-5-Publik-Analytics | UC-037 | — | Confirmed | — |

### Could Have

| ID | Requirement | Chapter | Use Case | Business Rules | Status | Owner |
|----|-------------|---------|----------|----------------|--------|-------|
| FR-085 | UTM param otomatis pada link | BAB-3-Link-Grup | UC-018 | — | Confirmed | — |
| FR-086 | Campaign source (`utm_source`/`ref`) mengatasi in-app browser | BAB-5-Publik-Analytics | UC-035 | — | Confirmed | — |
| FR-087 | Country dari header reverse proxy (tanpa GeoIP DB) | BAB-5-Publik-Analytics | UC-041 | BR-046 | Confirmed | — |
| FR-088 | Health check link mati harian jam 03:00 | BAB-6-Integrasi-Automasi | — | BR-024 | Confirmed | — |
| FR-089 | Update otomatis `domainVerified` saat pertama hit | BAB-2-Manajemen-Page | UC-036 | BR-020 | Confirmed | — |

## Full Traceability Matrix

| ID | Requirement | Chapter | Use Case | Business Rules | Non-Functional | Status |
|----|-------------|---------|----------|----------------|----------------|--------|
| FR-001 | Setup admin pertama | BAB-1 | UC-001 | BR-014 | NFR-SEC-001 | Confirmed |
| FR-002 | Login argon2id | BAB-1 | UC-002 | BR-014 | NFR-SEC-002 | Confirmed |
| FR-003 | 2FA TOTP | BAB-1 | UC-003, UC-007, UC-008 | BR-015 | NFR-SEC-003 | Confirmed |
| FR-004 | Rate limit | BAB-1 | UC-002, UC-003, UC-005 | — | NFR-SEC-004 | Confirmed |
| FR-005 | Reset via log | BAB-1 | UC-005, UC-006 | BR-016 | NFR-SEC-005 | Confirmed |
| FR-006 | Session HMAC | BAB-1 | UC-002, UC-004 | — | NFR-SEC-006 | Confirmed |
| FR-007 | API token scoped | BAB-1 | UC-009 | BR-030, BR-031, BR-032 | NFR-SEC-007 | Confirmed |
| FR-013 | Multi-page CRUD | BAB-2 | UC-010, UC-015 | BR-018 | — | Confirmed |
| FR-014 | Slug valid | BAB-2 | UC-010, UC-011 | BR-017 | — | Confirmed |
| FR-018 | Custom domain | BAB-2 | UC-017, UC-036 | BR-019, BR-020 | — | Confirmed |
| FR-025 | 10 tipe link | BAB-3 | UC-018 | BR-023, BR-026 | — | Confirmed |
| FR-028 | Drag & drop | BAB-3 | UC-024 | — | NFR-USAB-001 | Confirmed |
| FR-058 | Redirect + analytics | BAB-5 | UC-038 | BR-025, BR-045, BR-047 | — | Confirmed |
| FR-059 | Anti path traversal | BAB-5 | UC-040 | — | NFR-SEC-008 | Confirmed |
| FR-069 | API scope | BAB-6 | UC-043–UC-047 | BR-030 | NFR-SEC-007 | Confirmed |
| FR-070 | Webhook HMAC + retry | BAB-6 | UC-048–UC-050 | BR-033–BR-036 | — | Confirmed |
| FR-071 | Automasi show/hide | BAB-6 | UC-051 | BR-037–BR-039 | — | Confirmed |
| FR-073 | RSS auto-post | BAB-6 | UC-053 | BR-043, BR-044 | — | Confirmed |

## Coverage Summary

| Area | Total | Confirmed | Pending | N/A |
|------|-------|-----------|---------|-----|
| Functional Requirements | 47 | 47 | 0 | 0 |
| Business Rules (tertulis) | 48 | 48 | 0 | 0 |
| Non-Functional | 8 | 8 | 0 | 0 |
| Use Cases | 54 | 54 | 0 | 0 |

## Gaps and Risks

| Gap | Impact | Mitigation |
|-----|--------|------------|
| FR-047–FR-056, FR-075–FR-084, FR-090 belum dipakai (block di NUMBERING-LOG) | Nomorompat kosong — bukan bug, hanya block belum terpakai | Aman dibiarkan; jangan digunakan ulang tanpa catatan |
| `scope discord:*` ada di FR-069 tapi tak bisa dibuat di kode (BUG-001) | Requirement tercatat Confirmed (route-nya ada) tapi **tidak tercapai end-to-end** | Dicatat di `bug-report.md` BUG-001 |
| Ownership `linkId`/`groupId` tidak divalidasi (BUG-002) | FR-013/FR-025 pada praktik lintas-page tidak terjamin | Dicatat di `bug-report.md` BUG-002 |
| Tidak ada dokumen `docs/arsitektur.md` (disebut di CLAUDE.md tapi absen) | Referensi arsitektur internal tak tersedia untuk cross-check | Hanya `docs/arsitektur-link-in-bio.md` yang ada |
| NFR (performa, ketersediaan, kapasitas) tidak didefinisikan di kode | Tak ada baseline untuk audit non-fungsional | Perlu input stakeholder bila diperlukan |

## Status Legend

| Status | Meaning |
|--------|---------|
| Confirmed | Terlihat langsung di kode sumber |
| Pending | Belum terkonfirmasi / hanya tersirat |
| In Progress | Sedang diimplementasikan |
| Deprecated | Tak lagi berlaku |
| N/A | Tidak berlaku untuk sistem ini |
