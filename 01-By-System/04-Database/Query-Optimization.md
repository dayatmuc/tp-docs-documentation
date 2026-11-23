# Query Optimization dan Pencegahan N+1 — TP Docs

## 1. Tujuan

Dokumen ini menetapkan aturan query PostgreSQL dan Prisma untuk TP Docs. Tujuannya bukan “menggunakan N+1”, tetapi mendeteksi dan mencegah pola satu query induk diikuti satu query tambahan untuk setiap baris hasil.

Referensi desain tabel tersedia pada [ERD.dbml](./ERD.dbml) dan [Data-Dictionary.md](./Data-Dictionary.md).

## 2. Definisi N+1 pada TP Docs

N+1 terjadi ketika aplikasi mengambil daftar sebanyak `N` baris, lalu menjalankan query relasi di dalam loop untuk setiap baris.

Contoh yang dilarang:

```ts
const localFiles = await prisma.localFile.findMany({
  where: { organizationId },
});

for (const localFile of localFiles) {
  localFile.assignments = await prisma.documentAssignment.findMany({
    where: { localFileId: localFile.id },
  });
}
```

Jika daftar memuat 100 dokumen, pola tersebut menjalankan satu query daftar dan 100 query assignment.

## 3. Strategi Pencegahan N+1

### 3.1 Nested read dengan field terpilih

Gunakan nested `select` atau `include` untuk relasi yang memang dibutuhkan oleh response.

```ts
const localFiles = await prisma.localFile.findMany({
  relationLoadStrategy: 'join',
  where: {
    organizationId,
    archivedAt: null,
  },
  select: {
    id: true,
    title: true,
    taxYear: true,
    status: true,
    updatedAt: true,
    assignments: {
      where: { revokedAt: null },
      select: {
        id: true,
        membershipId: true,
        assignedAt: true,
      },
    },
  },
  orderBy: [{ updatedAt: 'desc' }, { id: 'desc' }],
  take: 25,
});
```

`relationLoadStrategy: 'join'` digunakan setelah versi Prisma yang dipilih mendukungnya dan telah diuji terhadap pola data TP Docs. Jangan menggunakan `include: { relation: true }` tanpa batas pada relasi besar.

### 3.2 Batch menggunakan `IN`

Gunakan batch query apabila satu join besar menghasilkan row explosion atau relasi membutuhkan pengolahan terpisah.

```ts
const localFiles = await prisma.localFile.findMany({
  where: { organizationId },
  select: { id: true, title: true, status: true },
  take: 25,
});

const ids = localFiles.map((item) => item.id);

const assignments = await prisma.documentAssignment.findMany({
  where: {
    organizationId,
    localFileId: { in: ids },
    revokedAt: null,
  },
  select: {
    id: true,
    localFileId: true,
    membershipId: true,
  },
});
```

Hasil dikelompokkan di memory berdasarkan `localFileId`. Pola ini menghasilkan jumlah query tetap, bukan bertambah mengikuti jumlah dokumen.

### 3.3 DataLoader hanya untuk resolver berulang

Apabila pada masa depan digunakan GraphQL atau resolver yang memanggil relasi berulang, gunakan DataLoader per request. REST service biasa tetap mengutamakan nested read, join terukur, atau batch `IN`.

### 3.4 Dilarang melakukan I/O database dalam loop

Code review harus menolak pola berikut kecuali terdapat alasan transaction yang terdokumentasi:

- `for`, `forEach`, atau `map` yang memanggil Prisma per item.
- `Promise.all(items.map(() => prisma...))` karena tetap menghasilkan N query walaupun paralel.
- Getter repository yang diam-diam melakukan lazy loading.
- Template document generator yang melakukan query untuk setiap placeholder atau baris tabel.

Document generation harus membaca snapshot lengkap yang telah dibentuk mapper, bukan melakukan query saat Docxtemplater melakukan render.

## 4. Query Plan per Endpoint Utama

| Kebutuhan | Pola query | Target jumlah query |
| --- | --- | --- |
| Daftar Local File User Induk | Filter tenant, cursor pagination, assignment count terpilih | 1–2 |
| Daftar tugas Karyawan | Mulai dari `document_assignments`, join Local File dan membership aktif | 1 |
| Detail anggota organisasi | Membership + user field minimum + assignment count batch | 1–2 |
| Dashboard Administrator | Query agregat per domain atau read model terjadwal | Jumlah tetap, tidak berdasarkan jumlah organisasi |
| Detail Local File editor | Query per bounded aggregate/section, bukan seluruh dokumen sekaligus | Jumlah tetap per section |
| Review and Generate | Beberapa batch query per aggregate lalu mapper membentuk snapshot | Jumlah tetap dan diukur |
| Histori pembayaran/refund | Filter tenant, cursor pagination, join payment/refund terpilih | 1–2 |
| Library pembanding | Filter tenant, projection field minimum, cursor pagination | 1 |

## 5. Projection: Hindari `SELECT *`

Setiap repository wajib memilih field sesuai DTO atau kebutuhan domain.

```ts
const payment = await prisma.payment.findFirst({
  where: {
    id: paymentId,
    organizationId,
  },
  select: {
    id: true,
    status: true,
    purpose: true,
    currency: true,
    amountMinor: true,
    paidAt: true,
  },
});
```

Manfaat projection:

- Mengurangi data yang dibaca dan dikirim.
- Menghindari kebocoran kolom sensitif.
- Mengurangi memory backend.
- Membuat kontrak response lebih stabil.

## 6. Pagination dan Pembatasan Baris

Endpoint daftar tidak boleh mengembalikan seluruh tabel.

- Gunakan cursor pagination berbasis `(created_at, id)` atau `(updated_at, id)` untuk histori besar.
- Gunakan `take` dengan maksimum yang ditetapkan API.
- Offset pagination hanya untuk dataset kecil atau halaman Administrator yang volumenya terbatas.
- Sorting wajib deterministik dengan `id` sebagai tie breaker.
- Export besar dilakukan melalui background job, bukan endpoint daftar tanpa batas.

Contoh indeks untuk pagination Local File:

```sql
CREATE INDEX idx_local_files_org_updated
ON local_files (organization_id, updated_at DESC, id DESC)
WHERE archived_at IS NULL;
```

## 7. Strategi Indeks

### Query Diskon Event

Ambil kandidat promo dalam satu query dengan indeks `(status, starts_at, ends_at)` dan target `(product_type, product_id)`; hitung potongan di service berdasarkan harga server dan pilih satu nilai terbesar (tie-break ID terkecil). Jangan melakukan query per kartu paket atau per item quotation. Ringkasan penggunaan dihitung agregat dari `discount_redemptions`, sedangkan reservasi checkout memiliki TTL dan locking atomik agar batas penggunaan tidak terlewati.


### 7.1 Prinsip

- Indeks mengikuti pola `WHERE`, `JOIN`, dan `ORDER BY` yang nyata.
- Foreign key yang sering digunakan untuk join diberi indeks.
- Pada tabel tenant, `organization_id` umumnya menjadi kolom awal indeks.
- Gunakan composite index dengan urutan equality filter, range, lalu sorting.
- Gunakan partial index untuk record aktif atau belum diproses.
- Jangan membuat indeks untuk setiap kolom karena menambah biaya write dan storage.
- Evaluasi indeks yang duplikat atau tidak digunakan sebelum production.

### 7.2 Indeks akses utama

| Tabel | Indeks | Kebutuhan |
| --- | --- | --- |
| `local_files` | `(organization_id, status, updated_at)` | Daftar dan filter dokumen tenant |
| `document_assignments` | `(membership_id, revoked_at, assigned_at)` | Daftar tugas Karyawan |
| `memberships` | `(organization_id, status, role)` | Anggota aktif dan pengecekan kursi |
| `subscriptions` | `(organization_id, status, end_at)` | Entitlement aktif |
| `payments` | `(organization_id, status, created_at)` | Histori dan monitoring pembayaran |
| `quota_ledger` | `(subscription_id, quota_type, occurred_at)` | Perhitungan dan audit kuota |
| `ai_generations` | `(status, created_at)` | Worker dan monitoring generation |
| `document_generations` | `(status, created_at)` | Worker document generation |
| `audit_logs` | `(organization_id, occurred_at)` | Audit per tenant dan rentang waktu |

### 7.3 Full-text search

Jangan menambahkan full-text index sebelum kebutuhan pencarian bebas ditetapkan. Pencarian awal menggunakan kolom terstruktur seperti nama dokumen, tahun pajak, status, dan nama perusahaan. Jika pencarian teks bebas diperlukan, gunakan PostgreSQL `tsvector`/GIN berdasarkan benchmark dan klasifikasi data.

## 8. Join yang Efisien

- Mulai dari tabel yang paling selektif, misalnya assignment aktif untuk daftar dokumen Karyawan.
- Selalu join menggunakan primary/foreign key atau kolom yang terindeks.
- Gunakan `INNER JOIN` ketika relasi wajib dan `LEFT JOIN` hanya ketika data relasi memang opsional.
- Hindari join seluruh aggregate Local File sekaligus karena beberapa relasi one-to-many dapat menghasilkan perkalian baris.
- Untuk detail Local File, ambil section yang sedang dibuka atau gunakan beberapa batch query dengan jumlah tetap.
- Gunakan CTE untuk membuat query agregasi kompleks lebih mudah dibaca; jangan menganggap CTE otomatis lebih cepat tanpa `EXPLAIN`.

Contoh daftar tugas Karyawan:

```sql
SELECT
  lf.id,
  lf.title,
  lf.tax_year,
  lf.status,
  lf.updated_at
FROM document_assignments da
JOIN memberships m
  ON m.id = da.membership_id
 AND m.organization_id = da.organization_id
JOIN local_files lf
  ON lf.id = da.local_file_id
 AND lf.organization_id = da.organization_id
WHERE da.membership_id = $1
  AND da.organization_id = $2
  AND da.revoked_at IS NULL
  AND m.status = 'ACTIVE'
ORDER BY lf.updated_at DESC, lf.id DESC
LIMIT $3;
```

## 9. Filter Sargable

Hindari fungsi pada kolom indeks di dalam `WHERE`.

Tidak disarankan:

```sql
WHERE EXTRACT(YEAR FROM created_at) = 2026;
```

Disarankan:

```sql
WHERE created_at >= TIMESTAMPTZ '2026-01-01 00:00:00+00'
  AND created_at <  TIMESTAMPTZ '2027-01-01 00:00:00+00';
```

Aturan lain:

- Gunakan equality untuk enum/status.
- Gunakan rentang eksplisit untuk tanggal.
- Hindari `LOWER(column)` pada filter berfrekuensi tinggi kecuali disediakan functional index yang sesuai.
- Normalisasi email sebelum penyimpanan agar login tidak membutuhkan fungsi pada kolom.

## 10. `EXISTS`, Subquery, CTE, dan Set Operation

- Gunakan `EXISTS` ketika hanya memeriksa keberadaan relasi.
- Hindari correlated subquery yang dijalankan per baris pada dataset besar.
- Gunakan join atau batch jika data relasi juga harus dikembalikan.
- Gunakan `UNION ALL` apabila penghapusan duplikat tidak diperlukan.
- Hindari `DISTINCT` untuk menutupi join yang salah; perbaiki cardinality atau constraint.

Contoh pemeriksaan apakah organisasi mempunyai subscription aktif:

```sql
SELECT EXISTS (
  SELECT 1
  FROM subscriptions s
  WHERE s.organization_id = $1
    AND s.status = 'ACTIVE'
    AND s.end_at > now()
);
```

## 11. Agregasi dan Materialized View

Agregasi transaksi, kuota, dashboard, dan laporan harus menggunakan SQL set-based, bukan loop aplikasi.

- `transaction_summaries` dapat menjadi read model yang diregenerasi setelah perubahan transaksi.
- Dashboard Administrator menggunakan query agregat terpisah atau read model terjadwal ketika volume meningkat.
- Materialized view baru ditambahkan setelah slow-query evidence menunjukkan agregasi berulang menjadi bottleneck.
- Refresh materialized view harus mempunyai jadwal, freshness target, dan monitoring.
- Nilai yang memengaruhi entitlement atau pembayaran tidak boleh bergantung hanya pada materialized view yang terlambat.

## 12. Bulk Operation dan Transaction

- Import Excel/TSV menggunakan validasi batch dan bulk insert, bukan satu insert per row.
- Gunakan `createMany`/bulk SQL dengan chunk yang terukur.
- Pembuatan Local File dan konsumsi kuota berlangsung dalam transaction yang sama.
- Aktivasi subscription, pencatatan payment, dan grant kuota berlangsung atomik.
- Refund berhasil, ledger adjustment, dan `access_adjusted_at` diperbarui atomik.
- Transaction harus singkat; jangan melakukan panggilan Xendit, Resend, storage, AI, atau LibreOffice di dalam transaction database.
- Gunakan outbox/event pattern atau job idempotent untuk efek eksternal setelah commit apabila diperlukan.

## 13. Analisis Execution Plan

Query kritis wajib diuji menggunakan dataset representatif.

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT
  id,
  title,
  tax_year,
  status,
  updated_at
FROM local_files
WHERE organization_id = $1
  AND archived_at IS NULL
ORDER BY updated_at DESC, id DESC
LIMIT 25;
```

Periksa:

- Sequential scan pada tabel besar yang seharusnya selektif.
- Jumlah actual rows dibanding estimated rows.
- Sort besar atau spill ke disk.
- Nested loop dengan jumlah loop tidak wajar.
- Buffer read tinggi.
- Filter yang membuang sebagian besar baris setelah scan.

`EXPLAIN ANALYZE` benar-benar menjalankan query. Untuk query perubahan data, gunakan environment pengujian atau transaction yang aman dan di-rollback.

## 14. Statistik dan Maintenance PostgreSQL

- Aktifkan autovacuum dan auto-analyze dengan konfigurasi yang sesuai volume.
- Jalankan `ANALYZE` setelah bulk import besar jika statistik belum diperbarui.
- Gunakan `pg_stat_statements` jika tersedia untuk menemukan query dengan total time, mean time, atau call count tinggi.
- Pantau index usage, dead tuples, lock wait, connection pool, dan transaction duration.
- Partitioning dipertimbangkan untuk `audit_logs`, integration events, dan webhook events setelah volume serta retention menunjukkan kebutuhan nyata.
- Sharding bukan kebutuhan MVP dan tidak diterapkan sebelum batas satu PostgreSQL instance terbukti.

## 15. Stored Procedure dan Database Function

Stored procedure tidak menjadi lokasi utama business rule TP Docs. Aturan membership, entitlement, Local File, refund, dan document generation tetap berada pada domain/application layer NestJS agar mudah diuji dan ditelusuri.

Stored procedure atau database function hanya dipertimbangkan apabila:

- Operasi bersifat set-based dan harus atomik pada data berukuran besar.
- Query yang sama terbukti berulang serta menjadi bottleneck melalui profiling.
- Implementasi Prisma atau SQL biasa tidak memenuhi target performa.
- Kontrak input, output, permission, migration, dan rollback terdokumentasi.

Contoh kandidat adalah maintenance ledger atau agregasi operasional, bukan panggilan provider eksternal. Procedure tidak boleh memanggil Xendit, Resend, AI, atau document converter.

Query daftar file, histori dokumen, dan dashboard dilarang melakukan join ke `file_blob_chunks`. Isi biner hanya dibaca oleh `BinaryFileRepository` saat upload, scan, generate, preview, atau download menggunakan query berurutan berdasarkan `(file_id, chunk_index)`. Streaming wajib membatasi concurrency agar connection pool dan memori tidak habis.

## 16. Query Logging dan Data Rahasia

- Production tidak mencatat nilai bind parameter yang memuat NPWP, uraian Local File, transaksi, atau data finansial mentah.
- Slow-query log menggunakan query fingerprint dan metadata aman.
- Prisma query event tidak boleh mengirim isi parameter sensitif ke error tracker.
- Correlation ID boleh dicatat; prompt AI dan snapshot dokumen tidak boleh dicatat.

## 17. Pengujian Performa dan Acceptance Criteria

- Endpoint daftar mempunyai query count tetap saat jumlah hasil bertambah.
- Test N+1 membandingkan 5 dan 100 parent rows; jumlah query tidak boleh bertambah linear.
- Daftar Local File maksimum menggunakan 1–2 query sesuai response shape.
- Daftar tugas Karyawan menggunakan satu query join atau dua batch query.
- Review and Generate mempunyai query budget tetap serta tidak melakukan query dari renderer.
- Import transaksi tidak melakukan satu insert per row.
- Pagination selalu mempunyai batas dan ordering deterministik.
- Query tenant selalu memuat `organization_id` atau berjalan dari relasi tenant yang telah diverifikasi.
- Query kritis memiliki `EXPLAIN (ANALYZE, BUFFERS)` yang disimpan sebagai artefak pengujian performa, bukan source of truth permanen.
- Tidak ada endpoint API yang menggunakan raw `SELECT *`.
- Indeks baru harus mempunyai query use case dan bukti plan; indeks yang tidak digunakan dievaluasi berkala.

Contoh query-count test:

```ts
it('does not grow query count with the number of local files', async () => {
  const smallCount = await captureQueryCount(() =>
    repository.listWithAssignments({ organizationId, take: 5 }),
  );

  const largeCount = await captureQueryCount(() =>
    repository.listWithAssignments({ organizationId, take: 100 }),
  );

  expect(largeCount).toBeLessThanOrEqual(smallCount + 1);
});
```

## 18. Checklist Code Review

- Apakah query memuat tenant filter?
- Apakah hanya field yang dibutuhkan yang dipilih?
- Apakah terdapat Prisma call di dalam loop?
- Apakah pagination dan ordering dibatasi?
- Apakah relasi besar diambil tanpa filter?
- Apakah composite index sesuai urutan filter dan sort?
- Apakah transaction singkat dan bebas external call?
- Apakah query count diuji untuk response dengan relasi?
- Apakah data rahasia tidak masuk query log?
- Apakah raw SQL menggunakan parameter binding?

## 19. Referensi Teknis

- [Prisma ORM — Query optimization and the N+1 problem](https://www.prisma.io/docs/orm/v7/prisma-client/queries/advanced/query-optimization-performance)
- [Prisma ORM — Relation queries and load strategy](https://www.prisma.io/docs/orm/v7/prisma-client/queries/relation-queries)
- [PostgreSQL — EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
- [PostgreSQL — CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html)

