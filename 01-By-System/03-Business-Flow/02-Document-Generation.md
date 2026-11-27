# Document Generation Business Flow — TP Docs (By System)

> Template master dan hasil DOCX/PDF disimpan sebagai file biner PostgreSQL. Worker membaca serta menulisnya melalui `BinaryFileRepository`; pengguna mengunduh melalui streaming NestJS setelah validasi akses dan paket.

## 1. Tujuan dan Cakupan

Dokumen ini menjelaskan proses detail sejak data Local File selesai diisi sampai draft dokumen dihasilkan, disimpan pada history, dibuka kembali, atau diduplikasi.

Flow ini merupakan kelanjutan dari [03-Local-File-Management.md](./03-Local-File-Management.md) dan mengacu pada:

- `UI-04.11 — Review and Generate`.
- `UI-04.11.1 — Preview Ringkasan`.
- `UI-04.11.2 — Generate Docs`.
- `UI-06 — Melihat History Dokumen`.

Dokumen yang dihasilkan TP Docs tetap berstatus **draft** dan wajib ditinjau pengguna yang melakukan generate. Sistem tidak mengirim dokumen langsung kepada DJP.

## 2. Prasyarat Generate

| Pemeriksaan | Kondisi yang Harus Dipenuhi |
| --- | --- |
| Identitas dokumen | Entitas dan tahun pajak tersedia |
| Profil perusahaan | Seluruh data wajib telah diisi |
| Kepemilikan dan manajemen | Tabel pemegang saham serta pengurus valid |
| Bisnis dan proses | Uraian bisnis tersedia; skema proses bisnis AI bersifat opsional dan hanya digunakan jika telah diterima User Induk |
| Transaksi | Data afiliasi, independen, dan summary dapat diproses |
| Analisis FAR | Fungsi, aset, risiko, metode, pihak dianalisis, PLI, dan alasan tersedia |
| Pembanding | Strategi, PLI, pembanding final, serta rentang interkuartil tersedia |
| Hak akses | User Induk, Karyawan pembuat, atau Karyawan dengan penugasan aktif memiliki akses terhadap dokumen dan organisasi |
| Paket | Format output yang dipilih diizinkan oleh paket aktif |

Kuota dokumen telah digunakan ketika dokumen baru berhasil dibuat. Menekan generate tidak mengurangi kuota dokumen untuk kedua kalinya.

Bagian Review & Generate dapat dibuka kapan saja oleh pengguna yang berhak mengakses dokumen. Urutan pengisian bagian 01–10 bebas. User Induk, Karyawan pembuat, dan Karyawan dengan penugasan aktif dapat memberi konfirmasi final serta memulai generate setelah seluruh validasi lulus. Hasil dokumen privat tetap privat dari User Induk hingga dibagikan atau diambil alih setelah membership pembuat dicabut.

## 3. Flow Pemeriksaan Kesiapan Generate

```mermaid
flowchart TD
    DG_START([Pengguna berhak memilih Review and Generate]) --> DG_LOAD[Muat data Local File terbaru]
    DG_LOAD --> DG_ACCESS[Validasi organisasi, hak akses, dan status dokumen]
    DG_ACCESS --> DG_ACCESS_OK{"Akses valid?"}
    DG_ACCESS_OK -- "Tidak" --> DG_DENIED[Tolak proses dan tampilkan alasan akses]
    DG_DENIED --> DG_END_DENIED([Generate tidak dijalankan])

    DG_ACCESS_OK -- "Ya" --> DG_VALIDATE[Validasi seluruh bagian wajib]
    DG_VALIDATE --> DG_COMPLETE{"Data lengkap dan valid?"}
    DG_COMPLETE -- "Tidak" --> DG_LIST[Tampilkan pratinjau kelengkapan dan tautan ke bagian yang harus diperbaiki]
    DG_LIST --> DG_SAVE[Simpan dokumen sebagai Draft atau In Progress; tombol Generate tidak aktif]
    DG_SAVE --> DG_RETURN([Kembali ke bagian Local File terkait])

    DG_COMPLETE -- "Ya" --> DG_CONSISTENCY[Periksa konsistensi entitas, tahun pajak, transaksi, metode, PLI, dan pembanding]
    DG_CONSISTENCY --> DG_CONSISTENT{"Data konsisten?"}
    DG_CONSISTENT -- "Tidak" --> DG_CONFLICT[Tampilkan konflik data dan sumbernya]
    DG_CONFLICT --> DG_RETURN
    DG_CONSISTENT -- "Ya" --> DG_READY[Ubah status menjadi Ready to Generate]
    DG_READY --> DG_PREVIEW([Lanjut ke Preview Ringkasan])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class DG_START,DG_END_DENIED,DG_RETURN,DG_PREVIEW terminal;
    class DG_LOAD,DG_ACCESS,DG_DENIED,DG_VALIDATE,DG_LIST,DG_SAVE,DG_CONSISTENCY,DG_CONFLICT,DG_READY process;
    class DG_ACCESS_OK,DG_COMPLETE,DG_CONSISTENT decision;
```

## 4. Flow Preview Ringkasan dan Warning

Preview harus menampilkan data inti sebelum proses generate dimulai.

```mermaid
flowchart TD
    DP_START([Status Ready to Generate]) --> DP_ENTITY[Tampilkan entitas]
    DP_ENTITY --> DP_YEAR[Tampilkan tahun pajak]
    DP_YEAR --> DP_CHARACTER[Tampilkan karakteristik usaha pihak yang dianalisis]
    DP_CHARACTER --> DP_METHOD[Tampilkan metode Transfer Pricing]
    DP_METHOD --> DP_TRANSACTION[Tampilkan ringkasan transaksi afiliasi]
    DP_TRANSACTION --> DP_COMPARABLE[Tampilkan pembanding final dan nilai PLI]
    DP_COMPARABLE --> DP_RANGE[Bandingkan hasil pihak dianalisis dengan rentang kewajaran]
    DP_RANGE --> DP_IN_RANGE{"Hasil berada dalam rentang wajar?"}

    DP_IN_RANGE -- "Ya" --> DP_CONFIRM[Tampilkan konfirmasi data siap dibuat menjadi dokumen]
    DP_IN_RANGE -- "Tidak" --> DP_WARNING[Tampilkan warning di luar rentang wajar]
    DP_WARNING --> DP_REVIEW[User Induk meninjau metode, transaksi, PLI, pembanding, dan alasannya]
    DP_REVIEW --> DP_DECISION{"Keputusan User Induk?"}
    DP_DECISION -- "Perbaiki data" --> DP_BACK[Simpan dan kembali ke Local File Management]
    DP_BACK --> DP_END_BACK([Generate ditunda])
    DP_DECISION -- "Lanjut setelah ditinjau" --> DP_CONFIRM

    DP_CONFIRM --> DP_FINAL_CHECK{"Konfirmasi final diberikan?"}
    DP_FINAL_CHECK -- "Tidak" --> DP_END_BACK
    DP_FINAL_CHECK -- "Ya" --> DP_GENERATE([Lanjut ke Generate Docs])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class DP_START,DP_END_BACK,DP_GENERATE terminal;
    class DP_ENTITY,DP_YEAR,DP_CHARACTER,DP_METHOD,DP_TRANSACTION,DP_COMPARABLE,DP_RANGE,DP_WARNING,DP_REVIEW,DP_BACK,DP_CONFIRM process;
    class DP_IN_RANGE,DP_DECISION,DP_FINAL_CHECK decision;
```

Warning di luar rentang wajar tidak otomatis mengubah data atau memberikan opini pajak. Keputusan untuk memperbaiki data atau melanjutkan generate tetap berada pada User Induk.

## 5. Flow Generate Dokumen

```mermaid
flowchart TD
    GG_START([Pengguna berhak mengonfirmasi Generate Docs]) --> GG_FORMAT[Pilih format output yang tersedia pada paket]
    GG_FORMAT --> GG_ALLOWED{"Format diizinkan paket aktif?"}
    GG_ALLOWED -- "Tidak" --> GG_UPGRADE[Tampilkan batas paket dan pilihan upgrade]
    GG_UPGRADE --> GG_END_UPGRADE([Generate format tersebut dibatalkan])

    GG_ALLOWED -- "Ya" --> GG_SNAPSHOT[Buat snapshot data final dan nomor versi]
    GG_SNAPSHOT --> GG_JOB[Buat proses generate dan tampilkan progress]
    GG_JOB --> GG_TEMPLATE[Susun struktur serta template Local File]
    GG_TEMPLATE --> GG_CONTENT[Masukkan profil, kepemilikan, bisnis, industri, transaksi, FAR, dan pembanding]
    GG_CONTENT --> GG_TABLE[Render tabel, summary transaksi, rentang interkuartil, dan skema AI yang telah diterima jika tersedia]
    GG_TABLE --> GG_FILE[Buat file output]
    GG_FILE --> GG_SUCCESS{"File berhasil dibuat?"}

    GG_SUCCESS -- "Tidak" --> GG_FAIL[Catat kegagalan tanpa menghapus data atau snapshot]
    GG_FAIL --> GG_RETRY{"Coba generate ulang?"}
    GG_RETRY -- "Ya" --> GG_JOB
    GG_RETRY -- "Tidak" --> GG_FAILED_STATUS[Ubah status generate menjadi Failed]
    GG_FAILED_STATUS --> GG_END_FAIL([User dapat mencoba kembali kemudian])

    GG_SUCCESS -- "Ya" --> GG_SCAN[Validasi file dan hasil render]
    GG_SCAN --> GG_STORE[Simpan file pada penyimpanan organisasi]
    GG_STORE --> GG_HISTORY[Tambahkan nama dokumen, tahun pajak, status, versi, dan waktu pembaruan ke history]
    GG_HISTORY --> GG_STATUS[Ubah status menjadi Generated]
    GG_STATUS --> GG_NOTIFY[Tampilkan keberhasilan dan pilihan membuka atau mengunduh]
    GG_NOTIFY --> GG_END([Generate selesai])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class GG_START,GG_END_UPGRADE,GG_END_FAIL,GG_END terminal;
    class GG_FORMAT,GG_UPGRADE,GG_SNAPSHOT,GG_JOB,GG_TEMPLATE,GG_CONTENT,GG_TABLE,GG_FILE,GG_FAIL,GG_FAILED_STATUS,GG_SCAN,GG_STORE,GG_HISTORY,GG_STATUS,GG_NOTIFY process;
    class GG_ALLOWED,GG_SUCCESS,GG_RETRY decision;
```

Target proses generate adalah maksimal lima menit. Sistem harus menampilkan progress sehingga pengguna tidak perlu mengirim permintaan generate berulang kali.

## 6. Flow History, Buka, dan Duplikasi Dokumen

```mermaid
flowchart TD
    DH_START([Buka History Dokumen]) --> DH_LIST[Tampilkan nama dokumen, tahun pajak, status, dan waktu diperbarui]
    DH_LIST --> DH_SELECT[Pilih dokumen]
    DH_SELECT --> DH_ACTION{"Aktivitas yang dipilih?"}

    DH_ACTION -- "Buka" --> DH_OPEN[Buka detail, data, dan versi dokumen]
    DH_OPEN --> DH_DOWNLOAD{"Unduh hasil generate?"}
    DH_DOWNLOAD -- "Ya" --> DH_URL[Backend memvalidasi tenant, role, penugasan, status scan, paket, dan hak format]
    DH_URL --> DH_FILE[NestJS stream file dari PostgreSQL ke pengguna]
    DH_DOWNLOAD -- "Tidak" --> DH_LIST
    DH_FILE --> DH_LIST

    DH_ACTION -- "Duplikasi" --> DH_CONFIRM[Konfirmasi entitas, tahun pajak baru, dan nama dokumen]
    DH_CONFIRM --> DH_QUOTA{"Kuota pembuatan dokumen tersedia?"}
    DH_QUOTA -- "Tidak" --> DH_LIMIT[Arahkan ke langganan atau add-on kuota]
    DH_LIMIT --> DH_LIST
    DH_QUOTA -- "Ya" --> DH_COPY[Salin data yang diizinkan ke dokumen baru]
    DH_COPY --> DH_NEW_VERSION[Dokumen baru berstatus Draft dan memiliki histori sendiri]
    DH_NEW_VERSION --> DH_LIST

    DH_ACTION -- "Kembali" --> DH_END([Selesai])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class DH_START,DH_END terminal;
    class DH_LIST,DH_SELECT,DH_OPEN,DH_URL,DH_FILE,DH_CONFIRM,DH_LIMIT,DH_COPY,DH_NEW_VERSION process;
    class DH_ACTION,DH_DOWNLOAD,DH_QUOTA decision;
```

### 6.1 Perbandingan History Versi

Pengguna yang berhak memilih satu Local File dari History Dokumen, lalu membuka panel **History Versi**. Daftar versi hasil generate ditampilkan terbaru dahulu dan dimuat bertahap agar puluhan versi tetap dapat ditelusuri. Dokumen privat buatan Karyawan tidak ditampilkan kepada User Induk sebelum dibagikan atau diambil alih. Draft yang sedang dikerjakan ditampilkan terpisah; draft bukan versi hasil generate.

Pengguna dapat memilih tepat dua versi hasil generate dari Local File yang sama sebagai titik A dan B. Sistem memeriksa izin akses, mengambil dua `document_snapshots` immutable yang terkait dengan `document_versions`, lalu membandingkan data terstruktur per bagian. Ringkasan menampilkan bagian yang berubah dan nilai sebelum/sesudah; rincian dapat dibuka per bagian. Perbedaan format DOCX/PDF atau template saja tidak boleh diklaim sebagai perubahan data. Jika snapshot menggunakan schema berbeda, sistem menormalisasi dengan pemetaan yang didukung atau menampilkan bahwa perbandingan bagian tersebut tidak tersedia.

Perbandingan bersifat baca saja, tidak mengubah draft, versi lama, nomor versi, maupun kuota. Versi dari Local File berbeda, versi yang gagal dibuat, atau dokumen di luar hak akses harus ditolak. Buka versi, unduh, dan duplikasi tetap merupakan tindakan terpisah.

## 7. Status Dokumen dan Generate

| Status | Makna | Aktivitas Berikutnya |
| --- | --- | --- |
| `Draft` | Dokumen baru atau belum lengkap | Lanjutkan pengisian Local File |
| `In Progress` | Sebagian data telah disimpan | Buka bagian yang belum lengkap |
| `Ready to Generate` | Validasi minimum berhasil | Preview ringkasan |
| `Generating` | File sedang dibuat | Pantau progress; jangan mengirim proses duplikat |
| `Generated` | File berhasil disimpan | Buka, unduh, atau buat versi berikutnya |
| `Failed` | Proses file gagal | Jalankan generate ulang tanpa membuat dokumen baru |

## 8. Exception dan Aturan Penting

1. Perubahan data setelah generate tidak mengubah file lama; generate berikutnya membuat versi baru.
2. Proses generate ulang harus idempoten dan tidak membuat file atau transaksi kuota ganda akibat klik berulang.
3. Kegagalan generate tidak menghapus data Local File.
4. Warning rentang kewajaran harus tetap tercatat pada versi yang dihasilkan jika User Induk memilih melanjutkan.
5. File hanya dapat diakses melalui streaming endpoint NestJS setelah validasi organisasi pemilik, role, penugasan aktif jika berlaku, status scan, paket, dan hak format; akses dicatat dalam audit log.
6. Isi dokumen, NPWP lengkap, dan data finansial mentah tidak boleh dicatat pada application log.
7. Administrator tidak dapat membuka isi dokumen melalui portal operasional.
8. Draft hasil sistem tidak merupakan opini atau jaminan kepatuhan pajak.

## 9. Referensi

- [Business Flow Overview](./00-Business-Flow-Overview.md).
- [Local File Management](./03-Local-File-Management.md).
- [Use Case User Induk](../02-Use-Case/Organization-Owner.md): `UI-04.11` dan `UI-06`.
- [Document Generation Architecture](../05-Architecture/Document-Generation-Architecture.md).
- [Non-Functional Requirements](../01-Requirements/Non-Functional-Requirements.md).
