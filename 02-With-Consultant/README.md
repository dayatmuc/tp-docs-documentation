# TP Docs — With Consultant

Folder ini merupakan dokumentasi lengkap dan mandiri untuk TP Docs With Consultant. Seluruh kemampuan inti—autentikasi, organisasi, Local File, subscription, AI, document generation, keamanan, database, dan operasional platform—dituliskan penuh di dalam varian ini, kemudian diperluas dengan layanan Konsultan.

## Peta folder

| Folder | Maksud dan isi |
| --- | --- |
| [01-Requirements](./01-Requirements/) | [PRD](./01-Requirements/PRD.md) menetapkan tujuan, cakupan, aktor, serta aturan produk; dokumen fungsional dan nonfungsional merinci kemampuan sistem dan layanan Konsultan. |
| [02-Use-Case](./02-Use-Case/) | Aktivitas dan hak akses [User Induk](./02-Use-Case/Organization-Owner.md), [Karyawan](./02-Use-Case/Employee.md), [Konsultan](./02-Use-Case/Consultant.md), dan [Administrator](./02-Use-Case/Admin.md), berikut diagram. |
| [03-Business-Flow](./03-Business-Flow/) | Urutan kerja dan titik keputusan mulai dari autentikasi dan Local File hingga pembayaran jasa, assignment, review, komentar, revisi, dan delivery. Mulai dari [overview](./03-Business-Flow/00-Business-Flow-Overview.md). |
| [04-Database](./04-Database/) | [ERD](./04-Database/ERD.dbml), arti tabel dan kolom, serta pedoman query untuk data inti dan layanan Konsultan, termasuk order, assignment, komentar, dan working copy. |
| [05-Architecture](./05-Architecture/) | [Arsitektur sistem](./05-Architecture/System-Architecture.md), teknologi, kontrak API, AI, pembuatan dokumen, serta desain teknis konsultasi dan viewer komentar. |
| [06-Security-Audit](./06-Security-Audit/) | Materi dan alat bantu audit keamanan untuk autentikasi, isolasi data, akses Konsultan, integrasi, deployment, serta temuan. Folder ini bukan alur penggunaan pelanggan. |

## Referensi Utama

- [PRD](./01-Requirements/PRD.md)
- [Consultant Use Case](./02-Use-Case/Consultant.md)
- [Consultation Service Flow](./03-Business-Flow/08-Consultation-Service.md)
- [Consultant Workflow](./03-Business-Flow/09-Consultant-Workflow.md)
- [Consultation Architecture](./05-Architecture/Consultation-Architecture.md)
- [Document Review Architecture](./05-Architecture/Document-Review-Architecture.md)

Urutan baca yang disarankan: PRD → Use Case → Business Flow → Database dan Architecture.

