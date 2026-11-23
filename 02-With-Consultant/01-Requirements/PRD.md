# PRODUCT REQUIREMENTS DOCUMENT (PRD)

## TP Docs — Transfer Pricing Documentation System (With Consultant)

## 1. Document Information

| Informasi          | Nilai                                                     |
| ------------------ | --------------------------------------------------------- |
| Nama proyek        | TP Docs — Transfer Pricing Documentation System           |
| Varian produk      | 02 — With Consultant                                            |
| Versi dokumen      | 1.0                                                       |
| Tanggal dokumen    | 19 November 2026                                          |
| Penulis            | Hidayat Nur Hakim                                         |
| Reviewer           | M Nur Kusumo Ferby Prihartoro dan Mohamad Budhi Kurniawan |
| Approver           | [Nama Approver]                                           |
| Status persetujuan | Draft — Belum Direview                                    |

Dokumen ini menjadi dokumen induk tingkat produk. Rincian kebutuhan dan use case dipelihara secara modular pada dokumen yang ditautkan dalam PRD ini.

## 2. Product Overview

### 2.1 Latar Belakang

Penyusunan dokumentasi transfer pricing, khususnya **Local File**, membutuhkan pengumpulan data perusahaan, transaksi afiliasi, analisis fungsi-aset-risiko (FAR), pemilihan metode transfer pricing, data pembanding, serta penyusunan dokumen yang konsisten. Proses manual berisiko menimbulkan duplikasi pekerjaan, ketidakkonsistenan data, kesalahan perhitungan, dan kebocoran informasi keuangan maupun perpajakan.

### 2.2 Permasalahan yang Diselesaikan

TP Docs dirancang untuk mengatasi permasalahan berikut:

- Data Local File tersebar pada banyak dokumen dan format.
- Proses penyusunan membutuhkan input berulang dan sulit dilacak.
- Kolaborasi tim berisiko menggunakan akun bersama atau hak akses yang terlalu luas.
- Analisis FAR, transaksi, metode TP, dan pembanding belum berada dalam satu alur terstruktur.
- Pembuatan dan pembaruan dokumen membutuhkan waktu serta pemeriksaan manual yang besar.
- Data perpajakan dan keuangan memerlukan kontrol akses, audit, serta perlindungan yang memadai.

### 2.3 Gambaran Umum Produk

TP Docs adalah aplikasi **Software as a Service (SaaS) multi-tenant berbasis web** untuk membantu perusahaan menyusun draft Local File secara mandiri. Sistem menyediakan pengisian data terstruktur, kolaborasi berbasis peran, pengolahan transaksi dan pembanding, analisis FAR, pemilihan metode TP, preview, serta generate dokumen.

TP Docs merupakan platform penyusunan Local File yang juga dapat menyediakan layanan review atau penyusunan oleh Konsultan yang ditugaskan. Sistem dan Konsultan tidak memberikan jaminan kepatuhan, tidak bertindak sebagai kuasa hukum atau wakil pajak, dan tidak mengirimkan dokumen langsung kepada Direktorat Jenderal Pajak.

## 3. Product Objectives

Tujuan pengembangan TP Docs adalah:

- Menstandarkan pengumpulan dan pengelolaan data Local File.
- Mengurangi pekerjaan manual, input berulang, dan ketidakkonsistenan data.
- Mengintegrasikan analisis transaksi afiliasi, FAR, metode TP, PLI, dan pembanding.
- Mendukung kolaborasi User Induk dan Karyawan tanpa berbagi akun.
- Menghasilkan draft Local File yang dapat ditinjau dan diekspor oleh pengguna.
- Menyediakan riwayat dokumen, duplikasi, pustaka pembanding, dan audit aktivitas.
- Mendukung model bisnis langganan, kuota dokumen, kursi pengguna, serta add-on.
- Menjaga kerahasiaan, integritas, dan ketersediaan data perusahaan.

Manfaat yang diharapkan adalah proses penyusunan yang lebih cepat, konsisten, dapat ditelusuri, aman, dan mudah dikelola oleh perusahaan.

## 4. Product Scope

### 4.1 In Scope

- Registrasi, verifikasi email, login, MFA, dan pemulihan akun.
- Profil pengguna, perusahaan, keanggotaan, undangan, dan penugasan dokumen.
- Paket langganan, pembayaran, refund, kuota dokumen, kursi, dan add-on.
- Pengisian profil perusahaan dan data kepemilikan serta manajemen.
- Pengisian gambaran industri, transaksi afiliasi, dan transaksi independen.
- Impor dan validasi data melalui Excel, TSV, PDF benchmarking, dan TP Catalyst.
- Analisis FAR, pemilihan metode TP, evaluasi metode, PLI, dan rentang kewajaran.
- Pengelolaan data pembanding dan pustaka pembanding.
- Preview, warning, generate, ekspor, history, dan duplikasi dokumen.
- Settings informasi pribadi.
- Portal Administrator untuk operasional, maintenance, keamanan, billing, backup, retensi, dan laporan agregat.
- Portal Konsultan, undangan akun, MFA, keahlian, kapasitas, assignment, dan ETA.
- Katalog jasa konsultasi, quotation Penyusunan Penuh, pembayaran per dokumen, dan refund konsultasi.
- Preview HTML, highlight, komentar, thread, ringkasan per bab, review ulang, working copy, dan change set.
- Isolasi tenant, audit log, backup, ekspor, retensi, dan penghapusan data.

### 4.2 Out of Scope

- Penyusunan Master File dan Country-by-Country Report.
- Representasi hukum, pendampingan pemeriksaan, keberatan, banding, atau sengketa pajak oleh Konsultan.
- Opini hukum, opini pajak, atau jaminan kepatuhan.
- Pelaporan atau pengiriman dokumen secara langsung kepada DJP.
- Pemeriksaan, keberatan, banding, dan sengketa pajak.
- Fungsi ERP, pembukuan, atau penyusunan laporan keuangan.
- Penyediaan database pembanding komersial atau berlisensi.
- Verifikasi mutlak terhadap kebenaran seluruh data pengguna.
- Tanda tangan elektronik dan pengesahan dokumen final.

### 4.3 Asumsi

- Versi awal berfokus pada Local File.
- Pengguna memiliki hak untuk menggunakan seluruh data dan file yang diunggah.
- Dokumen hasil sistem tetap berstatus draft sampai ditinjau pengguna.
- Satu akun menggunakan identitas pribadi dan tidak boleh dibagikan.
- User Induk, Administrator, dan Konsultan wajib menggunakan MFA.
- Integrasi email, pembayaran, malware scanning, dan AI menggunakan layanan eksternal; seluruh file permanen disimpan di PostgreSQL.
- Ketersediaan fitur mengikuti paket dan status langganan organisasi.

### 4.4 Batasan Sistem

- Sistem membantu penyusunan, tetapi keputusan metode TP dan interpretasi hasil tetap menjadi tanggung jawab pengguna.
- Sistem tidak bertindak sebagai konsultan otomatis atau wakil pajak; layanan manusia hanya dilakukan oleh Konsultan yang ditugaskan melalui order konsultasi yang sah.
- Kualitas keluaran bergantung pada kelengkapan dan kebenaran data masukan.
- Administrator tidak otomatis dapat membaca isi dokumen pelanggan.
- Karyawan hanya dapat mengakses organisasi dan dokumen sesuai keanggotaan serta penugasan.
- Paket Enterprise/Custom dapat bergantung pada kontrak dan kemampuan infrastruktur yang tersedia.

## 5. User Roles & Responsibilities

| Aktor             | Tanggung jawab utama                                                                                                                                                       |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **User Induk**    | Mendaftarkan perusahaan, mengelola profil perusahaan, langganan, pembayaran, kuota, anggota, penugasan, seluruh dokumen perusahaan, serta permintaan penghapusan data.     |
| **Karyawan**      | Menggunakan akun pribadi, menerima undangan perusahaan, melihat status akses langganan, dan mengerjakan dokumen yang ditugaskan tanpa mengelola anggota maupun pembayaran. |
| **Konsultan**     | Mengerjakan assignment berbayar yang ditugaskan, memberikan review dan komentar, melakukan review ulang, atau menyusun working copy tanpa melihat harga maupun dokumen lain. |
| **Administrator** | Memelihara platform, katalog jasa, akun Konsultan, assignment, quotation, billing, keamanan, integrasi, backup, retensi, insiden, dan laporan operasional.  |

Detail aktivitas aktor tersedia pada:

- [Use Case User Induk](../02-Use-Case/Organization-Owner.md)
- [Use Case Karyawan](../02-Use-Case/Employee.md)
- [Use Case Konsultan](../02-Use-Case/Consultant.md)
- [Use Case Administrator](../02-Use-Case/Admin.md)

## 6. Product Features Overview

| Modul                           | Ringkasan                                                                                                                                                 |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Authentication & Identity**   | Registrasi, verifikasi email, login, MFA, pemulihan password, sesi, dan profil pribadi.                                                                   |
| **Organization & Membership**   | Profil perusahaan, User Induk, Karyawan, link undangan berbasis email, registrasi penerima bila diperlukan, kursi, dan penugasan dokumen.                 |
| **Document Management**         | Membuat, menyimpan, membuka, memperbarui, menduplikasi, menelusuri history, dan mengelola status dokumen.                                                 |
| **Local File Wizard**           | Pengisian terstruktur untuk profil perusahaan, kepemilikan, manajemen, uraian proses bisnis, visual AI opsional, industri, transaksi, dan data pendukung. |
| **FAR Analysis**                | Pengelolaan fungsi, aset, risiko, pihak yang dianalisis, metode terpilih, indikator laba, dan evaluasi metode TP.                                         |
| **Benchmarking**                | Upload benchmarking, strategi pencarian, pemilihan PLI, pembanding final, dan rentang interkuartil.                                                       |
| **Document Generation**         | Preview ringkasan, warning di luar rentang, generate draft, serta ekspor dokumen sesuai paket.                                                            |
| **Subscription & Billing**      | Paket Free, Basic, Professional, Enterprise/Custom, pembayaran, refund, kuota dokumen, kursi, dan add-on.                                                 |
| **Consultation Services**       | Katalog jasa, order per document version, quotation, payment, assignment, SLA, revisi, dan refund konsultasi.                                             |
| **Document Review**             | Preview HTML, highlight, anchor, komentar, thread, ringkasan per bab, review ulang, dan delivery immutable.                                               |
| **Full Preparation Workspace**  | Working copy terpisah, change set, preview draft, persetujuan User Induk, dan promotion menjadi versi Local File baru.                                    |
| **Comparison Library**          | Penyajian data pembanding tersimpan dan rentang kuartil tanpa membuka file sumber yang tidak diizinkan.                                                   |
| **Administration & Operations** | Dashboard operasional, maintenance mode, pengaturan dasar, langganan, akun, Enterprise/Custom, keamanan, integrasi, backup, retensi, dan laporan agregat. |

### 6.1 Komposisi Local File

Local File merupakan dokumen pendukung penerapan prinsip kewajaran dan kelaziman usaha atas transaksi yang dipengaruhi hubungan istimewa. TP Docs menyusun draft Local File ke dalam lima kelompok informasi:

1. Identitas, kegiatan, dan strategi usaha Wajib Pajak.
2. Transaksi afiliasi dan transaksi independen.
3. Analisis FAR, kesebandingan, metode transfer pricing, dan penerapannya.
4. Informasi keuangan.
5. Peristiwa, kejadian, atau fakta nonkeuangan yang memengaruhi pembentukan harga atau tingkat laba.

Uraian proses bisnis merupakan bagian wajib dari penjelasan kegiatan dan aspek operasional perusahaan. Skema Proses Bisnis — Generated by AI hanya merupakan visual pendukung opsional. Local File tetap dapat diselesaikan tanpa skema AI selama uraian dan data wajib lainnya lengkap. Jika digunakan, hasil AI berstatus draft sampai ditinjau dan diterima User Induk.

Kategori visual `Input`, `Process`, `Control`, dan `Output` merupakan klasifikasi internal TP Docs dan bukan kategori resmi Local File. TP Docs tetap merupakan alat bantu penyusunan draft dan tidak menjamin kepatuhan perpajakan.

Dasar rujukan: [PMK 172 Tahun 2023 — Pasal 30 dan Lampiran E](https://www.jdih.kemenkeu.go.id/api/download/b0b74668-cf4d-456b-bb23-be5e1aeac2cb/2023pmkeuangan172.pdf).

## 7. Requirements Summary

### 7.1 Functional Requirements

Kebutuhan fungsional mencakup identitas dan autentikasi, organisasi dan anggota, langganan dan kuota, pengisian Local File, impor transaksi, analisis FAR, pembanding, preview dan generate dokumen, history, settings, layanan konsultasi, review dokumen, working copy, notifikasi, audit, ekspor, penghapusan data, serta operasional Administrator.

Detail kebutuhan fungsional tersedia pada [Functional Requirements](./Functional-Requirements.md).

### 7.2 Non-Functional Requirements

Kebutuhan nonfungsional mencakup performa, ketersediaan, backup, pemulihan, keamanan data, isolasi tenant, kontrol akses, enkripsi, keamanan file, audit log, retensi, lokasi data, respons insiden, kompatibilitas, dan aksesibilitas.

Detail kebutuhan nonfungsional tersedia pada [Non-Functional Requirements](./Non-Functional-Requirements.md).

### 7.3 Use Case

- [Use Case User Induk](../02-Use-Case/Organization-Owner.md)
- [Use Case Karyawan](../02-Use-Case/Employee.md)
- [Use Case Konsultan](../02-Use-Case/Consultant.md)
- [Use Case Administrator](../02-Use-Case/Admin.md)
- [Use Case Diagram](../02-Use-Case/Use-Case-Diagram.drawio)

## 8. Business Rules

### Aturan promosi

Diskon event bersifat opsional. Administrator dapat mengaktifkan promosi untuk paket, add-on, dan quotation; varian Konsultan juga mencakup jasa konsultasi. Tanpa event yang cocok, harga normal berlaku. Total pembayaran dibekukan dan refund mengacu pada jumlah yang dibayar.


1. Setiap dokumen, data, histori perubahan, dan hasil generate dimiliki oleh organisasi yang membuatnya.
2. User Induk dapat mengelola seluruh dokumen, anggota, langganan, dan data perusahaan dalam organisasinya.
3. Karyawan hanya dapat mengakses dokumen yang ditugaskan dan tidak dapat mengelola paket, pembayaran, atau anggota.
4. Administrator tidak boleh mengubah isi Local File, memilih metode TP, menugaskan dokumen, atau menyamar sebagai pengguna.
5. Administrator tidak dapat membaca atau mengubah isi Local File dan tidak memiliki akses dukungan langsung ke data dokumen pelanggan.
6. Free/Trial menyediakan satu dokumen selama masa akun dan satu pengguna.
7. Basic menyediakan lima dokumen dan satu User Induk untuk periode 12 bulan sejak paket diaktifkan.
8. Professional menyediakan sepuluh dokumen dan tiga pengguna total untuk periode 12 bulan sejak paket diaktifkan, yaitu satu User Induk dan maksimal dua Karyawan.
9. Kursi tambahan dan add-on dokumen hanya berlaku sampai akhir periode paket yang sedang berjalan, kecuali kontrak menyatakan lain.
10. Kuota dokumen berkurang setelah dokumen baru berhasil dibuat, bukan ketika pengguna membuka formulir.
11. Sistem menggunakan kuota paket sebelum menggunakan kuota add-on.
12. Akun Karyawan tetap menjadi akun pribadi; akses fitur berbayar mengikuti masa aktif langganan organisasi.
13. User Induk mengundang calon Karyawan melalui link unik yang terikat pada email tujuan; calon Karyawan yang belum memiliki akun wajib registrasi dan memverifikasi email sebelum menerima undangan.
14. Dokumen yang sudah dibuat tetap menjadi milik perusahaan ketika Karyawan dikeluarkan atau langganan berakhir.
15. Generate Local File hanya dapat dilakukan jika data wajib dan validasi minimum telah dipenuhi serta kuota dan paket mengizinkan.
16. Hasil generate berstatus draft dan wajib ditinjau oleh pengguna sebelum digunakan.
17. Data organisasi harus terisolasi dari organisasi lain pada UI, API, storage, cache, pencarian, dan background job.
18. Tindakan berisiko tinggi Administrator membutuhkan autentikasi ulang, alasan, tiket, audit log, dan persetujuan Administrator kedua.
19. Penghapusan data perusahaan hanya dapat diminta User Induk setelah autentikasi ulang dan mengikuti kebijakan retensi.

## 9. Success & Acceptance Criteria

### 9.1 Indikator Keberhasilan

- Pengguna dapat menyelesaikan alur penyusunan draft Local File melalui tahapan terstruktur.
- Data yang sama dapat digunakan kembali tanpa input berulang yang tidak diperlukan.
- Kolaborasi perusahaan berjalan menggunakan akun pribadi dan hak akses terkontrol.
- Dokumen dapat dibuat, ditinjau, diekspor, dilacak, dan diduplikasi sesuai paket.
- Penggunaan kuota, kursi, dan masa aktif langganan tercatat secara konsisten.
- Platform memenuhi target performa, ketersediaan, keamanan, backup, dan retensi yang ditetapkan.

### 9.2 Acceptance Criteria

Sistem dapat diterima apabila:

- Seluruh kebutuhan fungsional prioritas telah lulus pengujian.
- Seluruh use case User Induk, Karyawan, dan Administrator memiliki hasil yang sesuai hak aksesnya.
- Pengguna tidak dapat mengakses data organisasi lain melalui UI, API, URL file, cache, atau background job.
- Karyawan tidak dapat membuka dokumen yang tidak ditugaskan.
- Administrator tidak dapat membaca atau mengubah isi dokumen pelanggan.
- MFA wajib diterapkan kepada User Induk dan Administrator.
- Perhitungan kuota, kursi, add-on, dan masa aktif menghasilkan nilai yang benar.
- Generate dokumen hanya berlangsung setelah validasi dan tidak mengurangi kuota apabila proses pembuatan dokumen gagal.
- File berbahaya dikarantina dan tidak dapat diproses.
- Audit aktivitas sensitif tersedia dan tidak memuat password, token, NPWP lengkap, atau isi dokumen.
- Backup dapat dipulihkan sesuai target RPO dan RTO.
- Tautan internal menuju dokumen requirements dan use case dapat diakses.
- Tidak terdapat defect kritis terbuka pada saat persetujuan rilis.

## 10. Risks, Dependencies & Open Questions

### 10.1 Risks

| Risiko                                       | Dampak                                       | Mitigasi awal                                                            |
| -------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------ |
| Data pengguna tidak lengkap atau tidak benar | Draft dan analisis menjadi tidak akurat      | Validasi, warning, checklist, dan tanggung jawab peninjauan pengguna     |
| Akses lintas tenant                          | Kebocoran data rahasia perusahaan            | Tenant isolation, RBAC, automated test, audit, dan monitoring            |
| Kegagalan generate dokumen                   | Dokumen terlambat atau kuota salah           | Job queue, idempotency, status progres, retry, dan rekonsiliasi kuota    |
| Kegagalan integrasi eksternal                | Email, pembayaran, AI, atau upload terganggu | Retry terkontrol, circuit breaker, monitoring, dan fallback operasional  |
| Perubahan regulasi transfer pricing          | Struktur dokumen tidak lagi sesuai kebutuhan | Konfigurasi template, versioning, dan review berkala                     |
| Penyalahgunaan akses Administrator           | Akses data pelanggan tanpa kewenangan        | MFA, least privilege, dual approval, akses berbatas waktu, dan audit log |

### 10.2 Dependencies

- Layanan email transaksional.
- Payment gateway.
- PostgreSQL terkelola, termasuk penyimpanan file biner dan layanan backup.
- Malware scanner.
- Layanan AI untuk fitur yang memerlukannya.
- Infrastruktur cloud, database, cache, dan background job.
- Template dokumen Local File yang telah disetujui.
- Kebijakan paket, harga, kuota, retensi, dan lokasi data.
- Ketersediaan data pembanding yang sah dari pengguna atau sumber berlisensi.

## Persetujuan Dokumen

| Peran           | Nama                          | Status          |
| --------------- | ----------------------------- | --------------- |
| Penulis         | Hidayat Nur Hakim             | Draft           |
| Reviewer Produk | [Nama Reviewer Produk]        | Belum Direview  |
| Reviewer Tim TP | M Nur Kusumo Ferby Prihartoro | Belum Direview  |
| Reviewer Tim IT | Mohamad Budhi Kurniawan       | Belum Direview  |
| Approver        | [Nama Approver]               | Belum Disetujui |

---

## Ekstensi Produk dan Layanan Konsultan

### Tujuan Ekstensi

Varian ini menambahkan layanan Konsultan manusia pada seluruh kemampuan inti TP Docs. Konsultasi hanya dapat dimulai dari document version yang berhasil di-generate.

### Aktor Tambahan

**Konsultan** adalah role platform internal yang diundang Administrator, wajib menggunakan MFA, dan hanya dapat mengakses request berbayar yang ditugaskan kepadanya.

### Fitur Tambahan

- Katalog jasa konsultasi yang nama, harga, currency, SLA, revisi, dan statusnya dikelola Administrator.
- Review Konsultan TP dengan preview HTML, highlight, komentar, thread, ringkasan per bab, dan review ulang.
- Penyusunan Penuh menggunakan quotation, payment, assignment, working copy, change set, delivery, serta persetujuan User Induk.
- Harga dan SLA disalin sebagai snapshot order agar perubahan katalog tidak mengubah order lama.
- Review tambahan setelah kuota revisi habis menggunakan order dan pembayaran baru.
- Karyawan hanya dapat membeli layanan untuk dokumen yang ditugaskan dan hanya melihat transaksi konsultasi yang dibuatnya.
- User Induk dapat melihat seluruh request konsultasi organisasi.

### Batas Layanan

Konsultan tidak dapat melihat request yang belum dibayar, harga atau detail pembayaran, anggota organisasi, maupun dokumen di luar assignment. Layanan tidak mencakup representasi hukum, pemeriksaan, keberatan, banding, sengketa pajak, atau jaminan kepatuhan.
