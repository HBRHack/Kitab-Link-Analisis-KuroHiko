# PROCESS-FLOW — Master Index

> Index global seluruh process flow (PF) di `docs/analyst/BAB-*/PROCESS-FLOW.md`. Setiap PF-XXX yang dibuat per chapter **wajib** tercatat di sini (aturan format `analyst-extract-code/PROCESS-FLOW-FORMAT.md`).
>
> Notasi: flowchart linear = ASCII · sequence/activity/BPMN = Mermaid (GLOBAL-00, [`NOTATION.md`](NOTATION.md)).

## Index PF

| PF ID | Judul | Chapter File | Trigger | Goal | Mermaid? |
|-------|-------|--------------|---------|------|----------|
| PF-001 | Login dengan Challenge 2FA | [`BAB-1-Otentikasi-Akun/PROCESS-FLOW.md`](../BAB-1-Otentikasi-Akun/PROCESS-FLOW.md) | Form `/login` | Session cookie valid (atau challenge TOTP) | ✅ Sequence |
| PF-002 | Reset Password Tanpa SMTP | [`BAB-1-Otentikasi-Akun/PROCESS-FLOW.md`](../BAB-1-Otentikasi-Akun/PROCESS-FLOW.md) | `/login/forgot-password` | Password baru terpasang, token hangus | — (ASCII) |
| PF-003 | Resolusi Host Custom Domain | [`BAB-2-Manajemen-Page/PROCESS-FLOW.md`](../BAB-2-Manajemen-Page/PROCESS-FLOW.md) | Request HTTP masuk (`proxy.ts`) | Host custom → `/{slug}`, admin 404 | ✅ Sequence |
| PF-004 | Unlock Halaman Berpassword | [`BAB-2-Manajemen-Page/PROCESS-FLOW.md`](../BAB-2-Manajemen-Page/PROCESS-FLOW.md) | Buka `/{slug}` berpassword | Cookie grant 30 hari | — (ASCII) |
| PF-005 | Simpan Link (Tambah/Edit) | [`BAB-3-Link-Grup/PROCESS-FLOW.md`](../BAB-3-Link-Grup/PROCESS-FLOW.md) | Submit modal link | Baris `links` valid tersimpan | — (ASCII) |
| PF-006 | Persist Board (Drag & Drop) | [`BAB-3-Link-Grup/PROCESS-FLOW.md`](../BAB-3-Link-Grup/PROCESS-FLOW.md) | Lepas kartu di board | `orderIndex`/`groupId` atomic | ✅ Activity |
| PF-007 | Simpan Perubahan Theme | [`BAB-4-Tema-Profil/PROCESS-FLOW.md`](../BAB-4-Tema-Profil/PROCESS-FLOW.md) | Submit editor tema | `tokensJson` ter-merge | — (ASCII) |
| PF-008 | Simpan Profil Page + Custom CSS | [`BAB-4-Tema-Profil/PROCESS-FLOW.md`](../BAB-4-Tema-Profil/PROCESS-FLOW.md) | Submit Settings → Profil | `profileJson` ter-update (css ≤ 20.000) | — (ASCII) |
| PF-009 | Klik Link — Redirect + Analytics | [`BAB-5-Publik-Analytics/PROCESS-FLOW.md`](../BAB-5-Publik-Analytics/PROCESS-FLOW.md) | Klik kartu link | `link_click` terekam, 302 ke tujuan | — (ASCII) |
| PF-010 | Render Halaman Publik + Page View | [`BAB-5-Publik-Analytics/PROCESS-FLOW.md`](../BAB-5-Publik-Analytics/PROCESS-FLOW.md) | GET `/{slug}` | HTML terkirim + view tercatat | — (ASCII) |
| PF-011 | Proses Antrean Webhook | [`BAB-6-Integrasi-Automasi/PROCESS-FLOW.md`](../BAB-6-Integrasi-Automasi/PROCESS-FLOW.md) | Cron `*/10 detik` | Event `sent` / `failed` setelah 5× | ✅ Activity |
| PF-012 | Automasi Show/Hide (Scheduled Rules) | [`BAB-6-Integrasi-Automasi/PROCESS-FLOW.md`](../BAB-6-Integrasi-Automasi/PROCESS-FLOW.md) | Cron `*/2 menit` | `isVisible` target sinkron + log automation | — (ASCII) |

**Total: 12 PF** · Next available: **PF-013**

## Relationship Map (lintas chapter)

```
PF-001 (Login) --gagal/lupa--> PF-002 (Reset) --sukses--> PF-001
PF-003 (Host resolution) --> UC-035 render --> PF-004 (unlock) --> PF-009/PF-010 (BAB-5)
PF-005 (Simpan link) <--> PF-006 (Persist board)
PF-007 (Theme) + PF-008 (Profil) --> mempengaruhi render PF-010 (BAB-5)
PF-010 (Render) --klik--> PF-009 (Redirect /r)
PF-011 (Webhook queue) & PF-012 (Automasi) --> menulis isVisible (PF-010 membacanya)
```

## Catatan

- Chapter BAB-1..6 hanya memuat PF yang **diverifikasi dari kode**; tidak ada alur karangan.
- PF baru = tambah baris di atas + tulis di file chapter, lanjutkan dari PF-013.
