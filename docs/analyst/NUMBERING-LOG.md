# NUMBERING-LOG — Kitab Link

Register nomor ID lintas dokumen `docs/analyst/`. **Nomor legacy tidak boleh dipakai ulang.**

## Blocks (reserved per chapter)

| Block Range | Chapter | Next Available | Status |
|-------------|---------|----------------|--------|
| 001–050 | BAB-1-Otentikasi-Akun | 041 | Active |
| 051–100 | BAB-2-Manajemen-Page | 081 | Active |
| 101–150 | BAB-3-Link-Grup | 141 | Active |
| 151–200 | BAB-4-Tema-Profil | 181 | Active |
| 201–250 | BAB-5-Publik-Analytics | 241 | Active |
| 251–300 | BAB-6-Integrasi-Automasi | 291 | Active |

## Usage Rules

1. Setiap chapter baru = buat blok **50** nomor.
2. Update kolom **Next Available** setiap kali nomor dipakai.
3. BR hasil inferensi (Medium/Low confidence) tetap mengambil nomor dari blok chapter-nya.
4. **Tidak ada nomor legacy di project ini** — repo clone belum pernah punya `NUMBERING-LOG.md` sebelumnya (dicek: file tidak ada sebelum sesi ini dibuat).

## UC Numbering (global, bukan per chapter)

| Range | Chapter | Used |
|-------|---------|------|
| UC-001 – UC-009 | BAB-1-Otentikasi-Akun | 9 |
| UC-010 – UC-017 | BAB-2-Manajemen-Page | 8 |
| UC-018 – UC-026 | BAB-3-Link-Grup | 9 |
| UC-027 – UC-034 | BAB-4-Tema-Profil | 8 |
| UC-035 – UC-042 | BAB-5-Publik-Analytics | 8 |
| UC-043 – UC-054 | BAB-6-Integrasi-Automasi | 12 |
| **Total** | | **54** |

## FR Numbering

| Range | Chapter |
|-------|---------|
| FR-001 – FR-012 | BAB-1 |
| FR-013 – FR-024 | BAB-2 |
| FR-025 – FR-042 | BAB-3 |
| FR-043 – FR-056 | BAB-4 |
| FR-057 – FR-068 | BAB-5 |
| FR-069 – FR-090 | BAB-6 |

## BR Numbering

| Range | Notes |
|-------|-------|
| BR-001 – BR-013 | **Tidak pernah ada** (repo clone tanpa legacy) |
| BR-014 – BR-048 | Definisi lengkap: [`business-rules.md`](business-rules.md) · sumber: `00-Global/DATA-DICT-MASTER.md` · Next available: **BR-049** |

## PF Numbering (global)

| Range | Chapter | Used |
|-------|---------|------|
| PF-001 – PF-002 | BAB-1-Otentikasi-Akun | 2 |
| PF-003 – PF-004 | BAB-2-Manajemen-Page | 2 |
| PF-005 – PF-006 | BAB-3-Link-Grup | 2 |
| PF-007 – PF-008 | BAB-4-Tema-Profil | 2 |
| PF-009 – PF-010 | BAB-5-Publik-Analytics | 2 |
| PF-011 – PF-012 | BAB-6-Integrasi-Automasi | 2 |
| **Total** | | **12** · Next available: **PF-013** |

> Index & relationship map PF: [`00-Global/PROCESS-FLOW-MASTER.md`](00-Global/PROCESS-FLOW-MASTER.md)
