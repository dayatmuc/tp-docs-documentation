# TP Docs — By System

Varian ini mendokumentasikan TP Docs sebagai SaaS untuk menyusun draft Local File secara mandiri. User Induk dan Karyawan mengisi serta mengelola dokumen sesuai hak aksesnya; Administrator mengelola operasional platform. Tidak ada layanan review atau penyusunan oleh Konsultan dalam varian ini.

## Peta folder

| Folder | Maksud dan isi |
| --- | --- |
| [01-Requirements](./01-Requirements/) | Menjelaskan kebutuhan produk: [PRD](./01-Requirements/PRD.md) berisi tujuan, cakupan, aktor, dan aturan bisnis; dokumen fungsional dan nonfungsional merinci kemampuan serta kualitas sistem. |
| [02-Use-Case](./02-Use-Case/) | Menjelaskan aktivitas dan batas kewenangan [User Induk](./02-Use-Case/Organization-Owner.md), [Karyawan](./02-Use-Case/Employee.md), dan [Administrator](./02-Use-Case/Admin.md). |
| [03-Business-Flow](./03-Business-Flow/) | Menunjukkan urutan kerja dan keputusan dari autentikasi, pengisian Local File, AI, pembuatan dokumen, keanggotaan, langganan, hingga operasional Admin. Mulai dari [overview](./03-Business-Flow/00-Business-Flow-Overview.md). |
| [04-Database](./04-Database/) | Mendefinisikan tabel dan relasi pada [ERD](./04-Database/ERD.dbml), arti data pada [Data Dictionary](./04-Database/Data-Dictionary.md), serta pedoman query dan pencegahan N+1. |
| [05-Architecture](./05-Architecture/) | Menjelaskan susunan komponen, pilihan teknologi, kontrak API, pemrosesan AI, dan cara data Local File dirender menjadi dokumen. Mulai dari [System Architecture](./05-Architecture/System-Architecture.md). |
| [06-Security-Audit](./06-Security-Audit/) | Berisi materi dan alat bantu audit keamanan untuk menilai autentikasi, isolasi data, integrasi, deployment, serta temuan. Folder ini bukan alur penggunaan pelanggan. |

Urutan baca yang disarankan: PRD → Use Case → Business Flow → Database dan Architecture.
