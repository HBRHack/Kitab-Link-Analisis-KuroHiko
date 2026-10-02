# NOTATION — Matriks Pemilihan Notasi (GLOBAL-00)

Berlaku untuk seluruh dokumen di `docs/analyst/` project **Kitab Link (Self-Hosted)**.

| Jenis Diagram / Artefak | Notasi Wajib | Format File | Keterangan |
|-------------------------|--------------|-------------|------------|
| ERD (Entity Relationship) | **ASCII / Unicode Box Art** | Markdown fenced ```markdown block | Bukan Mermaid, sesuai `ERD-FORMAT.md` |
| BPMN | **Mermaid JS** | Markdown fenced ```mermaid block | Flow lintas modul |
| Sequence Diagram | **Mermaid JS** | Markdown fenced ```mermaid block | Interaksi client/server/API |
| Activity Diagram | **Mermaid JS** | Markdown fenced ```mermaid block | Alur proses dalam action |
| Use Case Diagram | **Mermaid JS** | Markdown fenced ```mermaid block | Opsional; UC utama ditulis teks |
| Tabel (matrix, data dict, value list) | Markdown table | `.md` | — |
| Laporan temuan (bug/security) | Markdown table + teks | `.md` | Confidence level wajib |

## Aturan

1. **ERD selalu ASCII box art.** Menggunakan Mermaid untuk ERD = CRITICAL FAILURE.
2. **Empat notasi diagram (BPMN, Sequence, Activity, Use Case) selalu Mermaid.** Menggunakan ASCII untuk keempatnya = CRITICAL FAILURE.
3. Dokumen ringkas (Opsi A) boleh **melewati diagram non-ERD** sepenuhnya — cukup tabel dan prosa. Ketika diagram dibutuhkan, wajib ikuti matriks di atas. *(Aturan ini berlaku untuk ekstraksi awal; kelengkapan per-chapter kini sudah dilengkapi — lihat tabel Penerapan.)*
4. ID wajib konsisten lintas dokumen: `UC-NNN`, `FR-NNN`, `BR-NNN`, `NFR-*`.

## Penerapan di Dokumen Ini

Dokumen ini awalnya dihasilkan dalam **mode ringkas-esensial** (Opsi A — reviewer-only, tanpa perbaikan kode), lalu **dilengkapi** pada sesi berikutnya (ERD-EXCERPT, DATA-DICT-EXCERPT, PROCESS-FLOW per chapter — seluruh isi diverifikasi dari kode):

| Artefak | Status | Notasi Dipakai |
|---------|--------|----------------|
| ERD-MASTER | ✅ Ada | ASCII box art |
| DATA-DICT-MASTER | ✅ Ada | Markdown table |
| REQUIREMENTS-MATRIX | ✅ Ada | Markdown table |
| GLOSSARY | ✅ Ada | Markdown table |
| USE-CASE per chapter | ✅ Ada | Markdown teks + tabel |
| ERD-EXCERPT per chapter | ✅ Ada (6 file) | ASCII box art (extract dari ERD-MASTER) |
| DATA-DICT-EXCERPT per chapter | ✅ Ada (6 file) | Markdown table (extract dari DATA-DICT-MASTER) |
| PROCESS-FLOW per chapter | ✅ Ada (6 file, PF-001–012) | ASCII flowchart linear · **Mermaid** untuk sequence/activity kompleks (PF-001, PF-003, PF-006, PF-011) |
| PROCESS-FLOW-MASTER | ✅ Ada | Markdown index |
| business-rules.md | ✅ Ada (BR-014–048) | Markdown table |
