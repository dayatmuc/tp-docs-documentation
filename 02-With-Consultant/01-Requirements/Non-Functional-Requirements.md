# Non-Functional Requirements — TP Docs (With Consultant)

Dokumen ini memuat kebutuhan nonfungsional, target layanan, keamanan, retensi, dan kompatibilitas TP Docs versi **With Consultant**. Ringkasan produk tersedia pada [PRD](./PRD.md).

## 1. Kebutuhan Nonfungsional

### 1.1 Performa dan Ketersediaan

- Ketersediaan layanan minimal 99,5% per bulan di luar pemeliharaan terjadwal.
- Sebanyak 95% permintaan halaman biasa harus selesai maksimal 3 detik pada kondisi operasional normal.
- Proses generate dokumen harus menampilkan progres dan ditargetkan selesai maksimal 5 menit untuk dokumen dalam batas yang didukung.
- Backup dilakukan minimal satu kali setiap hari.
- Target kehilangan data maksimal 24 jam (`RPO 24 jam`).
- Target pemulihan layanan maksimal 8 jam (`RTO 8 jam`).
- Pemulihan backup diuji minimal setiap tiga bulan.

### 1.2 Keamanan Data

**Integritas promosi.** Integritas diskon: perhitungan dan kelayakan event dilakukan di NestJS dengan waktu Asia/Jakarta, transaksi atomik untuk batas penggunaan, idempotensi webhook, dan audit setiap perubahan Admin. Harga dari browser tidak dipercaya; invoice, snapshot, histori, dan refund harus cocok dengan jumlah yang dibayar.


- Data profil, NPWP, transaksi, keuangan, struktur kepemilikan, FAR, pembanding, dan dokumen diklasifikasikan sebagai **Data Rahasia Perusahaan**.
- Seluruh komunikasi menggunakan TLS 1.2 atau versi yang lebih baru.
- Database PostgreSQL—termasuk metadata dan chunk file biner—serta backup dienkripsi saat tersimpan menggunakan AES-256 atau mekanisme setara dari penyedia cloud.
- Kunci enkripsi disimpan terpisah dari data dan memiliki mekanisme rotasi.
- Password disimpan menggunakan Argon2id, bcrypt, atau algoritma hashing adaptif yang setara dan tidak boleh disimpan dalam bentuk asli.
- MFA wajib bagi User Induk dan Administrator serta bersifat opsional bagi Karyawan pada versi awal.
- Akses menerapkan prinsip least privilege dan role-based access control.
- User Induk hanya dapat mengakses organisasinya dan Karyawan hanya dapat mengakses dokumen yang ditugaskan.
- Administrator tidak otomatis dapat membaca isi dokumen perusahaan.
- Akses dukungan terhadap data pelanggan memerlukan persetujuan, alasan, batas waktu, dan audit log.
- Isolasi tenant diterapkan pada tabel metadata file, chunk biner, cache, pencarian, queue, dan proses background.
- File dibatasi maksimal 50 MB dan disimpan sebagai chunk `bytea` 1 MiB; daftar dokumen, dashboard, dan assignment Konsultan tidak boleh mengambil isi chunk.
- Upload dan download melewati NestJS dengan rate limit, concurrency limit, checksum SHA-256, serta validasi tenant, assignment, dan hak akses.
- Backup dan restore harus mencakup seluruh chunk dan membuktikan kesamaan checksum file.
- File hanya dapat diakses melalui URL bertanda tangan dengan masa berlaku singkat.
- Upload divalidasi berdasarkan tipe, ukuran, dan ekstensi serta dipindai terhadap malware.
- File berbahaya dikarantina dan tidak dapat diproses.
- Secret, API key, token email, dan kredensial pembayaran disimpan pada secret manager, bukan source code.
- Log tidak boleh menyimpan password, token autentikasi, isi lengkap dokumen, NPWP lengkap, atau data finansial mentah.
- Login, perubahan anggota dan hak akses, upload, download, generate, pembayaran, ekspor, serta penghapusan dicatat dalam audit log.
- Audit log dilindungi dari perubahan dan disimpan minimal satu tahun.
- Sistem menerapkan rate limiting, perlindungan brute force, dan penguncian sementara setelah kegagalan login berulang.
- Sesi tidak aktif berakhir otomatis setelah 30 menit.
- Perubahan email dan tindakan berisiko tinggi memerlukan autentikasi ulang.
- Kerentanan kritis ditangani maksimal 7 hari dan kerentanan tinggi maksimal 30 hari.
- Pemindaian dependensi dan pengujian keamanan dilakukan pada setiap rilis dan secara berkala.
- Pengelolaan data diselaraskan dengan prinsip UU Perlindungan Data Pribadi tanpa mengklaim sertifikasi yang belum dimiliki.

### 1.3 Retensi dan Penghapusan Data

- Setelah langganan berakhir, data perusahaan dibekukan dan tersedia untuk pemulihan selama 30 hari.
- Setelah 30 hari, data dihapus dari sistem produksi melalui proses terjadwal.
- Salinan pada backup berakhir mengikuti siklus backup maksimal 90 hari setelah penghapusan produksi.
- Permintaan penghapusan hanya dapat dilakukan oleh User Induk setelah autentikasi ulang dan harus dicatat.
- Penghapusan akun Karyawan tidak menghapus data atau histori perusahaan.
- Sebelum penghapusan permanen, User Induk dapat mengekspor dokumen dan data yang didukung.
- Bukti penghapusan dicatat dalam audit log tanpa menyimpan kembali isi data yang telah dihapus.

### 1.4 Lokasi Data

- Lokasi penyimpanan data mengikuti paket langganan.
- Paket standar menggunakan region Asia Pasifik yang ditentukan penyedia layanan.
- Enterprise/Custom dapat meminta region Indonesia atau lokasi lain yang disepakati apabila tersedia.
- Data produksi, backup, dan replikasi mengikuti kebijakan lokasi paket yang sama.
- Pemindahan lokasi data harus disetujui User Induk dan dicatat.

### 1.5 Insiden Keamanan

- Sistem memonitor aktivitas tidak normal, kegagalan autentikasi, akses lintas tenant, malware, dan kebocoran secret.
- Insiden keamanan dicatat sebagai tiket dengan tingkat keparahan.
- Akses atau kredensial yang terdampak dapat segera dicabut.
- Pelanggan terdampak diberi notifikasi maksimal 72 jam setelah insiden terkonfirmasi atau mengikuti ketentuan hukum yang lebih ketat.
- Notifikasi menjelaskan jenis insiden, data terdampak, tindakan mitigasi, dan langkah yang perlu dilakukan pelanggan.

### 1.6 Kompatibilitas dan Aksesibilitas

- Sistem mendukung dua versi terbaru Chrome, Edge, dan Firefox.
- Fitur utama dapat digunakan pada desktop dan tablet, sedangkan penyusunan kompleks dioptimalkan untuk desktop.
- Form memiliki label, pesan validasi, dukungan navigasi keyboard, dan kontras yang memadai.

---

## Ekstensi Kebutuhan Nonfungsional Konsultan

### 1. Security dan Privacy

- MFA wajib bagi Konsultan.
- Konsultan bukan membership organisasi dan hanya memperoleh scoped access dari assignment aktif.
- Viewer, download, comment, reply, delivery, dan working-copy edit memvalidasi tenant, request, assignment, payment, serta status akses.
- Admin tidak otomatis dapat membaca isi dokumen.
- Preview dan file menggunakan URL privat berumur pendek.
- Selected text, komentar, dan draft diklasifikasikan Data Rahasia Perusahaan.
- Harga, provider credential, token invitation, dan service token tidak dikirim ke browser kecuali nilai display yang sah.
- Semua akses dokumen Konsultan dicatat dengan user, request, document version, waktu, dan tujuan.

### 2. Integrity dan Concurrency

- Document version yang direview immutable.
- Comment anchor menyimpan selected-text hash untuk mendeteksi perubahan.
- Thread dan working copy menggunakan optimistic concurrency version.
- Order menyimpan snapshot service version dan tidak bergantung pada katalog aktif setelah dibuat.
- Payment, webhook, assignment, delivery, promotion, serta refund wajib idempotent.

### 3. Performance

- Daftar assignment menggunakan pagination dan projection; tidak boleh mengalami N+1.
- Preview halaman awal ditargetkan tampil maksimal 3 detik untuk dokumen normal setelah aset tersedia.
- Comment autosave memberikan respons maksimal 2 detik pada persentil ke-95.
- Pembuatan preview dapat diproses asynchronous dan menampilkan progress.
- Highlight tidak melakukan query per block atau per komentar.

### 4. Availability dan Recovery

- Kegagalan viewer tidak menghapus document version atau komentar.
- Delivery dapat diulang secara idempotent.
- Working copy memiliki autosave, version, dan recovery point.
- Notifikasi gagal dikirim ulang melalui worker tanpa menggandakan thread atau delivery.

### 5. Audit dan Retention

- Audit meliputi perubahan katalog, quotation, payment, refund, assignment, ETA, viewer access, download, komentar, delivery, dan promotion.
- Comment history dan delivery yang telah diterbitkan immutable; koreksi membuat revision baru.
- Retensi mengikuti organisasi dan kewajiban legal yang ditetapkan pada NFR baseline.
- Log tidak memuat isi dokumen, selected text penuh, komentar penuh, NPWP, atau data finansial mentah.

### 6. Compatibility dan Accessibility

- Viewer mendukung dua versi terbaru Chrome, Edge, dan Firefox.
- Highlight, comment, reply, serta navigation bab dapat digunakan dengan keyboard.
- Warna highlight memiliki kontras memadai dan bukan satu-satunya penanda status.
- Penyusunan Penuh dioptimalkan untuk desktop.

### 7. Acceptance Criteria

- Percobaan akses request tenant lain menghasilkan `403` dan audit event aman.
- Konsultan tanpa MFA tidak memperoleh akses penuh.
- Query assignment tetap berjumlah konstan saat jumlah request bertambah.
- Perubahan katalog tidak mengubah snapshot order.
- Concurrent edit pada thread atau working copy menghasilkan version conflict, bukan silent overwrite.
- Highlight dapat ditemukan kembali pada document version asal.
