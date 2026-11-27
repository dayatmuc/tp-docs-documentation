# Local File Management Business Flow — TP Docs (With Consultant)

> Seluruh upload Local File melewati NestJS sebagai `multipart/form-data` dengan batas 50 MB. Sistem menyimpan metadata dan chunk biner di PostgreSQL, lalu hanya mengizinkan pemrosesan setelah checksum lengkap dan hasil malware scan berstatus `CLEAN`.

## 1. Tujuan dan Cakupan

Dokumen ini menjelaskan pengelolaan Local File oleh User Induk, Karyawan pembuat, dan Karyawan yang ditugaskan, mulai dari membuat atau membuka dokumen, mengisi data, menyimpan draft, sampai review dan generate.

Urutan flow mengikuti `UI-04 — Mengisi Data Dokumen`, `UI-05 — Kelola Data Pembanding dan Seleksi`, serta overview pada [00-Business-Flow-Overview.md](./00-Business-Flow-Overview.md).

### 1.1 Posisi Skema AI dalam Local File

Local File mencakup identitas dan kegiatan usaha, transaksi afiliasi dan independen, penerapan prinsip kewajaran termasuk FAR dan metode TP, informasi keuangan, serta kejadian nonkeuangan yang relevan. Uraian proses bisnis merupakan bagian wajib dari penjelasan kegiatan operasional. Skema AI hanya merupakan visual pendukung opsional untuk uraian tersebut dan bukan pengganti analisis transaksi, FAR, metode TP, atau pembanding.

Jika Skema AI tidak digunakan atau layanan AI gagal, pengguna yang berhak tetap dapat menyimpan uraian bisnis dan melanjutkan penyusunan Local File.

Dokumen buatan User Induk dapat ditugaskan kepada Karyawan. Dokumen buatan Karyawan memakai kuota organisasi dan dimulai privat; hanya pembuat aktif yang dapat memberi atau mencabut akses lihat/ubah User Induk. User Induk tidak dapat menugaskan dokumen tersebut kepada Karyawan lain. Dokumen privat hanya dihitung pada kuota agregat User Induk tanpa metadata, isi, versi, atau hasil. Jika membership pembuat dicabut, akses diambil alih User Induk dengan audit.

## 2. Siklus Awal Dokumen Local File

```mermaid
flowchart TD
    LF_START([Dashboard User Induk atau Karyawan aktif]) --> LF_ACTION{"Buat dokumen baru atau buka dokumen?"}

    LF_ACTION -- "Buat baru" --> LF_ACCESS[Validasi organisasi, paket, membership, dan hak pembuat]
    LF_ACCESS --> LF_QUOTA{"Kuota dokumen tersedia?"}
    LF_QUOTA -- "Tidak" --> LF_LIMIT[Hanya User Induk dapat mengelola kapasitas paket]
    LF_LIMIT --> LF_END_LIMIT([Dokumen belum dibuat])
    LF_QUOTA -- "Ya" --> LF_IDENTITY[Input nama dokumen, entitas, tahun pajak, dan jenis dokumen]
    LF_IDENTITY --> LF_VALIDATE_ID{"Identitas dokumen valid dan tidak duplikat?"}
    LF_VALIDATE_ID -- "Tidak" --> LF_ID_ERROR[Perbaiki identitas dokumen]
    LF_ID_ERROR --> LF_IDENTITY
    LF_VALIDATE_ID -- "Ya" --> LF_CREATE[Buat dokumen dan gunakan satu kuota; buatan Karyawan privat]
    LF_CREATE --> LF_DRAFT[Status awal Draft]

    LF_ACTION -- "Buka draft" --> LF_HISTORY[Buka daftar history dokumen]
    LF_HISTORY --> LF_SELECT[Pilih dokumen yang dapat diakses]
    LF_SELECT --> LF_LOAD[Muat data dan progress terakhir]
    LF_LOAD --> LF_DRAFT

    LF_DRAFT --> LF_STEP[Masuk ke tahapan pengisian Local File]

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class LF_START,LF_END_LIMIT terminal;
    class LF_ACCESS,LF_LIMIT,LF_IDENTITY,LF_ID_ERROR,LF_CREATE,LF_DRAFT,LF_HISTORY,LF_SELECT,LF_LOAD,LF_STEP process;
    class LF_ACTION,LF_QUOTA,LF_VALIDATE_ID decision;
```

Kuota berkurang hanya setelah dokumen baru berhasil dibuat. Membuka draft, mengubah data, menyimpan, atau melakukan generate ulang tidak menggunakan kuota dokumen baru.

## 3. Tahap 1 — Profil Perusahaan (`UI-04.01`)

```mermaid
flowchart TD
    P_START([Mulai Profil Perusahaan]) --> P_ENTITY[Input nama badan atau entitas]
    P_ENTITY --> P_ALIAS[Input singkatan atau alias]
    P_ALIAS --> P_BUSINESS[Input bidang usaha]
    P_BUSINESS --> P_ESTABLISH[Input tanggal pendirian]
    P_ESTABLISH --> P_COMMERCIAL[Input tanggal mulai komersial]
    P_COMMERCIAL --> P_ADDRESS[Input alamat kantor atau pabrik]
    P_ADDRESS --> P_EMPLOYEE[Input informasi jumlah karyawan]
    P_EMPLOYEE --> P_TAX_YEAR[Input tahun pajak]
    P_TAX_YEAR --> P_TYPE[Input jenis dokumen]
    P_TYPE --> P_VALIDATE[Validasi field wajib, format, dan konsistensi tanggal]
    P_VALIDATE --> P_VALID{"Profil valid?"}
    P_VALID -- "Tidak" --> P_ERROR[Tandai field yang harus diperbaiki]
    P_ERROR --> P_ENTITY
    P_VALID -- "Ya" --> P_SAVE[Simpan profil dan tandai tahap selesai]
    P_SAVE --> P_NEXT([Lanjut ke Kepemilikan dan Manajemen])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class P_START,P_NEXT terminal;
    class P_ENTITY,P_ALIAS,P_BUSINESS,P_ESTABLISH,P_COMMERCIAL,P_ADDRESS,P_EMPLOYEE,P_TAX_YEAR,P_TYPE,P_VALIDATE,P_ERROR,P_SAVE process;
    class P_VALID decision;
```

## 4. Tahap 2–4 — Kepemilikan, Bisnis, AI, dan Keuangan

```mermaid
flowchart TD
    K_START([Tahap 2: Kepemilikan dan Manajemen]) --> K_SHAREHOLDER[Tambah baris pemegang saham]
    K_SHAREHOLDER --> K_SHARE_DATA[Input nama, nilai nominal, dan persentase kepemilikan]
    K_SHARE_DATA --> K_SHARE_MORE{"Ada pemegang saham lain?"}
    K_SHARE_MORE -- "Ya" --> K_SHAREHOLDER
    K_SHARE_MORE -- "Tidak" --> K_TOTAL[Validasi total persentase kepemilikan]
    K_TOTAL --> K_TOTAL_OK{"Total dan nilai valid?"}
    K_TOTAL_OK -- "Tidak" --> K_SHARE_DATA
    K_TOTAL_OK -- "Ya" --> K_MANAGEMENT[Tambah data komisaris atau direksi]
    K_MANAGEMENT --> K_MANAGER_DATA[Input jabatan dan nama pengurus atau direksi]
    K_MANAGER_DATA --> K_MANAGER_MORE{"Ada pengurus lain?"}
    K_MANAGER_MORE -- "Ya" --> K_MANAGEMENT
    K_MANAGER_MORE -- "Tidak" --> K_SAVE[Simpan Kepemilikan dan Manajemen]

    K_SAVE --> B_START[Tahap 3: Isi Bisnis, Produk, dan Strategi]
    B_START --> B_DESCRIPTION[Isi uraian aktivitas, produk atau jasa, rantai nilai, dan strategi]
    B_DESCRIPTION --> B_STEPS_OPTION{"Tambahkan daftar langkah proses?"}
    B_STEPS_OPTION -- "Ya" --> B_STEPS[Input dan urutkan aktivitas serta aktor/departemen opsional]
    B_STEPS_OPTION -- "Tidak" --> B_AI_OPTION{"Gunakan Skema AI?"}
    B_STEPS --> B_AI_OPTION

    B_AI_OPTION -- "Tidak" --> B_SAVE_TEXT[Simpan uraian tanpa skema AI]
    B_AI_OPTION -- "Ya" --> B_AI[Generate fase dinamis dari input pengguna]
    B_AI --> B_AI_SUCCESS{"Generation berhasil dan valid?"}
    B_AI_SUCCESS -- "Tidak" --> B_AI_FAILURE[Tampilkan kegagalan tanpa menghapus input]
    B_AI_FAILURE --> B_RETRY{"Generate ulang?"}
    B_RETRY -- "Tidak" --> B_SAVE_TEXT
    B_RETRY -- "Ya" --> B_REVISION[Perbaiki uraian atau daftar langkah]
    B_REVISION --> B_AI
    B_AI_SUCCESS -- "Ya" --> B_AI_RESULT[Tampilkan draft fase proses bisnis]
    B_AI_RESULT --> B_AI_VALID{"Skema sesuai input?"}
    B_AI_VALID -- "Tidak" --> B_REVISION
    B_AI_VALID -- "Ya" --> B_ACCEPT[Terima skema AI]
    B_ACCEPT --> B_SAVE_AI[Simpan uraian dan skema yang diterima]

    B_SAVE_TEXT --> F_START[Tahap 4: Isi Ringkasan Laporan Keuangan]
    B_SAVE_AI --> F_START
    F_START --> F_INPUT[Input data keuangan yang diwajibkan untuk analisis]
    F_INPUT --> F_VALIDATE[Validasi periode, format angka, dan konsistensi nilai]
    F_VALIDATE --> F_VALID{"Ringkasan keuangan valid?"}
    F_VALID -- "Tidak" --> F_INPUT
    F_VALID -- "Ya" --> F_SAVE[Simpan ringkasan keuangan]
    F_SAVE --> F_NEXT([Lanjut ke Afiliasi dan Pesaing])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class K_START,F_NEXT terminal;
    class K_SHAREHOLDER,K_SHARE_DATA,K_TOTAL,K_MANAGEMENT,K_MANAGER_DATA,K_SAVE,B_START,B_DESCRIPTION,B_STEPS,B_AI,B_AI_FAILURE,B_REVISION,B_AI_RESULT,B_ACCEPT,B_SAVE_TEXT,B_SAVE_AI,F_START,F_INPUT,F_VALIDATE,F_SAVE process;
    class K_SHARE_MORE,K_TOTAL_OK,K_MANAGER_MORE,B_STEPS_OPTION,B_AI_OPTION,B_AI_SUCCESS,B_RETRY,B_AI_VALID,F_VALID decision;
```

Uraian proses bisnis merupakan langkah wajib. Daftar langkah proses dan Skema AI bersifat opsional. Jika daftar langkah diberikan, setiap langkah harus terwakili tepat satu kali dalam fase AI dan urutannya harus dipertahankan. Pengguna yang berhak tidak mengedit kartu fase secara langsung; koreksi dilakukan dengan memperbaiki input lalu memilih generate ulang.

## 5. Tahap 5–6 — Pihak Afiliasi, Pesaing, dan Gambaran Industri

```mermaid
flowchart TD
    A_START([Tahap 5: Pihak Afiliasi dan Pesaing]) --> A_AFFILIATE[Tambah perusahaan afiliasi]
    A_AFFILIATE --> A_AFFILIATE_DATA[Input nama perusahaan, aktivitas usaha atau peran, dan bentuk hubungan]
    A_AFFILIATE_DATA --> A_MORE{"Ada afiliasi lain?"}
    A_MORE -- "Ya" --> A_AFFILIATE
    A_MORE -- "Tidak" --> A_COMPETITOR[Tambah pesaing utama]
    A_COMPETITOR --> A_COMPETITOR_DATA[Input nama perusahaan dan domisili atau negara]
    A_COMPETITOR_DATA --> A_COMPETITOR_MORE{"Ada pesaing lain?"}
    A_COMPETITOR_MORE -- "Ya" --> A_COMPETITOR
    A_COMPETITOR_MORE -- "Tidak" --> A_SAVE[Simpan pihak afiliasi dan pesaing]

    A_SAVE --> I_START[Tahap 6: Gambaran Industri]
    I_START --> I_SOURCE{"Sumber data industri?"}
    I_SOURCE -- "Word / PDF" --> I_UPLOAD[Unggah file gambaran industri]
    I_UPLOAD --> I_FILE_VALIDATE{"Tipe, ukuran, dan malware scan valid?"}
    I_FILE_VALIDATE -- "Tidak" --> I_REJECT[Karantina atau tolak file dan tampilkan alasan]
    I_REJECT --> I_SOURCE
    I_FILE_VALIDATE -- "Ya" --> I_EXTRACT[Proses isi file sebagai sumber gambaran industri]
    I_SOURCE -- "Deskripsi teks" --> I_TEXT[Input deskripsi industri]
    I_EXTRACT --> I_SCOPE
    I_TEXT --> I_SCOPE[Isi definisi dan ruang lingkup industri]
    I_SCOPE --> I_IMPORT[Isi ketergantungan impor dan pasokan]
    I_IMPORT --> I_PROSPECT[Isi prospek masa depan dan kondisi pasar]
    I_PROSPECT --> I_SAVE[Simpan gambaran industri dan sumbernya]
    I_SAVE --> I_NEXT([Lanjut ke Transaksi Afiliasi])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class A_START,I_NEXT terminal;
    class A_AFFILIATE,A_AFFILIATE_DATA,A_COMPETITOR,A_COMPETITOR_DATA,A_SAVE,I_START,I_UPLOAD,I_REJECT,I_EXTRACT,I_TEXT,I_SCOPE,I_IMPORT,I_PROSPECT,I_SAVE process;
    class A_MORE,A_COMPETITOR_MORE,I_SOURCE,I_FILE_VALIDATE decision;
```

## 6. Tahap 7 — Transaksi Afiliasi (`UI-04.08`)

```mermaid
flowchart TD
    T_START([Mulai Transaksi Afiliasi]) --> T_MODE{"Metode input?"}

    T_MODE -- "Template Excel" --> T_DOWNLOAD[Download template Excel]
    T_DOWNLOAD --> T_FILL[Isi data sesuai kolom template]
    T_FILL --> T_UPLOAD[Upload file Excel transaksi]
    T_MODE -- "Upload Excel" --> T_UPLOAD
    T_MODE -- "Copy TSV" --> T_TSV[Salin dan tempel data dalam format TSV]
    T_MODE -- "Input manual" --> T_AFFILIATE[Tambah transaksi pihak afiliasi]

    T_UPLOAD --> T_FILE_VALIDATE[Validasi tipe file, header, format baris, dan duplikasi]
    T_TSV --> T_FILE_VALIDATE
    T_FILE_VALIDATE --> T_FILE_OK{"Data impor valid?"}
    T_FILE_OK -- "Tidak" --> T_IMPORT_ERROR[Tampilkan baris dan kolom yang harus diperbaiki]
    T_IMPORT_ERROR --> T_MODE
    T_FILE_OK -- "Ya" --> T_IMPORT[Masukkan data valid ke tabel transaksi]
    T_IMPORT --> T_AFFILIATE

    T_AFFILIATE --> T_AFFILIATE_DATA[Input jenis, lawan, dan deskripsi transaksi afiliasi]
    T_AFFILIATE_DATA --> T_METHOD[Pilih metode TP: CUP, CUT, RPM, C+, PSM, TNMM, atau Valuasi]
    T_METHOD --> T_AFFILIATE_MORE{"Ada transaksi afiliasi lain?"}
    T_AFFILIATE_MORE -- "Ya" --> T_AFFILIATE
    T_AFFILIATE_MORE -- "Tidak" --> T_INDEPENDENT[Tambah transaksi pihak independen]
    T_INDEPENDENT --> T_INDEPENDENT_DATA[Input jenis transaksi, lawan transaksi, dan nilai rupiah]
    T_INDEPENDENT_DATA --> T_INDEPENDENT_MORE{"Ada transaksi independen lain?"}
    T_INDEPENDENT_MORE -- "Ya" --> T_INDEPENDENT
    T_INDEPENDENT_MORE -- "Tidak" --> T_RECONCILE[Validasi nilai dan rekonsiliasi data transaksi]
    T_RECONCILE --> T_SUMMARY[Buat tabel summary transaksi TP otomatis]
    T_SUMMARY --> T_RENT[Ringkas pendapatan sewa berelasi dan independen]
    T_RENT --> T_INTEREST[Ringkas biaya bunga pinjaman intra-grup dan totalnya]
    T_INTEREST --> T_SAVE[Simpan transaksi serta summary]
    T_SAVE --> T_NEXT([Lanjut ke Analisis FAR])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class T_START,T_NEXT terminal;
    class T_DOWNLOAD,T_FILL,T_UPLOAD,T_TSV,T_FILE_VALIDATE,T_IMPORT_ERROR,T_IMPORT,T_AFFILIATE,T_AFFILIATE_DATA,T_METHOD,T_INDEPENDENT,T_INDEPENDENT_DATA,T_RECONCILE,T_SUMMARY,T_RENT,T_INTEREST,T_SAVE process;
    class T_MODE,T_FILE_OK,T_AFFILIATE_MORE,T_INDEPENDENT_MORE decision;
```

## 7. Tahap 8 — Analisis FAR (`UI-04.09`)

```mermaid
flowchart TD
    FAR_START([Mulai Analisis FAR]) --> FAR_FUNCTION[Tambah fungsi yang dilakukan]
    FAR_FUNCTION --> FAR_FUNCTION_DATA[Input fungsi dan deskripsi pelaksanaan]
    FAR_FUNCTION_DATA --> FAR_FUNCTION_MORE{"Ada fungsi lain?"}
    FAR_FUNCTION_MORE -- "Ya" --> FAR_FUNCTION
    FAR_FUNCTION_MORE -- "Tidak" --> FAR_ASSET[Tambah struktur dan nilai aset]

    FAR_ASSET --> FAR_ASSET_DATA[Input kategori aset, nilai buku bersih, dan keterangan]
    FAR_ASSET_DATA --> FAR_ASSET_MORE{"Ada kategori aset lain?"}
    FAR_ASSET_MORE -- "Ya" --> FAR_ASSET
    FAR_ASSET_MORE -- "Tidak" --> FAR_RISK[Tambah risiko yang ditanggung]

    FAR_RISK --> FAR_RISK_DATA[Input risiko dan deskripsi penanganan]
    FAR_RISK_DATA --> FAR_RISK_MORE{"Ada risiko lain?"}
    FAR_RISK_MORE -- "Ya" --> FAR_RISK
    FAR_RISK_MORE -- "Tidak" --> FAR_SELECTION[Input metode terpilih, pihak dianalisis, indikator laba, dan alasan]
    FAR_SELECTION --> FAR_EVALUATION[Evaluasi lima metode TP]
    FAR_EVALUATION --> FAR_EVAL_DATA[Untuk setiap metode: input metode, status, dan keterangan atau alasan]
    FAR_EVAL_DATA --> FAR_CONSISTENCY[Periksa konsistensi metode dengan transaksi, fungsi, aset, dan risiko]
    FAR_CONSISTENCY --> FAR_VALID{"Analisis FAR konsisten?"}
    FAR_VALID -- "Tidak" --> FAR_SELECTION
    FAR_VALID -- "Ya" --> FAR_SAVE[Simpan Analisis FAR]
    FAR_SAVE --> FAR_NEXT([Lanjut ke Data Pembanding])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class FAR_START,FAR_NEXT terminal;
    class FAR_FUNCTION,FAR_FUNCTION_DATA,FAR_ASSET,FAR_ASSET_DATA,FAR_RISK,FAR_RISK_DATA,FAR_SELECTION,FAR_EVALUATION,FAR_EVAL_DATA,FAR_CONSISTENCY,FAR_SAVE process;
    class FAR_FUNCTION_MORE,FAR_ASSET_MORE,FAR_RISK_MORE,FAR_VALID decision;
```

## 8. Tahap 9 — Data Pembanding dan Seleksi (`UI-05`)

```mermaid
flowchart TD
    C_START([Mulai Data Pembanding]) --> C_UPLOAD[Upload PDF benchmarking atau TP Catalyst]
    C_UPLOAD --> C_VALIDATE{"File valid dan aman?"}
    C_VALIDATE -- "Tidak" --> C_REJECT[Tolak atau karantina file]
    C_REJECT --> C_UPLOAD
    C_VALIDATE -- "Ya" --> C_STRATEGY[Susun strategi pencarian pembanding]
    C_STRATEGY --> C_STEPS[Catat langkah dan kriteria pencarian]
    C_STEPS --> C_RESULT[Catat hasil perusahaan yang ditemukan]
    C_RESULT --> C_PLI{"Pilih satu indikator laba"}

    C_PLI -- "CPM" --> C_CPM[Gunakan Cost Plus Mark-up]
    C_PLI -- "OM" --> C_OM[Gunakan Operating Margin]
    C_PLI -- "NCP" --> C_NCP[Gunakan Net Cost Plus]
    C_CPM --> C_FORMULA[Hitung PLI yang dipilih]
    C_OM --> C_FORMULA
    C_NCP --> C_FORMULA
    C_FORMULA --> C_SCREEN[Seleksi perusahaan pembanding]
    C_SCREEN --> C_FINAL[Susun pembanding final terpilih]
    C_FINAL --> C_FINAL_DATA[Isi nama perusahaan, negara, COGS, GP, OP, dan PLI]
    C_FINAL_DATA --> C_RANGE[Hitung minimum, Q1, median, Q2, Q3, dan maksimum]
    C_RANGE --> C_REVIEW[Review rentang kewajaran interkuartil]
    C_REVIEW --> C_VALID{"Pembanding dan rentang valid?"}
    C_VALID -- "Tidak" --> C_STRATEGY
    C_VALID -- "Ya" --> C_SAVE[Simpan pembanding final dan rentang]
    C_SAVE --> C_NEXT([Lanjut ke Review and Generate])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class C_START,C_NEXT terminal;
    class C_UPLOAD,C_REJECT,C_STRATEGY,C_STEPS,C_RESULT,C_CPM,C_OM,C_NCP,C_FORMULA,C_SCREEN,C_FINAL,C_FINAL_DATA,C_RANGE,C_REVIEW,C_SAVE process;
    class C_VALIDATE,C_PLI,C_VALID decision;
```

User Induk hanya dapat memilih satu PLI—CPM, OM, atau NCP—untuk satu proses analisis pembanding.

## 9. Penyimpanan Draft, Progress, dan Validasi Antarbagian

```mermaid
flowchart TD
    S_INPUT([User mengubah field atau tabel]) --> S_LOCAL[Validasi field pada bagian aktif]
    S_LOCAL --> S_FIELD_VALID{"Input lokal valid?"}
    S_FIELD_VALID -- "Tidak" --> S_ERROR[Tampilkan pesan pada field atau baris terkait]
    S_ERROR --> S_INPUT
    S_FIELD_VALID -- "Ya" --> S_SAVE[Simpan perubahan dan waktu pembaruan]
    S_SAVE --> S_PROGRESS[Perbarui status progress bagian]
    S_PROGRESS --> S_NEXT{"Pindah ke bagian lain?"}
    S_NEXT -- "Ya" --> S_SECTION[Muat bagian berikutnya tanpa kehilangan data]
    S_SECTION --> S_INPUT
    S_NEXT -- "Tidak" --> S_LEAVE{"Keluar dari dokumen?"}
    S_LEAVE -- "Tidak" --> S_INPUT
    S_LEAVE -- "Ya" --> S_DRAFT[Simpan status Draft atau In Progress]
    S_DRAFT --> S_HISTORY[Perbarui history dan waktu terakhir diperbarui]
    S_HISTORY --> S_END([Dapat dilanjutkan kemudian])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class S_INPUT,S_END terminal;
    class S_LOCAL,S_ERROR,S_SAVE,S_PROGRESS,S_SECTION,S_DRAFT,S_HISTORY process;
    class S_FIELD_VALID,S_NEXT,S_LEAVE decision;
```

## 10. Kriteria Siap ke Document Generation

Dokumen dapat diteruskan ke [02-Document-Generation.md](./02-Document-Generation.md) setelah:

1. Identitas entitas dan tahun pajak valid.
2. Profil perusahaan lengkap.
3. Struktur kepemilikan dan manajemen valid.
4. Uraian bisnis tersedia; jika Skema AI digunakan, hanya hasil yang telah diterima yang dimasukkan ke draft Local File.
5. Ringkasan laporan keuangan tersedia.
6. Pihak afiliasi, pesaing, dan gambaran industri telah disimpan.
7. Transaksi afiliasi dan independen valid serta summary terbentuk.
8. Analisis FAR dan evaluasi metode TP selesai.
9. Satu PLI telah dipilih.
10. Pembanding final dan rentang interkuartil tersedia.

Jika salah satu syarat belum terpenuhi, sistem menampilkan bagian yang belum lengkap dan mempertahankan dokumen sebagai `Draft` atau `In Progress`.

## 11. Status Dokumen

| Status | Pemicu | Hak User Induk |
| --- | --- | --- |
| `Draft` | Dokumen baru berhasil dibuat | Mengisi seluruh bagian |
| `In Progress` | Minimal satu bagian telah disimpan | Melanjutkan, memperbaiki, atau kembali ke bagian sebelumnya |
| `Ready to Generate` | Seluruh validasi minimum berhasil | Membuka preview dan proses generate |
| `Generated` | File draft Local File berhasil dibuat | Membuka, mengunduh, atau membuat versi berikutnya |

## 12. Aturan Penting

1. Dokumen dan seluruh perubahan merupakan milik organisasi.
2. User Induk dapat melihat dan mengubah dokumen buatannya, dokumen Karyawan yang dibagikan kepadanya, atau dokumen yang diambil alih setelah membership pembuat dicabut. Karyawan dapat mengakses dokumen buatannya atau dokumen dengan penugasan aktif.
3. Sistem tidak memverifikasi kebenaran substantif seluruh data; pembuat dan penyunting bertanggung jawab atas isi serta hak penggunaan file.
4. File upload harus melewati validasi tipe, ukuran, ekstensi, dan malware scanning.
5. Data dari file tidak boleh menggantikan data tersimpan sebelum validasi berhasil.
6. Perubahan pada transaksi, metode, PLI, atau pembanding dapat membuat hasil analisis sebelumnya perlu dihitung atau ditinjau ulang.
7. AI hanya membantu membuat skema berdasarkan uraian pengguna dan tidak memberikan opini pajak.
8. Sistem menyimpan audit aktivitas penting tanpa memasukkan isi lengkap dokumen atau data finansial mentah ke log.

## 13. Referensi

- [Business Flow Overview](./00-Business-Flow-Overview.md).
- [Document Generation](./02-Document-Generation.md).
- [Use Case User Induk](../02-Use-Case/Organization-Owner.md): `UI-04`, `UI-05`, dan `UI-06`.
- [Functional Requirements](../01-Requirements/Functional-Requirements.md).
- [Non-Functional Requirements](../01-Requirements/Non-Functional-Requirements.md).

---

## Ekstensi Business Flow Layanan Konsultan

Pengelolaan Local File inti telah dijelaskan lengkap pada bagian sebelumnya. Ekstensi berikut menjelaskan akses Konsultan melalui working copy.

### Perbaikan Setelah Review

1. Pengguna membuka summary komentar per bab.
2. Pengguna membuka Local File asli dan melakukan perbaikan.
3. Pengguna menandai komentar `ADDRESSED_BY_CLIENT` bila sudah ditindaklanjuti.
4. Pengguna menjalankan validasi dan generate document version baru.
5. Version baru dikirim sebagai `consultation_round` berikutnya.

### Working Copy Penyusunan Penuh

- Working copy dibuat dari snapshot Local File dan tidak menggantikan data asli.
- Konsultan hanya mengubah bagian yang termasuk scope layanan.
- Setiap perubahan menghasilkan change set, actor, waktu, before hash, dan after hash.
- User Induk dapat menerima, meminta revisi, atau menolak delivery.
- Promotion dilakukan transactionally dan menghasilkan Local File/document version baru.
- Karyawan yang mengajukan Penyusunan Penuh dan masih ditugaskan dengan hak ubah dapat meninjau delivery serta menyetujui hasil. Jika haknya dicabut, User Induk dapat mengambil alih keputusan dengan audit; Konsultan tidak mengubah Local File utama secara langsung.
