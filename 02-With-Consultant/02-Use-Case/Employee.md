# Use Case Karyawan — TP Docs (With Consultant)


## 1. Definisi Aktor Karyawan

**Karyawan** adalah pengguna tambahan yang menerima link undangan dari User Induk dan menggunakan akun pribadi untuk bergabung ke perusahaan. Jika belum memiliki akun, calon Karyawan melakukan registrasi dan verifikasi email terlebih dahulu melalui alur undangan. Karyawan memperoleh fitur sesuai langganan perusahaan.

## 2. Daftar Use Case — Karyawan

Selain menerima undangan dan melihat akses serta kuota organisasi, Karyawan aktif dapat membuat dan men-generate Local File privat dengan kuota bersama, atau mengerjakan dokumen buatan User Induk yang ditugaskan. Karyawan dapat membayar request konsultasi miliknya sendiri, tetapi tidak dapat membeli paket atau add-on organisasi, mengelola pembayaran organisasi, mengundang anggota, atau mengubah penugasan.

### 2.1 Visual Aktivitas Karyawan

```mermaid
flowchart TB
    A([Karyawan])

    UIK01(["UK-01<br/>Menerima Undangan Langganan Perusahaan"])
    UIK02(["UK-02<br/>Melihat Status Akses Langganan"])

    A --> UIK01
    A --> UIK02
    A --> UIK03(["UK-03<br/>Mengisi Local File yang Ditugaskan"])
    A --> UIK04(["UK-04<br/>Membuat dan Generate Local File Sendiri"])
    A --> UIK05(["UK-05<br/>Melihat Dokumen dan History Versi yang Dapat Diakses"])

    OPEN["UK-01.1<br/>Membuka Link Undangan"]
    CHECK["UK-01.2<br/>Memeriksa Kepemilikan Akun"]
    REGISTER["UK-01.2.E1<br/>Registrasi & Verifikasi Akun Baru"]
    LOGIN["UK-01.3<br/>Login atau Melanjutkan dengan Akun"]
    VALIDATE["UK-01.4<br/>Memvalidasi Email Undangan"]
    ACCEPT["UK-01.5<br/>Menyetujui Bergabung"]
    ACTIVATE["UK-01.6<br/>Mengaktifkan Akses Langganan"]
    REJECT["UK-01.E1<br/>Menolak Undangan"]
    INVALID["UK-01.E2<br/>Undangan Tidak Valid / Kedaluwarsa"]

    UIK01 -. "&lt;&lt;include&gt;&gt;" .-> OPEN
    UIK01 -. "&lt;&lt;include&gt;&gt;" .-> CHECK
    CHECK -. "&lt;&lt;extend&gt;&gt;" .-> REGISTER
    UIK01 -. "&lt;&lt;include&gt;&gt;" .-> LOGIN
    UIK01 -. "&lt;&lt;include&gt;&gt;" .-> VALIDATE
    UIK01 -. "&lt;&lt;include&gt;&gt;" .-> ACCEPT
    UIK01 -. "&lt;&lt;include&gt;&gt;" .-> ACTIVATE
    UIK01 -. "&lt;&lt;extend&gt;&gt;" .-> REJECT
    UIK01 -. "&lt;&lt;extend&gt;&gt;" .-> INVALID
    ACTIVATE ~~~ REJECT
    ACTIVATE ~~~ INVALID

    COMPANY["UK-02.1<br/>Melihat Perusahaan Induk"]
    PACKAGE["UK-02.2<br/>Melihat Paket yang Diwarisi"]
    STATUS["UK-02.3<br/>Melihat Status Membership"]
    PERIOD["UK-02.4<br/>Melihat Masa Berlaku Akses"]
    DOCQUOTA["UK-02.5<br/>Melihat Kuota Dokumen Organisasi"]
    EXPIRED["UK-02.E1<br/>Akses Langganan Berakhir"]

    UIK02 -. "&lt;&lt;include&gt;&gt;" .-> COMPANY
    UIK02 -. "&lt;&lt;include&gt;&gt;" .-> PACKAGE
    UIK02 -. "&lt;&lt;include&gt;&gt;" .-> STATUS
    UIK02 -. "&lt;&lt;include&gt;&gt;" .-> PERIOD
    UIK02 -. "&lt;&lt;include&gt;&gt;" .-> DOCQUOTA
    UIK02 -. "&lt;&lt;extend&gt;&gt;" .-> EXPIRED
    PERIOD ~~~ EXPIRED
```

**Legenda:** garis penuh menunjukkan aktivitas yang dimulai langsung oleh Karyawan. Panah `<<include>>` menunjukkan tahapan wajib, sedangkan `<<extend>>` menunjukkan kondisi atau pilihan tambahan.

| ID         | Tingkat / Induk           | Use Case                                   | Deskripsi / Aktivitas                                                                                  | Include (`<<include>>`) | Extend (`<<extend>>`) |
| ---------- | ------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------ | ----------------------- | --------------------- |
| **UK-01**  | **`[AKTOR DIRECT]`**      | **Menerima Undangan Langganan Perusahaan** | [Aktivitas menerima akses paket perusahaan melalui link undangan yang dikirim ke email calon Karyawan] | UK-01.1 s/d UK-01.6     | UK-01.E1, UK-01.E2    |
| UK-01.1    | └── _sub dari UK-01_      | Membuka Link Undangan                      | [Aktivitas membuka tautan unik yang dikirim User Induk melalui sistem]                                 | [-]                     | [-]                   |
| UK-01.2    | └── _sub dari UK-01_      | Memeriksa Kepemilikan Akun                 | [Aktivitas sistem memeriksa apakah email tujuan sudah memiliki akun]                                   | [-]                     | UK-01.2.E1            |
| UK-01.2.E1 | └── _extend dari UK-01.2_ | Registrasi dan Verifikasi Akun Baru        | [Aktivitas calon Karyawan membuat akun dan memverifikasi email jika belum terdaftar]                   | [-]                     | [-]                   |
| UK-01.3    | └── _sub dari UK-01_      | Login atau Melanjutkan dengan Akun         | [Aktivitas autentikasi akun tanpa kehilangan konteks link undangan]                                    | [-]                     | [-]                   |
| UK-01.4    | └── _sub dari UK-01_      | Memvalidasi Email Undangan                 | [Aktivitas memastikan email akun sama dengan email tujuan undangan]                                    | [-]                     | [-]                   |
| UK-01.5    | └── _sub dari UK-01_      | Menyetujui Bergabung                       | [Aktivitas menerima membership perusahaan]                                                             | [-]                     | [-]                   |
| UK-01.6    | └── _sub dari UK-01_      | Mengaktifkan Akses Langganan               | [Aktivitas memperoleh fitur sesuai paket dan sisa masa aktif perusahaan]                               | [-]                     | [-]                   |
| UK-01.E1   | └── _extend dari UK-01_   | Menolak Undangan                           | [Aktivitas menolak bergabung ke perusahaan]                                                            | [-]                     | [-]                   |
| UK-01.E2   | └── _extend dari UK-01_   | Undangan Tidak Valid atau Kedaluwarsa      | [Aktivitas menolak token salah, digunakan, dibatalkan, atau kedaluwarsa]                               | [-]                     | [-]                   |
| **UK-02**  | **`[AKTOR DIRECT]`**      | **Melihat Status Akses Langganan**         | [Aktivitas melihat sumber, masa akses, dan kuota dokumen agregat]                                       | UK-02.1 s/d UK-02.5     | UK-02.E1              |
| UK-02.1    | └── _sub dari UK-02_      | Melihat Perusahaan Induk                   | [Aktivitas menampilkan perusahaan tempat Karyawan menjadi anggota]                                     | [-]                     | [-]                   |
| UK-02.2    | └── _sub dari UK-02_      | Melihat Paket yang Diwarisi                | [Aktivitas menampilkan paket perusahaan yang berlaku bagi Karyawan]                                    | [-]                     | [-]                   |
| UK-02.3    | └── _sub dari UK-02_      | Melihat Status Membership                  | [Aktivitas menampilkan status aktif, tertunda, atau dicabut]                                           | [-]                     | [-]                   |
| UK-02.4    | └── _sub dari UK-02_      | Melihat Masa Berlaku Akses                 | [Aktivitas menampilkan tanggal berakhir berdasarkan langganan perusahaan]                              | [-]                     | [-]                   |
| UK-02.5    | └── _sub dari UK-02_      | Melihat Kuota Dokumen Organisasi           | [Aktivitas melihat total, terpakai, dan sisa kuota bersama tanpa rincian pembayaran organisasi]        | [-]                     | [-]                   |
| UK-02.E1   | └── _extend dari UK-02_   | Akses Langganan Berakhir                   | [Aktivitas mencabut fitur berbayar saat paket perusahaan berakhir]                                     | [-]                     | [-]                   |

---

## 3. Local File Buatan Sendiri dan Ditugaskan

| ID | Use Case | Aktivitas / hasil |
| --- | --- | --- |
| **UK-03** | Mengisi Local File yang ditugaskan | Daftar “Ditugaskan kepada saya” hanya memuat dokumen buatan User Induk dengan penugasan aktif. Karyawan mengisi bagian 01–10 bebas urutan, memeriksa bagian 11, dan dapat melakukan generate setelah validasi berhasil. |
| UK-03.E1 | Penugasan dicabut | Akses draft, versi, preview, dan unduhan langsung berhenti tanpa menghapus perubahan sebelumnya. |
| **UK-04** | Membuat Local File sendiri | Sistem memvalidasi membership aktif, paket, kuota organisasi, serta identitas dokumen sebelum membuat draft privat dan memakai satu kuota. |
| UK-04.1 | Melihat “Dibuat oleh saya” | Karyawan membuka dokumen buatannya, progres, hasil generate, dan History Versi selama ia berhak. |
| UK-04.2 | Berbagi dengan User Induk | Hanya pembuat aktif yang dapat memberi atau mencabut akses lihat dan ubah User Induk; perubahan langsung berlaku dan diaudit. User Induk tidak dapat menugaskan dokumen ini kepada Karyawan lain. |
| UK-04.3 | Review dan generate | Pembuat mengisi bagian 01–10 bebas urutan dan melakukan generate setelah bagian 11 valid. Generate ulang tidak memakai kuota pembuatan baru. |
| UK-04.E1 | Kuota habis | Pembuatan ditolak tanpa mengurangi kuota; Karyawan meminta User Induk mengelola kapasitas, bukan membuka checkout paket. |
| **UK-05** | Melihat riwayat dokumen | Karyawan melihat dan membandingkan dua versi dari dokumen buatannya atau yang ditugaskan secara baca saja. |

Dokumen buatan Karyawan tetap milik organisasi, tetapi dimulai privat. User Induk hanya melihat pemakaian kuotanya secara agregat tanpa identitas, isi, atau hasil dokumen. Bila membership pembuat dicabut, sistem mengalihkan akses kepada User Induk dengan jejak audit. Administrator platform tidak memperoleh akses konten. Karyawan pembuat dan penerima penugasan dapat mengajukan konsultasi atas dokumen yang dapat mereka ubah.

## Layanan Konsultasi pada Karyawan

Karyawan yang berhak membeli jasa untuk dokumen buatannya atau yang ditugaskan melihat harga normal atau satu diskon event aktif yang sesuai sebelum checkout. Tanpa event, harga normal berlaku. Karyawan tidak dapat membuat atau mengubah event.


```mermaid
flowchart TB
    EMP([Karyawan]) --> K01["UK-C01<br/>Melihat Katalog"]
    EMP --> K02["UK-C02<br/>Membuat Request Dokumen yang Dapat Diakses"]
    EMP --> K03["UK-C03<br/>Membayar Request Sendiri"]
    EMP --> K04["UK-C04<br/>Melihat Status dan ETA"]
    EMP --> K05["UK-C05<br/>Melihat dan Membalas Komentar"]
    EMP --> K06["UK-C06<br/>Resubmit Dokumen yang Dapat Diakses"]
    EMP --> K07["UK-C07<br/>Meninjau dan Menyetujui Hasil Penyusunan Penuh"]
    K02 -. "&lt;&lt;include&gt;&gt;" .-> ASSIGN["Validasi Pembuat atau Document Assignment"]
    K03 -. "&lt;&lt;include&gt;&gt;" .-> OWN["Batasi Payment Milik Sendiri"]
    K06 -. "&lt;&lt;include&gt;&gt;" .-> QUOTA["Validasi Revision Quota"]
    K07 -. "&lt;&lt;include&gt;&gt;" .-> EDIT["Validasi Pemohon dan Hak Ubah Local File"]
```

| ID | Aktivitas | Batasan |
| --- | --- | --- |
| UK-C01 | Melihat katalog | Hanya layanan aktif |
| UK-C02 | Membuat request | Hanya document version dari dokumen buatan sendiri atau yang ditugaskan aktif |
| UK-C03 | Membayar konsultasi | Dapat checkout tetapi tidak mengelola subscription/add-on organisasi |
| UK-C04 | Melihat status | Hanya request yang dibuat atau dokumen buatannya/yang masih ditugaskan |
| UK-C05 | Melihat komentar | Reply dan tandai addressed; tidak menghapus komentar Konsultan |
| UK-C06 | Resubmit | Hanya jika hak ubah sebagai pembuat/anggota ditugaskan dan revision quota masih aktif |
| UK-C07 | Meninjau hasil Penyusunan Penuh | Hanya request yang diajukan sendiri dan Local File yang masih dapat diubah sebagai pembuat atau anggota ditugaskan; dapat meminta revisi atau menyetujui hasil untuk diterapkan sebagai versi baru |

Karyawan tidak dapat melihat seluruh billing konsultasi organisasi, mengubah katalog, atau menerima quotation atas nama organisasi lain. Karyawan pemohon dapat menyetujui hasil Penyusunan Penuh jika masih memiliki hak ubah pada Local File tersebut. Jika aksesnya dicabut, keputusan dapat diambil alih User Induk dan dicatat dalam audit; Karyawan dan Konsultan tidak menjalankan promotion secara langsung.

Karyawan dapat membuka History Versi pada Local File buatannya atau yang ditugaskan dan membandingkan dua versi hasil generate dari dokumen tersebut secara baca saja. Karyawan tanpa penugasan tidak dapat melihat dokumen buatan User Induk.
