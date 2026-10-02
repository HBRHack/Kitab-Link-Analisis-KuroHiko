# ANALISIS-KITAB-LINK

## Analysis & System Design Portfolio — Review Dokumentasi Kitab Link (Self-Hosted)

> **Status proyek:** Analysis-first / documentation review portfolio  
> **Bentuk:** Ekstraksi & review dokumentasi sistem dari kode: requirement, use case, data model, business rules, process flow, security/API notes, dan decision records  
> **Sumber yang di-review:** [`github.com/Kurohiko-id/Kitab-Link-Self-Hosted`](https://github.com/Kurohiko-id/Kitab-Link-Self-Hosted) `v0.9.0` (clone lokal **read-only** — tidak ada satu baris kode pun yang diubah)  
> **Implementasi aplikasi:** Bukan fokus repository ini — aplikasinya sudah ada di repo sumber; yang didemonstrasikan di sini adalah kemampuan **membaca kode → menurunkan dokumentasi sistem**

---

## 1. Apa sebenarnya proyek ini?

`ANALISIS-KITAB-LINK` adalah **review analitis terhadap sebuah aplikasi link-in-bio self-hosted yang sudah jadi** (Kitab Link, v0.9.0), bukan proyek menulis aplikasi dari nol.

Fokusnya pada:

- otentikasi & akun (session, 2FA, API token);
- manajemen page (aggregate root, custom domain, proteksi akses);
- konten link & grup;
- tema & profil;
- halaman publik & analytics; serta
- integrasi & automasi (webhook, cron, RSS, Discord widget).

Repository ini sengaja dibuat **analysis-first**.

Yang ingin ditunjukkan bukan "saya bisa bikin aplikasi", melainkan bagaimana sebuah **kode produksi yang sudah berjalan** diterjemahkan kembali menjadi:

- requirement & traceability (FR ↔ UC ↔ BR);
- use case per chapter;
- ERD + data dictionary (15 tabel);
- business rules bernomor;
- process flow (state machine + alur kritis);
- temuan bug/keamanan berteori evidence; serta
- Architecture Decision Records (ADR) yang di-*recover* dari kode.

Dengan pendekatan ini, interviewer dapat membaca **cara saya berpikir sebagai analis/system designer dari kode nyata**, tanpa harus menjalankan program lebih dulu.

---

## 2. Repository ini ditujukan kepada siapa?

**Untuk interviewer / hiring team**
- IT Business Analyst · System Analyst · Solution Architect
- System / Software Architecture track
- posisi IT yang berhubungan dengan review sistem & dokumentasi teknis

**Untuk technical interviewer**
Bagian paling relevan: model data 15 tabel, state machine antar-modul, business rules bernomor, keputusan arsitektur (ADR-001–012), serta kontradiksi yang jujur dilaporkan (mis. Zod terpasang tapi 0 import — ADR-011).

**Untuk pembaca non-teknis**
Mulai dari **Project Overview**, **Business Rules Utama**, dan **Struktur Dokumentasi** — tidak perlu memahami Next.js untuk mengikuti.

> **Catatan penting:** ini **portfolio analisis dari kode sumber milik pihak ketiga** (dilaporkan apa adanya, termasuk kelemahannya), bukan klaim bahwa saya menulis aplikasi tersebut, dan bukan representasi sistem produksi resmi apa pun.

---

## 3. Kenapa review dokumentasi, bukan menulis program baru?

Karena titik yang ingin didemonstrasikan adalah **cara membaca sistem**.

Sebuah aplikasi bisa saja ditulis ulang dari requirement yang tersedia — tapi nilai analitis justru terlihat saat kode sudah ada: apa aturan sebenarnya, di mana state berubah, mana yang jadi source of truth, dan kontradiksi apa yang tersembunyi di antara dokumentasi lama dan kode.

Repository ini diposisikan sebagai:

> **"Bukti bahwa saya mampu membaca sebuah sistem nyata dan menurunkannya kembali menjadi rancangan sistem yang dapat dipertanggungjawabkan — lengkap dengan bukti per-klaim (`file:baris`)."`**

---

# 4. Project Overview

## Sumber yang Di-review (Git)

| Field | Value |
|---|---|
| **Repo** | `https://github.com/Kurohiko-id/Kitab-Link-Self-Hosted` |
| **Versi** | `v0.9.0` (`package.json:3`) |
| **Mode review** | Ekstraksi dari kode (read-only) · kedalaman *deep* · cakupan **fungsional** |
| **Perubahan kode** | Tidak ada — seluruh artefak berada di `docs/` |

## Domain

Link-in-bio self-hosted berskala **single-user, multi-page** — satu akun operator, banyak halaman publik `/{slug}` dengan tema, link, dan automasi masing-masing.

## Tujuan

- **Latihan:** membedah domain nyata jadi requirement, data model, business rules, dan process flow.
- **Portofolio:** artefak review yang bisa dibedah saat interview, tiap klaim berbekal bukti `file:baris`.

---

# 5. Tech Stack (Recovered dari Kode)

> Bukan rekomendasi — **fakta yang dibaca dari kode sumber**. Detail + evidence per layer: `docs/architect/00-Global/TECH-STACK-MASTER.md`.

| Layer | Technology | Catatan |
|---|---|---|
| Runtime | **Node.js 22** (`node:22-slim`) | dipatok lewat base image Docker |
| Language | **TypeScript 5** (`strict: true`) | — |
| Framework | **Next.js 16.3.5** (App Router, Server Actions) | `output: "standalone"` |
| UI | **React 19.2.8** + **Tailwind CSS 4** + **shadcn/ui** | — |
| Database | **SQLite** (better-sqlite3, WAL, FK ON) | tanpa pool, koneksi tunggal |
| ORM | **Drizzle ORM** | migrasi `drizzle/0000–0035` |
| Auth | **Custom** — argon2id + stateless HMAC cookie | 0 dependensi auth eksternal |
| Cron | **node-cron in-process** (6 job) | di `instrumentation.ts`, tanpa queue external |
| Cache | **Tidak ada** cache layer terstruktur | Map in-process + fetch revalidate |
| Deploy | **Docker multi-stage → GHCR** | image multi-arch |

**Prinsip stack sumber:** self-host sesederhana mungkin — 1 container + 1 volume, tanpa service database terpisah, tanpa SMTP.

---

# 6. Operasional Flow / State Machine

Contoh state machine yang didokumentasikan (penuh di `docs/analyst/BAB-6-Integrasi-Automasi/PROCESS-FLOW.md`, PF-011):

```text
event (link diklik, live status berubah, dst.)
        │
        ▼
   ┌─────────┐   cron ~10 detik    ┌────────┐
   │ pending │────────────────────►│  sent  │
   └────┬────┘                     └────────┘
        │ gagal (timeout / non-2xx)
        ▼
   attempts + 1 → backoff berjenjang
   [10s → 30s → 1m → 5m → 30m]
        │
        ├── attempts < 5  → kembali pending (nextRetryAt)
        └── attempts = 5  → ┌────────┐
                            │ failed │  (permanen)
                            └────────┘
```

### Prinsip state machine

Status tidak diperlakukan sebagai label bebas. Contoh aturan yang menjadi inti desain:

- **Klik link** selalu melewati `/r/{linkId}` dulu — analytics terekam **sebelum** redirect 302 (BR-025).
- **Link mati** baru ditandai `dead` setelah **2× gagal berturut-turut**, counter reset saat sukses (BR-024).
- **Antrean webhook** berhenti permanen setelah 5 percobaan — backoff dijadwalkan lewat `nextRetryAt`, bukan loop (BR-035).

---

# 7. Business Rules Utama

Aturan bernomor (BR-014–BR-048, total 35) didefinisikan lengkap di `docs/analyst/business-rules.md`. Sorotan:

- **Slug wajib valid & bukan reserved** (`login`, `setup`, `dashboard`, `api`, `r`, `uploads`) — slug yang sama dengan route top-level akan men-shadow-nya permanen (BR-017).
- **Custom domain diblokir dari path admin** (`/dashboard`, `/login`, `/setup`, `/api`) — di-block 404 oleh proxy (BR-019).
- **Reset password tanpa SMTP** — token dicetak ke log container, berlaku 15 menit (BR-016).
- **Semua upload gambar wajib lewat `processImage()`** → WebP; file asli tidak pernah disimpan (BR-022).
- **API token di-hash SHA-256** (entropi tinggi, butuh lookup cepat); raw token hanya tampil sekali (BR-030/031).
- **`targetId` automasi sengaja bukan FK** (polymorphic ke grup ATAU link) — tidak dijamin konsisten oleh DB (BR-039).

---

# 8. Roles & Actors

Bukan sistem multi-role — model akunnya sederhana:

| Actor | Fokus utama |
|---|---|
| **Operator** | Satu-satunya role dashboard: kelola page, link, tema, integrasi (session + opsi 2FA) |
| **Pengunjung** | Halaman publik `/{slug}` — akses lewat password page / kode akses sementara |
| **Sistem eksternal** | Konsumsi REST API v1 dengan `Bearer klk_...` + scope izin |
| **Cron in-process** | 6 job terjadwal (webhook queue, automasi, live badge, RSS, cek update, cek link mati) |

---

# 9. Data yang Dimodelkan

**15 tabel SQLite** (Drizzle ORM), `pages` sebagai aggregate root.

| Blok | Isi |
|---|---|
| Akun & akses | `users`, `api_tokens`, `page_access_codes` |
| Konten | `pages`, `link_groups`, `links`, `themes` |
| Observasi | `analytics_events`, `activity_logs` |
| Automasi | `webhooks`, `webhook_queue`, `scheduled_rules`, `live_badges`, `discord_widgets`, `content_feeds` |

Statistik dokumentasi: **54 use case (UC-001–054)** · **90 FR** · **35 BR** · **12 process flow (PF-001–012)** · **12 ADR** · **5 temuan bug**.

### Contoh format data

| Jenis | Format |
|---|---|
| Halaman publik | `/{slug}` — pola `^[a-z0-9]+(-[a-z0-9]+)*$` |
| Token API | `klk_...` (prefix hanya ada sekali saat pembuatan) |
| Signature webhook | Header `X-KitabLink-Signature` (HMAC-SHA256) |

---

# 10. Chapter dan Hubungannya

```text
        ┌────────────────────────┐
        │  BAB-2 · Manajemen Page │  ← aggregate root (pages)
        └───────────┬────────────┘
     ┌──────────────┼──────────────────┬──────────────────┐
     ▼              ▼                  ▼                  ▼
 BAB-1          BAB-3             BAB-4              BAB-5
 Otentikasi     Link & Grup       Tema & Profil      Publik & Analytics
 & Akun         (children)        (themeId → page)   (views & logs)
                                                  │
                                                  ▼
                                             BAB-6
                                     Integrasi & Automasi
                                  (webhook, cron, RSS, Discord)
```

Tiga integrasi inti:

1. **Klik selalu lewat `/r/{linkId}`** → analytics tercatat sebelum pengunjung berpindah.
2. **Automasi menulis `isVisible` target** → dibaca ulang oleh render halaman publik.
3. **`pages.themeId` + `profileJson`** → menentukan seluruh tampilan halaman publik.

---

# 11. Scope dan Non-Scope

## In Scope

- ekstraksi requirement, use case, ERD, data dictionary, business rules per chapter
- process flow & state machine terverifikasi kode
- review bug & security posture (documentation-only)
- pemulihan ADR dari kode (`Recovered` / `Recovered-Contradictory`)
- cross-module review, API endpoint map, integration detail

## Out of Scope

- **perubahan kode apa pun** terhadap repo sumber
- implementasi aplikasi ulang / fitur baru
- audit dependency (`npm audit`) — akan mengubah direktori kerja
- NFR mapping penuh, SRS-MASTER terpisah, spesifikasi deployment produksi
- klaim bahwa seluruh praktik lapangan/operasional terpetakan sempurna

---

# 12. Struktur Dokumentasi Repository

```text
docs/
├── analyst/                        # hasil analisis (SA docs, Bahasa Indonesia)
│   ├── 00-Global/
│   │   ├── DATA-DICT-MASTER.md     # 15 tabel, atribut + validasi
│   │   ├── ERD-MASTER.md           # ASCII box art, relasi penuh
│   │   ├── REQUIREMENTS-MATRIX.md  # FR ↔ UC ↔ BR traceability
│   │   ├── PROCESS-FLOW-MASTER.md  # index PF-001–012
│   │   ├── GLOSSARY.md
│   │   └── NOTATION.md             # GLOBAL-00 (aturan notasi)
│   ├── BAB-1-Otentikasi-Akun/      # tiap BAB berisi:
│   ├── BAB-2-Manajemen-Page/       #   USE-CASE.md
│   ├── BAB-3-Link-Grup/            #   ERD-EXCERPT.md
│   ├── BAB-4-Tema-Profil/          #   DATA-DICT-EXCERPT.md
│   ├── BAB-5-Publik-Analytics/     #   PROCESS-FLOW.md
│   ├── BAB-6-Integrasi-Automasi/
│   ├── business-rules.md           # BR-014–048
│   ├── bug-report.md               # 5 temuan + evidence
│   ├── security-report.md          # posture self-hosted
│   ├── extraction-report.md        # mode & cakupan ekstraksi
│   ├── NUMBERING-LOG.md            # register nomor UC/FR/BR/PF
│   └── ISSUE.md
│
├── architect/                      # artefak arsitektur (recovered dari kode)
│   ├── 00-Global/
│   │   ├── TECH-STACK-MASTER.md    # stack + evidence per layer
│   │   ├── SYSTEM-ARCH-MASTER.md
│   │   ├── DATA-ARCH-MASTER.md
│   │   ├── SECURITY-ARCH-MASTER.md
│   │   ├── INFRA-CLOUD-MASTER.md
│   │   ├── TECH-RISK-MASTER.md
│   │   ├── API-STANDARD.md
│   │   ├── ADR-MASTER.md
│   │   └── ADR/                    # ADR-001 … ADR-012
│   ├── BAB-1…BAB-6/                # SECURITY-RULES · PERFORMANCE-NOTES
│   │                               # API-ENDPOINTS · INTEGRATION-DETAIL
│   └── extraction-report-arch.md
│
└── Kitab-Link-Self-Hosted/         # clone sumber (read-only)
```

> `docs/architect/` diposisikan sebagai **lapisan pemulihan arsitektur**: ia menjawab pertanyaan *"Kalau sistem ini dibangun ulang dari nol, keputusan teknis apa yang sudah terkandung di dalamnya?"* — seluruh isi berstatus **Recovered**, bukan rekomendasi baru.

---

# 13. What I Am Demonstrating

Repository ini tidak menunjukkan sebanyak mungkin halaman. Yang ditunjukkan:

**Kode → Requirement → Data → Rule → Process → Temuan → Keputusan**

```text
Kode produksi (v0.9.0)
   ↓ ekstraksi (read-only)
Requirement + traceability (FR/UC/BR)
   ↓
Model data 15 tabel + relasi
   ↓
Business rules bernomor (BR-014–048)
   ↓
Process flow + state machine (PF-001–012)
   ↓
Review bug & security (evidence file:baris)
   ↓
ADR-001–012 (keputusan arsitekturRecovered)
```

Contoh pertanyaan interview yang bisa dilontarkan — jawabannya tersebar di `docs/analyst/`:

- Kapan stok/analog data berkurang — saat *event*, bukan saat form disimpan?
- Kenapa `analytics_events.linkId` bukan FK?
- Apa yang terjadi kalau custom domain mengakses `/dashboard`?
- Kenapa token di-hash SHA-256, bukan argon2?
- Bagaimana sistem mencegah webhook mengirim duplikat setelah restart?
- Kontradiksi apa yang kamu temukan antara dokumentasi lama dan kode?

---

# 14. Why This Repository Is Interview-Friendly

Urutan baca yang disarankan:

```text
README.md  (file ini)
   ↓
docs/analyst/extraction-report.md      ← mode & cakupan review
   ↓
docs/analyst/00-Global/REQUIREMENTS-MATRIX.md
   ↓
docs/analyst/BAB-1 … BAB-6             ← USE-CASE → ERD-EXCERPT → PROCESS-FLOW
   ↓
docs/analyst/business-rules.md
   ↓
docs/analyst/bug-report.md + security-report.md
   ↓
docs/architect/extraction-report-arch.md  ← kontradiksi & What Was NOT Found
   ↓
docs/architect/00-Global/TECH-STACK-MASTER.md → SYSTEM-ARCH-MASTER.md → ADR/
```

Dengan urutan itu, pembaca melihat hubungan **keputusan kode ↔ keputusan dokumentasi**, bukan diagram yang berdiri sendiri.

---

# 15. Development Path (Bila Diperdalam)

```text
Audit independen dokumen (review lintas-dokumentasi)
      ↓
Perdalam kategori non-fungsional (NFR mapping, deployment spec)
      ↓
(Eksperimen) implementasi ulang terisolasi per chapter
      ↓
Feature test fokus 2 area paling rapuh logikanya:
  1. pengurangan data terjadi tepat satu kali
  2. transisi state mengikuti business rule
```

---

# 16. Limitations

- Review bersifat **dokumentasi** — tidak ada eksekusi runtime, tidak ada benchmark nyata.
- Cakupan **fungsional**; NFR & beberapa kategori sengaja dipangkas.
- Beberapa penilaian dilabeli **⚠️ INFERENSI** bila alasan tidak ada di kode — tidak dikarang seolah fakta.
- Repo sumber terus berkembang; dokumen ini merepresentasikan **v0.9.0** pada tanggal ekstraksi.

---

# 17. Closing

> **Saya tidak sedang menunjukkan "aplikasi link-in-bio yang saya bangun".**
>
> Saya sedang menunjukkan bagaimana saya **membaca sebuah sistem yang sudah berjalan**, menemukan aturan penting dan kontradiksinya, menghubungkan data dan proses, lalu menurunkannya kembali menjadi dokumentasi sistem yang siap dipertahankan — dengan bukti per-klaim.

Repository ini adalah **analysis portfolio** dengan review kode sebagai metode, bukan sebagai pengganti implementasi.

---

## Source of Decisions

Keputusan format, cakupan, dan penomoran pada README ini mengacu pada:

- `docs/analyst/extraction-report.md` — mode ekstraksi (Opsi A), cakupan fungsional, apa yang sengaja tidak dibuat;
- `docs/architect/extraction-report-arch.md` — kedalaman review, kontradiksi temuan, dan batasan ("What Was NOT Found");
- `docs/analyst/NUMBERING-LOG.md` — register nomor UC / FR / BR / PF.
