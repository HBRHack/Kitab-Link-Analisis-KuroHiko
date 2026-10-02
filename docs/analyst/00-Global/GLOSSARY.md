# Glossary — Kitab Link

Istilah bisnis dan teknis yang dipakai lintas dokumen. Sumber: kode (`lib/`, `app/`) dan `README.id.md`.

| Istilah | Definisi | Catatan / Contoh |
|---------|----------|------------------|
| **Page** | Satu halaman link-in-bio independen dalam satu akun, punya slug sendiri (`/main`, `/side-project`) | Aggregate root; link, grup, tema, analytics terikat ke page, bukan ke akun |
| **Link** | Item tombol individual di sebuah page | Kolom `url` dipakai ulang untuk 10 `linkType` berbeda |
| **Group (Grup)** | Kontainer untuk mengelompokkan link dalam satu page | Punya `orderIndex` dan `isVisible` sendiri |
| **Ungrouped** | Link yang tidak berada di grup mana pun | `links.groupId = null` |
| **Theme** | Library gaya (warna, font, tombol, background) milik akun, bisa dipakai lintas page | Ditempel ke page via `pages.themeId` |
| **Theme tokens** | Representasi JSON dari sebuah theme | Disimpan di `themes.tokens_json` |
| **Preset** | Theme bawaan siap pakai di galeri | Klik preset → di-duplicate jadi entry library (preset asli tidak pernah diedit) |
| **Custom CSS** | CSS tambahan yang disuntik ke halaman publik | Maksimal 20.000 karakter |
| **Slug** | Pengenal URL sebuah page | Regex `^[a-z0-9]+(-[a-z0-9]+)*$`, tidak boleh reserved |
| **Primary page** | Page yang tampil di root `/` | `users.primaryPageId`; null = `/` redirect ke `/dashboard` |
| **Custom domain** | Hostname eksternal yang diarahkan ke satu page | `domainType = custom_domain`; diverifikasi otomatis saat pertama ke-hit |
| **Domain verified** | Tanda bahwa DNS + reverse proxy sudah mengarah benar | Diset otomatis oleh `proxy.ts` |
| **Password-protected page** | Page yang mewajibkan password sebelum tampil | `pages.passwordHash` |
| **Access code (kode akses)** | Password tambahan yang berlaku terbatas waktu | `page_access_codes`; "revoke" = hapus baris |
| **Page session** | Cookie izin pengunjung setelah berhasil unlock | Cookie `kitab_page_{id}`, 30 hari, terpisah dari session akun |
| **Session (akun)** | Cookie HMAC identitas operator dashboard | Cookie `kitab_session`, 7 hari, payload `{userId}.{expires}.{signature}` |
| **Setup token** | Token one-time untuk membuat akun admin pertama | Dicetak ke log container, tidak dipersist ke disk |
| **Master secret** | Secret acak 32 byte yang digenerate sekali di first boot | Disimpan di `data/.secret` (mode 0600); dipakai sign session + derive key enkripsi |
| **TOTP / 2FA** | Autentikasi dua faktor berbasis Time-based OTP | Secret di-encrypt AES-256-GCM; backup code hash argon2id, sekali pakai |
| **API token** | Token akses API eksternal, prefix `klk_` | Di-hash SHA-256; berisi scope izin |
| **Scope** | Izin granular pada API token | `links:read`, `links:write`, `discord:read`, `discord:write` |
| **Webhook** | Endpoint keluar yang menerima event dari aplikasi | Ditandatangani HMAC-SHA256 via `X-KitabLink-Signature` |
| **Webhook queue** | Antrean pengiriman webhook + retry | Diproses cron tiap 10 detik, maks 5 percobaan |
| **Automation / Scheduled rule** | Aturan show/hide grup atau link otomatis | Trigger: `youtube_live`, `weekly_schedule`, `manual` |
| **Live badge** | Badge "sedang live" di halaman publik | 1 row per page; terpisah dari scheduled rule |
| **Content feed** | Sumber RSS/Atom yang item barunya otomatis jadi link | Ditulis ke `groupId` tertentu |
| **Dead link** | Link yang gagal diakses 2x berturut-turut | Dicek harian jam 03:00 |
| **Rich card** | Gaya link dengan thumbnail + deskripsi | `displayStyle = rich`; thumbnail bisa auto-fetch dari OG image |
| **Icon row** | Baris ikon sosial media di atas/bawah | `displayStyle = icon`, dibedakan oleh `iconPosition` |
| **Featured** | Penanda link unggulan | `links.featured` |
| **Streamer mode** | Mode yang menyembunyikan info sensitif di dashboard | Toggle disimpan di profil |
| **Activity log** | Riwayat perubahan per page | `action` = key generik, pesan tampilan dikomposisi dari kamus i18n |
| **Analytics** | Metrik views & klik per page/link | Termasuk referrer, device, negara |
| **Campaign source** | `utm_source` / `ref` pada URL page | Menutup celah in-app browser yang tak kirim Referer |
| **Board** | Tampilan drag & drop link + grup di dashboard | Disimpan sebagai urutan `orderIndex` |
| **Backup / Restore** | Export/import JSON (theme + profile + links) | `BACKUP_VERSION = 3`; gambar sengaja tidak ikut |
| **OG image** | Gambar pratinjau saat page dibagikan | Auto-generate dari warna theme bila user tidak upload custom |
| **Process image** | Util konversi gambar via sharp → resize → WebP | Wajib untuk semua upload gambar |
| **Self-hosted** | Berjalan di server/infra milik user sendiri | Deploy Docker 1 command, konfigurasi dari dashboard |
| **Single-user, multi-page** | Satu akun admin, banyak page independen | Prinsip inti arsitektur |
| **Proxy (Next.js 16)** | Pengganti `middleware.ts` | File `proxy.ts`, export `proxy()`, runtime Node.js |
| **Reserved slug** | Slug yang dilarang karena bentrok dengan route top-level | `login`, `setup`, `dashboard`, `api`, `r`, `uploads` |
| **Gate #6 (Value Consistency)** | Pemeriksaan konsistensi literal value di kode vs Value List di data dictionary | Lihat `extraction-report.md` |
