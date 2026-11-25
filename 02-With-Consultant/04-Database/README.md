# Database — TP Docs (With Consultant)

Folder ini memuat rancangan data PostgreSQL untuk varian TP Docs With Consultant.

| Dokumen | Fungsi |
| --- | --- |
| [ERD.dbml](./ERD.dbml) | Definisi tabel, enum, indeks, dan relasi yang dapat divisualisasikan melalui editor DBML |
| [Data-Dictionary.md](./Data-Dictionary.md) | Makna tabel, aggregate, constraint, tenant boundary, dan aturan retensi |
| [Query-Optimization.md](./Query-Optimization.md) | Pencegahan N+1, pola Prisma, indeks, pagination, execution plan, dan pengujian performa |

## Batas Dokumen

- ERD merupakan rancangan logis-fisik awal dan belum menggantikan migration Prisma/PostgreSQL.
- Constraint partial index, check constraint, trigger, serta Row Level Security ditambahkan melalui migration setelah proof of concept.
- Formula harga, upgrade, prorata, refund, dan retensi legal masih mengikuti keputusan bisnis yang ditandai pending pada PRD.
- File permanen disimpan di PostgreSQL: metadata pada `files` dan isi biner pada `file_blob_chunks` berukuran 1 MiB per chunk, maksimal 50 MB per file.
- Backup, PITR, restore, retensi, dan penghapusan PostgreSQL wajib mencakup metadata serta seluruh chunk file dan diverifikasi dengan checksum.
- Skema Konsultan mencatat seluruh payment attempt pada `consultation_request_payments`, status komentar pada `review_comment_status_history`, snapshot komentar pada `review_delivery_comments`, dan delivery Penyusunan Penuh pada `consultation_workspace_deliveries`. Constraint parsial serta aturan append-only dijelaskan pada Data Dictionary dan diterapkan pada migration production.

---

## Ekstensi Data Layanan Konsultan

- [ERD Extension](./ERD.dbml)
- [Data Dictionary](./Data-Dictionary.md)
- Skema terpadu: [ERD With Consultant](./ERD.dbml)
- Aturan query: [Query Optimization](./Query-Optimization.md)

`ERD.dbml` pada folder ini memvisualisasikan domain konsultasi dan stub tabel baseline yang direferensikan. Migration production menggabungkan model baseline dan extension dalam satu schema PostgreSQL/Prisma.
