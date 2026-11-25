# Functional Requirements — TP Docs (By System)

Dokumen ini memuat kebutuhan fungsional terperinci untuk TP Docs versi **By System**. Ringkasan dan konteks produk tersedia pada [PRD](./PRD.md), sedangkan alur setiap aktor tersedia pada [Use Case User Induk](../02-Use-Case/Organization-Owner.md), [Use Case Karyawan](../02-Use-Case/Employee.md), dan [Use Case Administrator](../02-Use-Case/Admin.md).

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
| FR-12 | Admin mengelola promosi opsional; User Induk melihat harga normal atau promo. Sistem memilih satu potongan terbesar yang memenuhi syarat, membekukan harga saat transaksi, dan mempertahankan harga transaksi lama. | UI-08.1, UI-08.6, UA-05.6 |
| FR-13 | User Induk dan Karyawan yang ditugaskan mengisi bagian Local File dalam urutan bebas pada satu draft organisasi. Sistem mencatat pengubah dan versi bagian serta menolak penyimpanan yang akan menimpa perubahan lebih baru. | UI-04, UI-08.5, UK-03 |
