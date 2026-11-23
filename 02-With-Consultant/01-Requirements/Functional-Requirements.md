# Functional Requirements — TP Docs (With Consultant)

Dokumen ini memuat kebutuhan fungsional terperinci untuk TP Docs versi **With Consultant**. Ringkasan dan konteks produk tersedia pada [PRD](./PRD.md), sedangkan alur setiap aktor tersedia pada [Use Case User Induk](../02-Use-Case/Organization-Owner.md), [Use Case Karyawan](../02-Use-Case/Employee.md), [Use Case Konsultan](../02-Use-Case/Consultant.md), dan [Use Case Administrator](../02-Use-Case/Admin.md).

## 1. Kebutuhan Fungsional

| ID    | Kebutuhan Fungsional                                                                                                                                                                                       | Referensi Use Case         |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------- |
| FR-01 | Sistem menyediakan registrasi, autentikasi, verifikasi email, MFA, pemulihan password, dan pengelolaan profil                                                                                              | UI-01, UI-02, UI-03, UI-09 |
| FR-02 | Sistem mengelola organisasi, keanggotaan, link undangan berbasis email, registrasi penerima bila belum memiliki akun, kursi, dan penugasan dokumen                                                         | UI-08, UK-01, UK-02        |
| FR-03 | Sistem mengelola paket, periode 12 bulan sejak aktivasi, pembayaran, refund, kuota pengguna, kuota dokumen, dan add-on                                                                                     | UI-08, UA-05               |
| FR-04 | Sistem menyediakan pengisian data perusahaan dan konten Local File secara terstruktur, termasuk uraian proses bisnis wajib, daftar langkah opsional, serta visual Skema AI opsional                     | UI-04.01 s/d UI-04.07      |
| FR-05 | Sistem mendukung impor, validasi, pengolahan, dan summary transaksi                                                                                                                                        | UI-04.08                   |
| FR-06 | Sistem mendukung analisis FAR, pemilihan metode, dan evaluasi metode TP                                                                                                                                    | UI-04.09                   |
| FR-07 | Sistem mengelola data pembanding, PLI, pembanding final, dan rentang kewajaran                                                                                                                             | UI-05, UI-07               |
| FR-08 | Sistem menyediakan preview, warning, dan generate draft Local File                                                                                                                                         | UI-04.11                   |
| FR-09 | Sistem menyediakan history, duplikasi dokumen, pustaka pembanding, dan Settings                                                                                                                            | UI-06, UI-07, UI-09        |
| FR-10 | Sistem mengirim notifikasi dan mencatat aktivitas penting, ekspor, serta penghapusan data                                                                                                                  | UI-01, UI-08, UI-09, UK-01 |
| FR-11 | Sistem menyediakan portal dan dashboard operasional, maintenance mode, pengaturan dasar, billing, pengelolaan akun, pengajuan Enterprise/Custom, keamanan, integrasi, backup, dan audit bagi Administrator | UA-01 s/d UA-09            |

---

## Ekstensi Kebutuhan Fungsional Konsultan

Kebutuhan inti TP Docs telah dijelaskan pada bagian sebelumnya. Tabel berikut merupakan kebutuhan tambahan khusus layanan Konsultan.

| ID | Kebutuhan | Referensi |
| --- | --- | --- |
| FR-C01 | Sistem mengundang Konsultan melalui email aman, pembuatan password mandiri, dan MFA wajib | UA, UCN-01 |
| FR-C02 | Admin mengelola profil, keahlian, kapasitas, status, dan ketersediaan Konsultan | UA, UCN-12 |
| FR-C03 | Admin mengelola katalog, model harga, currency, SLA, revisi, version, dan aktivasi layanan | UA |
| FR-C04 | Sistem hanya menampilkan harga dari service version aktif atau quotation | UI, UK, UA |
| FR-C05 | Sistem membuat request hanya dari document version `SUCCEEDED` | UI, UK |
| FR-C06 | Karyawan hanya dapat mengajukan dokumen yang ditugaskan dan hanya melihat payment yang dibuatnya | UK |
| FR-C07 | Sistem menyimpan snapshot harga, currency, SLA, revisi, dan service version pada order | UI, UK |
| FR-C08 | Sistem memproses payment konsultasi melalui provider, webhook terverifikasi, dan idempotency | UI, UK, UA |
| FR-C09 | Admin membuat quotation Penyusunan Penuh dan pengguna menerima atau menolaknya | UI, UK, UA |
| FR-C10 | Admin melakukan assignment/reassignment manual setelah payment berhasil | UA, UCN-02–UCN-04 |
| FR-C11 | Konsultan hanya melihat request paid yang ditugaskan | UCN-02 |
| FR-C12 | Sistem menghitung ETA, mendukung konfirmasi, pause, dan perubahan terotorisasi | UA, UCN-04 |
| FR-C13 | Viewer menampilkan preview semantik dan mendukung text highlight | UCN-05–UCN-06 |
| FR-C14 | Sistem menyimpan anchor, comment thread, replies, category, severity, dan status | UCN-06–UCN-07 |
| FR-C15 | Sistem membentuk summary komentar per bab secara deterministik | UCN-08 |
| FR-C16 | Pengguna memperbaiki Local File, generate versi baru, dan resubmit | UI, UK, UCN-09 |
| FR-C17 | Sistem membatasi review ulang berdasarkan snapshot revisi order | UI, UK, UCN-09 |
| FR-C18 | Review tambahan membuat order dan payment baru | UI, UK |
| FR-C19 | Penyusunan Penuh menggunakan working copy terpisah dan change set | UCN-10–UCN-11 |
| FR-C20 | Promotion working copy memerlukan approval User Induk dan menghasilkan versi baru | UI, UCN-11 |
| FR-C21 | Sistem mengirim notifikasi status, assignment, ETA, delivery, dan resubmission | Semua aktor |
| FR-C22 | Sistem mengaudit katalog, quotation, payment, assignment, akses viewer, komentar, delivery, dan promotion | Semua aktor |

### Acceptance Rules

- Request gagal jika document version tidak tersedia atau tidak dimiliki tenant.
- Layanan `FIXED` wajib memiliki `displayPrice > 0`.
- Layanan `STARTING_FROM` wajib memiliki `minimumPrice > 0`.
- Layanan `QUOTATION` tidak dapat checkout sebelum quotation diterima.
- Assignment gagal apabila payment belum `PAID`.
- Delivery gagal jika belum ada comment/draft yang dipublikasikan.
- Resubmission gagal jika revisi order habis.
- Akses Konsultan berakhir setelah completion, revoke, atau reassignment.
| FR-12 | Admin mengelola promosi opsional; User Induk melihat harga normal atau promo, termasuk untuk jasa konsultasi. Sistem memilih satu potongan terbesar yang memenuhi syarat, membekukan harga saat transaksi, dan mempertahankan harga transaksi lama. | UI-08.1, UI-08.6, UA-05.6 |
