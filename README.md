<p align="center">
<img width="250" height="250" alt="vogenik-logo" src="https://github.com/user-attachments/assets/209a0a69-6a24-4ea3-9521-2f04294874b3" />
</p>
<h1 align="center">Vogenikbot (Voucher Generator MikroTik)</h1>
<p align="center"><em>Bot WhatsApp Integrasi MikroTik untuk Manajemen Voucher Hotspot</em></p>

---

## 1. Deskripsi Umum
**Vogenik** (singkatan dari **Vo**ucher **Gen**erate M**ik**roTik) adalah sistem otomasi berbasis Node.js yang menghubungkan WhatsApp dengan router MikroTik. Program ini dirancang untuk memudahkan manajemen, pembuatan, dan pemantauan voucher Hotspot RT/RW Net secara langsung melalui *chat* WhatsApp. 

Selain beroperasi sebagai bot WhatsApp, Vogenikbot juga bertindak sebagai **Web Server (berjalan di Port 3030)** yang berfungsi untuk:
1. Melayani halaman antarmuka (Dashboard HTML) untuk pemantauan data menggunakan **Magic Link** rahasia tanpa memerlukan proses *login*.
2. Menerima jalur masuk data (*Webhook*) terenkripsi (*Secret Token*) langsung dari router MikroTik setiap kali ada pelanggan yang *login* atau vouchernya habis masa aktifnya (*expired*).

---

## 2. Cara Kerja dan Alur Sistem

Alur kerja Vogenikbot berjalan secara dua arah (*Two-way communication*):

### A. Dari WhatsApp ke MikroTik (Pembuatan Voucher)
1. **Perintah Diterima:** Pengguna (Superadmin atau Admin) mengirimkan perintah pembuatan voucher melalui WA (contoh: `Budi VC1`).
2. **Validasi Otorisasi:** Sistem mengecek apakah pengirim pesan terdaftar sebagai Superadmin atau Admin di dalam database internal. Jika tidak terdaftar, perintah akan diabaikan secara diam-diam (*silent drop*) untuk mencegah penyusup.
3. **Generate Kode:** Sistem memproses perintah, membuat kode voucher acak yang unik (kombinasi huruf dan angka).
4. **Injeksi MikroTik:** Bot menggunakan modul `node-routeros` (mendukung koneksi API-SSL) untuk login ke API MikroTik dan menambahkan *User* baru tersebut ke dalam `/ip hotspot user` lengkap dengan *Profile* dan *Limit-Uptime*-nya.
5. **Respon WA:** Setelah sukses diinjeksi, bot membalas pesan WA dengan detail voucher yang siap diserahkan ke pelanggan.

### B. Dari MikroTik ke WhatsApp (Notifikasi Real-time & Auto-Clean)
Sistem ini dirancang agar MikroTik bisa "melapor" ke WhatsApp.
1. **Injeksi Script On-Login & Token:** Saat Superadmin/Admin membuat profil paket internet baru melalui bot, bot secara cerdas menanamkan sebuah script khusus ke pengaturan *On-Login* di profil MikroTik tersebut beserta sebuah *Secret Token*.
2. **Trigger Webhook:** Ketika ada pelanggan yang berhasil *login* di halaman Hotspot menggunakan kode vouchernya, script *On-Login* di MikroTik akan berjalan dan "menembak" URL Webhook bot sambil membawa *Secret Token* tersebut.
3. **Notifikasi Superadmin:** Webhook memvalidasi kecocokan token, memperbarui database status voucher menjadi "terjual", dan mengirim log notifikasi ke WA Superadmin (dan Admin pembuat voucher) bahwa ada user yang baru saja login.
4. **Auto-Clean & Expired:** Script yang ditanam juga mencakup sistem penjadwalan (*Scheduler*). Jika durasi (*limit-uptime*) pelanggan habis, MikroTik akan memutus pelanggan, menghapus user tersebut dari MikroTik secara permanen agar tidak menumpuk (Auto-Clean), lalu menembak Webhook bot untuk mengubah status voucher di database menjadi "expired".

---

## 3. Struktur Database
Program ini bersifat mandiri dan ringan karena tidak memerlukan database eksternal (seperti MySQL). Semua data disimpan dalam bentuk file `.json` secara lokal:

* **`config.json`**: Penyimpanan inti untuk sistem. Menyimpan nomor WA Superadmin, IP Server STB, data login API MikroTik, nama server hotspot, IP Pool, *Webhook Token*, dan *Dashboard Magic Link*.
* **`admins.json`**: Menyimpan daftar nomor WA Admin yang sudah teregistrasi melalui tiket.
* **`profiles.json`**: Menyimpan daftar paket/profil internet yang sudah dibuat melalui bot. Menyimpan data kode paket, durasi (validity), dan harga.
* **`vouchers.json`**: Buku besar (Buku Kas) dari bot ini. Menyimpan seluruh riwayat voucher yang telah di-generate, pelanggan, siapa yang membuat, waktu pembuatan, waktu login, limit sisa waktu, dan status terkini (tersedia/terjual/expired). Terdapat juga rekaman **harga permanen**.
* **`tickets.json`**: Penyimpanan sementara untuk kode tiket registrasi Admin.

---

## 4. Daftar Perintah (Command) Lengkap

**Catatan:** Semua perintah (kecuali `!klaim_superadmin`) diketik tanpa menggunakan awalan tanda seru (`!`). Sistem mengenali huruf besar maupun kecil (Case-Insensitive).

### Konfigurasi Sistem (Hanya Superadmin)
Perintah ini digunakan untuk *setup* bot.
* `!klaim_superadmin`
  * **Fungsi:** Mengunci nomor pengirim sebagai Owner/Superadmin utama bot. Hanya bisa dieksekusi 1 kali saat bot baru menyala.
* `setstb [IP_ARMBIAN]`
  * **Fungsi:** Memberitahu bot alamat IP lokal STB itu sendiri agar bisa membuat URL Webhook yang benar.
* `setmikrotik [IP] [USER] [PASS] [PORT]`
  * **Fungsi:** Menautkan bot dengan router MikroTik. Sangat disarankan menggunakan port **8729** agar bot otomatis mengaktifkan enkripsi API-SSL. (Contoh: `setmikrotik 192.168.1.1 bot_wa password123 8729`).
* `setserver [NAMA_SERVER]`
  * **Fungsi:** Menetapkan ke server hotspot mana voucher akan dimasukkan.
* `setpool [NAMA_POOL]`
  * **Fungsi:** Menetapkan target IP Pool di MikroTik untuk profil internet yang dibuat bot. (Contoh: `setpool hs-pool-5`).

### Manajemen Admin (Hanya Superadmin)
* `addadmin [NAMA]`
  * **Fungsi:** Menghasilkan kode tiket acak yang nanti diberikan ke pegawai agar mereka bisa menghubungkan WA-nya ke bot sebagai Admin.
* `rmadmin [NAMA]`
  * **Fungsi:** Menghapus akses seorang Admin dari sistem.

### Registrasi Admin Baru
* `tiket [KODE_TIKET]`
  * **Fungsi:** Digunakan oleh pegawai dari HP mereka untuk mendaftar sebagai Admin dengan memasukkan kode dari Superadmin.

### Pembuatan & Manajemen Voucher (Superadmin & Admin)
* `panduan`
  * **Fungsi:** Menampilkan kembali panduan daftar perintah. (Tautan rahasia ke halaman Dashboard hanya akan ditampilkan kepada Superadmin).
* `addprof [NAMA_PAKET] [DURASI] [HARGA]`
  * **Fungsi:** Membuat profil paket baru di bot sekaligus menginjeksi profil + script auto-clean ke MikroTik.
  * **Contoh:** `addprof VC1 1d 5000` (Paket VC1, durasi 1 hari, harga Rp 5.000).
* `rmprof [NAMA_PAKET]`
  * **Fungsi:** Menghapus profil dari database bot dan sekaligus dari MikroTik.
  * **Contoh:** `rmprof VC1`.
* `[NAMA_PELANGGAN] [KODE_PAKET]`
  * **Fungsi:** Mencetak voucher baru sesuai profil yang dipilih dan menginjeksinya ke MikroTik.
  * **Contoh:** `Budi VC1` (Membuat voucher dari paket VC1 untuk Budi).

---

## 5. Fitur Keamanan Tingkat Lanjut dan Sistem Pelaporan
1. **Webhook Tokenization:** Menolak permintaan dari luar yang tidak memiliki *Secret Token* valid, sehingga mencegah manipulasi status voucher dari peretas.
2. **Magic Link Dashboard:** Menyembunyikan *dashboard panel* di URL acak yang di-*generate* otomatis agar tidak bisa diakses oleh publik di jaringan.
3. **Koneksi API-SSL:** Komunikasi antara STB dan MikroTik akan dienkripsi secara otomatis jika bot dikonfigurasi menggunakan port 8729.
4. **Permanent Price Lock:** Melindungi sistem pembukuan (*dashboard*) dari penurunan pendapatan akibat profil paket yang terhapus.
5. **Role-Based Access Control:** Fitur perintah dan visibilitas tautan panel dipisahkan secara ketat antara Superadmin dan Admin biasa.
6. **Rejeksi Tanpa Respon (Silent Drop):** Jika ada orang asing atau nomor tak dikenal yang mencoba mengetik perintah ke bot, bot tidak akan membalas pesan mereka.
7. **Audit Trail & Dual Notification:** Setiap Admin yang meng-generate voucher, nama admin tersebut akan dicatat. Superadmin dan Admin pembuat akan menerima notifikasi otomatis ketika pelanggan *login* atau vouchernya *expired*.
8. **Anti-Crash Connection:** Dilengkapi dengan sistem koneksi ulang (*auto-reconnect*) yang akan terus mencoba terhubung kembali ke server WA jika STB kehilangan akses internet.

---

## 6. Panduan Instalasi (Untuk STB Armbian Baru)

Ikuti langkah-langkah di bawah ini untuk menginstal Vogenikbot dari nol pada STB Armbian Anda:

**Langkah 1: Update Sistem Dasar**
```bash
sudo apt update && sudo apt upgrade -y
```

**Langkah 2: Install Node.js (Mesin Utama Bot)**
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

**Langkah 3: Unduh File Installer Vogenikbot**
```bash
wget -O vogenikbot_1.1.deb https://github.com/yoestration/vogenikbot/releases/download/v1.1/vogenikbot_1.1.deb
```

**Langkah 4: Eksekusi Instalasi**
```bash
sudo apt install ./vogenikbot_1.1.deb -y
```

**Langkah 5: Aktifkan Auto-Start Bot (Penting)**
Agar bot otomatis hidup saat STB di-restart atau menyala setelah mati listrik, jalankan perintah ini:
```bash
sudo systemctl enable vogenikbot
sudo systemctl start vogenikbot
```

**Langkah 6: Scan QR Code WhatsApp**
Setelah instalasi selesai, jalankan perintah di bawah ini untuk melihat pesan dan QR Code dari bot, lalu scan menggunakan aplikasi WhatsApp Anda (Tautkan Perangkat):
```bash
sudo journalctl -u vogenikbot -f
```
*(Tekan **Ctrl + C** untuk keluar dari log setelah bot berhasil terhubung).*

**Langkah 7: Klaim Superadmin**
Kirim pesan dari nomor pribadi Anda ke nomor bot dengan format:
`!klaim_superadmin`

**Langkah 8: Setup Keamanan & Port MikroTik (Wajib)**
Sebelum menautkan bot menggunakan perintah `setmikrotik`, Anda wajib mengatur hak akses dan port API di router Anda:
1. **Buka Port API:** Masuk ke Winbox, buka menu **IP > Services**. Pastikan layanan `api` (port 8728) atau `api-ssl` (port 8729) dalam keadaan **Enable** (aktif). *(Jika menggunakan `api-ssl`, pastikan Anda sudah men-generate sertifikat SSL di router).*
2. **Buat User Khusus:** Buat *User Group* baru di MikroTik (System > Users > Groups) khusus untuk bot dengan izin akses terbatas hanya untuk: `read`, `write`, `api`, dan `test`.
3. **Tambahkan User:** Buat *User* baru (System > Users) dan masukkan ke dalam grup yang baru dibuat tersebut. Hindari penggunaan akun *Full Admin*.

---

## 7. Panduan Uninstall / Hapus Vogenikbot

Panduan ini akan menghapus seluruh sistem Vogenikbot beserta datanya (file konfigurasi, database voucher, sesi WA) hingga bersih, **tanpa** menghapus Node.js atau program lain di STB Anda.

**Langkah 1: Hentikan Service Bot**
Matikan bot yang sedang berjalan agar file tidak terkunci saat dihapus:
```bash
sudo systemctl stop vogenikbot
sudo systemctl disable vogenikbot
```

**Langkah 2: Hapus Aplikasi**
Hapus paket instalasi Vogenikbot dari sistem Linux Anda:
```bash
sudo apt remove --purge vogenikbot -y
```

**Langkah 3: Hapus Sisa Data & Database (Penting!)**
Perintah ini sangat penting untuk menghapus seluruh file konfigurasi (`config.json`), database voucher (`vouchers.json`), dan folder sesi WhatsApp (`auth_info`) yang biasanya tertinggal. Langkah ini memastikan ketika Anda instal ulang nanti, semuanya benar-benar mulai dari nol:
```bash
sudo rm -rf /opt/vogenikbot
```

**Langkah 4: Segarkan Sistem**
Beri tahu sistem Linux bahwa service Vogenikbot sudah benar-benar dihapus:
```bash
sudo systemctl daemon-reload
```

**Langkah 5: Pembersihan di MikroTik (Wajib)**
Karena bot ini pernah membuat pengaturan otomatis di router Anda, pastikan untuk membersihkannya secara manual melalui **Winbox**:
1. Masuk ke **IP > Hotspot > User Profiles**: Hapus profil paket yang sebelumnya dibuat lewat bot.
2. Masuk ke **IP > Hotspot > Users**: Hapus sisa-sisa voucher pelanggan yang di-generate bot.
3. Masuk ke **System > Scheduler**: Hapus semua jadwal pembersihan otomatis (biasanya berawalan `CLEAN-`).
4. Masuk ke **System > Users**: Hapus *User* khusus bot yang sebelumnya Anda buat untuk menghubungkan sistem ini.

---

## Dukung Proyek Ini ☕

Jika bot ini bermanfaat dan Anda ingin mendukung pengembangannya, Anda bisa traktir saya kopi melalui link di bawah ini:

☕ [Traktir Kopi via Trakteer](https://trakteer.id/yoestration)

atau melalui QRIS di bawah ini: 
<br>
<p align="center">
<img width="250" height="250" alt="qris" src="https://github.com/user-attachments/assets/4c1d337e-00ba-47e8-9d7d-d6c4174e1516" />
</p>
