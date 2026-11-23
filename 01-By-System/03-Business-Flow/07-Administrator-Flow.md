# Administrator Business Flow — TP Docs (By System)
## Mengelola Event Diskon

Admin membuat draft, memvalidasi target, nilai, periode, dan batas, lalu mengaktifkannya. Dashboard billing menampilkan status dan pemakaian. Penonaktifan hanya memengaruhi transaksi baru; snapshot checkout dan quotation yang telah terbit tidak diubah. Semua perubahan masuk audit log.


> Pemeliharaan file mencakup status malware, karantina, retensi, serta bukti penghapusan. Backup dan uji restore PostgreSQL harus mencakup metadata maupun chunk biner dan memverifikasi checksum file hasil pemulihan.

## 1. Tujuan dan Batas Kewenangan

Dokumen ini menjelaskan aktivitas Administrator dalam memelihara operasional platform TP Docs, paket dan billing, akun dan organisasi, keamanan, integrasi, serta siklus data. Flow mengikuti `UA-01` sampai `UA-09` pada use case Administrator dan memperinci overview pada `00-Business-Flow-Overview.md`.

Administrator tidak diperbolehkan:

- Mengundang, menghapus, atau menugaskan Karyawan perusahaan.
- Mengubah isi Local File atau hasil analisis Transfer Pricing.
- Menetapkan atau melihat password pengguna.
- Menyamar sebagai pengguna.
- Membaca dokumen pelanggan tanpa kewenangan yang sah.
- Mengubah billing atau kuota tanpa alasan, autentikasi ulang, dan persetujuan yang diwajibkan.
- Menonaktifkan atau mengubah audit log.

Setiap Administrator menggunakan akun pribadi, MFA, dan audit log. Aktivitas berisiko tinggi memerlukan autentikasi ulang dan persetujuan Administrator kedua.

## 2. Flow Utama Portal Administrator

```mermaid
flowchart TD
    ADM_START([Mulai]) --> ADM_LOGIN[Login Portal Administrator]
    ADM_LOGIN --> ADM_AUTH{"Kredensial dan MFA valid?"}
    ADM_AUTH -- "Tidak" --> ADM_RECOVERY{"Lupa password?"}
    ADM_RECOVERY -- "Ya" --> ADM_RESET[Jalankan pemulihan password mandiri]
    ADM_RESET --> ADM_LOGIN
    ADM_RECOVERY -- "Tidak" --> ADM_RETRY[Tampilkan kegagalan autentikasi]
    ADM_RETRY --> ADM_LOGIN
    ADM_AUTH -- "Ya" --> ADM_DASHBOARD[Tampilkan Dashboard Administrator]
    ADM_DASHBOARD --> ADM_TASK{"Pilih aktivitas"}
    ADM_TASK -- "Status operasional" --> ADM_OPERATION[UA-02 Memantau Status Operasional]
    ADM_TASK -- "Maintenance" --> ADM_MAINTENANCE[UA-03 Mengelola Maintenance Mode]
    ADM_TASK -- "Pengaturan" --> ADM_SETTING[UA-04 Mengelola Pengaturan Dasar]
    ADM_TASK -- "Paket dan billing" --> ADM_BILLING[UA-05 Mengelola Paket, Langganan, dan Billing]
    ADM_TASK -- "Akun dan Custom" --> ADM_ACCOUNT[UA-06 Mengelola Akun, Organisasi, dan Enterprise/Custom]
    ADM_TASK -- "Keamanan" --> ADM_SECURITY[UA-07 Mengelola Keamanan dan Insiden]
    ADM_TASK -- "File dan integrasi" --> ADM_INTEGRATION[UA-08 Mengelola File dan Integrasi]
    ADM_TASK -- "Siklus data" --> ADM_DATA[UA-09 Mengelola Backup, Retensi, dan Penghapusan]
    ADM_OPERATION --> ADM_DASHBOARD
    ADM_MAINTENANCE --> ADM_DASHBOARD
    ADM_SETTING --> ADM_DASHBOARD
    ADM_BILLING --> ADM_DASHBOARD
    ADM_ACCOUNT --> ADM_DASHBOARD
    ADM_SECURITY --> ADM_DASHBOARD
    ADM_INTEGRATION --> ADM_DASHBOARD
    ADM_DATA --> ADM_DASHBOARD
    ADM_DASHBOARD --> ADM_LOGOUT[Logout dan akhiri sesi]
    ADM_LOGOUT --> ADM_END([Selesai])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class ADM_START,ADM_END startend;
    class ADM_LOGIN,ADM_RESET,ADM_DASHBOARD,ADM_OPERATION,ADM_MAINTENANCE,ADM_SETTING,ADM_BILLING,ADM_ACCOUNT,ADM_SECURITY,ADM_INTEGRATION,ADM_DATA,ADM_LOGOUT process;
    class ADM_AUTH,ADM_RECOVERY,ADM_TASK decision;
    class ADM_RETRY exception;
```

## 3. UA-01 — Mengakses Portal dan Dashboard Administrator

### 3.1 Login, MFA, dan Lupa Password

```mermaid
flowchart TD
    LOGIN_START([Administrator membuka portal terpisah]) --> LOGIN_INPUT[Input email dan password Administrator]
    LOGIN_INPUT --> LOGIN_VALID{"Kredensial valid dan akun aktif?"}
    LOGIN_VALID -- "Tidak" --> LOGIN_FORGOT{"Pilih Lupa Password?"}
    LOGIN_FORGOT -- "Tidak" --> LOGIN_FAIL[Catat kegagalan dan terapkan pembatasan login]
    LOGIN_FAIL --> LOGIN_INPUT
    LOGIN_FORGOT -- "Ya" --> LOGIN_EMAIL[Input email Administrator]
    LOGIN_EMAIL --> LOGIN_ACCOUNT{"Akun Administrator valid?"}
    LOGIN_ACCOUNT -- "Tidak" --> LOGIN_NEUTRAL[Tampilkan respons netral]
    LOGIN_NEUTRAL --> LOGIN_INPUT
    LOGIN_ACCOUNT -- "Ya" --> LOGIN_LINK[Kirim tautan reset dengan masa berlaku terbatas]
    LOGIN_LINK --> LOGIN_OPEN[Administrator membuka tautan]
    LOGIN_OPEN --> LOGIN_TOKEN{"Token valid dan belum kedaluwarsa?"}
    LOGIN_TOKEN -- "Tidak" --> LOGIN_EXPIRED[Tampilkan tautan tidak valid atau kedaluwarsa]
    LOGIN_TOKEN -- "Ya" --> LOGIN_NEW[Buat dan konfirmasi password baru]
    LOGIN_NEW --> LOGIN_SAVE[Simpan hash password dan cabut sesi lama]
    LOGIN_SAVE --> LOGIN_INPUT
    LOGIN_VALID -- "Ya" --> LOGIN_MFA[Input kode MFA]
    LOGIN_MFA --> LOGIN_MFA_VALID{"MFA valid?"}
    LOGIN_MFA_VALID -- "Tidak" --> LOGIN_MFA_FAIL[Catat kegagalan MFA]
    LOGIN_MFA_FAIL --> LOGIN_MFA
    LOGIN_MFA_VALID -- "Ya" --> LOGIN_SESSION[Buat sesi Administrator dan catat login]
    LOGIN_SESSION --> LOGIN_DASHBOARD([Masuk Dashboard])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class LOGIN_START,LOGIN_DASHBOARD startend;
    class LOGIN_INPUT,LOGIN_EMAIL,LOGIN_LINK,LOGIN_OPEN,LOGIN_NEW,LOGIN_SAVE,LOGIN_MFA,LOGIN_SESSION process;
    class LOGIN_VALID,LOGIN_FORGOT,LOGIN_ACCOUNT,LOGIN_TOKEN,LOGIN_MFA_VALID decision;
    class LOGIN_FAIL,LOGIN_NEUTRAL,LOGIN_EXPIRED,LOGIN_MFA_FAIL exception;
```

### 3.2 Dashboard dan Laporan Operasional

```mermaid
flowchart TD
    DASH_START([Login berhasil]) --> DASH_LOAD[Ambil statistik operasional terotorisasi]
    DASH_LOAD --> DASH_USER[Tampilkan statistik pengguna]
    DASH_USER --> DASH_ORG[Tampilkan statistik organisasi]
    DASH_ORG --> DASH_SUB[Tampilkan langganan aktif, tidak aktif, dan berakhir]
    DASH_SUB --> DASH_PLAN[Tampilkan distribusi paket]
    DASH_PLAN --> DASH_QUOTA[Tampilkan penggunaan kuota dan add-on]
    DASH_QUOTA --> DASH_PAY[Tampilkan status pembayaran]
    DASH_PAY --> DASH_REVENUE[Tampilkan pendapatan agregat]
    DASH_REVENUE --> DASH_CUSTOM[Tampilkan jumlah pengajuan Enterprise/Custom per status]
    DASH_CUSTOM --> DASH_ACTION{"Ekspor laporan?"}
    DASH_ACTION -- "Tidak" --> DASH_END([Dashboard siap digunakan])
    DASH_ACTION -- "Ya" --> DASH_FILTER[Validasi filter dan kewenangan ekspor]
    DASH_FILTER --> DASH_EXPORT[Ekspor laporan tanpa isi dokumen atau data rahasia mentah]
    DASH_EXPORT --> DASH_LOG[Catat ekspor pada audit log]
    DASH_LOG --> DASH_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    class DASH_START,DASH_END startend;
    class DASH_LOAD,DASH_USER,DASH_ORG,DASH_SUB,DASH_PLAN,DASH_QUOTA,DASH_PAY,DASH_REVENUE,DASH_CUSTOM,DASH_FILTER,DASH_EXPORT,DASH_LOG process;
    class DASH_ACTION decision;
```

Dashboard tidak menampilkan password, token, NPWP lengkap, data finansial mentah pelanggan, isi Local File, CPU/RAM, antrean internal, atau SLA teknis.

## 4. UA-02 — Memantau Status Operasional

```mermaid
flowchart TD
    OPS_START([Administrator membuka Status Operasional]) --> OPS_APP[Tampilkan status aplikasi dan API]
    OPS_APP --> OPS_INTEGRATION[Tampilkan ringkasan email, pembayaran, malware scanner, storage, dan AI]
    OPS_INTEGRATION --> OPS_ALERT{"Ada gangguan atau alert?"}
    OPS_ALERT -- "Tidak" --> OPS_HEALTHY[Tampilkan seluruh layanan yang tersedia]
    OPS_ALERT -- "Ya" --> OPS_DETAIL[Tampilkan layanan, waktu, tingkat dampak, dan pesan aman]
    OPS_DETAIL --> OPS_REVIEW[Administrator meninjau gangguan]
    OPS_REVIEW --> OPS_MODE{"Perlu maintenance mode?"}
    OPS_MODE -- "Ya" --> OPS_MAINTENANCE[Lanjut ke UA-03]
    OPS_MODE -- "Tidak" --> OPS_MONITOR[Pantau perubahan status]
    OPS_HEALTHY --> OPS_END([Kembali ke Dashboard])
    OPS_MAINTENANCE --> OPS_END
    OPS_MONITOR --> OPS_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    class OPS_START,OPS_END startend;
    class OPS_APP,OPS_INTEGRATION,OPS_HEALTHY,OPS_DETAIL,OPS_REVIEW,OPS_MAINTENANCE,OPS_MONITOR process;
    class OPS_ALERT,OPS_MODE decision;
```

## 5. UA-03 — Mengelola Maintenance Mode

```mermaid
flowchart TD
    MAINT_START([Administrator membuka Maintenance Mode]) --> MAINT_STATUS[Tampilkan status mode saat ini]
    MAINT_STATUS --> MAINT_ACTION{"Tindakan"}
    MAINT_ACTION -- "Aktifkan" --> MAINT_NOTICE[Susun judul dan isi pemberitahuan maintenance]
    MAINT_NOTICE --> MAINT_PREVIEW[Preview pesan yang akan ditampilkan]
    MAINT_PREVIEW --> MAINT_CONFIRM[Autentikasi ulang dan konfirmasi aktivasi]
    MAINT_CONFIRM --> MAINT_ENABLE[Aktifkan pemberitahuan dan maintenance mode]
    MAINT_ENABLE --> MAINT_ENABLE_OK{"Aktivasi berhasil?"}
    MAINT_ENABLE_OK -- "Tidak" --> MAINT_FAILED[Tampilkan kegagalan tanpa status palsu]
    MAINT_ENABLE_OK -- "Ya" --> MAINT_ACTIVE[Portal pelanggan menampilkan pemberitahuan maintenance]
    MAINT_ACTION -- "Nonaktifkan" --> MAINT_DISABLE[Autentikasi ulang dan nonaktifkan maintenance mode]
    MAINT_DISABLE --> MAINT_DISABLE_OK{"Penonaktifan berhasil?"}
    MAINT_DISABLE_OK -- "Tidak" --> MAINT_FAILED
    MAINT_DISABLE_OK -- "Ya" --> MAINT_NORMAL[Pulihkan akses normal]
    MAINT_ACTIVE --> MAINT_LOG[Catat pelaksana, waktu, dan perubahan]
    MAINT_NORMAL --> MAINT_LOG
    MAINT_LOG --> MAINT_END([Kembali ke Dashboard])
    MAINT_FAILED --> MAINT_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class MAINT_START,MAINT_END startend;
    class MAINT_STATUS,MAINT_NOTICE,MAINT_PREVIEW,MAINT_CONFIRM,MAINT_ENABLE,MAINT_ACTIVE,MAINT_DISABLE,MAINT_NORMAL,MAINT_LOG process;
    class MAINT_ACTION,MAINT_ENABLE_OK,MAINT_DISABLE_OK decision;
    class MAINT_FAILED exception;
```

Flow ini tidak mencakup penjadwalan maintenance, pemilihan layanan terdampak, maintenance darurat, atau pencatatan teknis hasil maintenance.

## 6. UA-04 — Mengelola Pengaturan Dasar Platform

```mermaid
flowchart TD
    SET_START([Administrator membuka Pengaturan Dasar]) --> SET_ACTION{"Pilih pengaturan"}
    SET_ACTION -- "Batas upload" --> SET_UPLOAD[Tampilkan ukuran maksimum dan tipe file]
    SET_UPLOAD --> SET_UPLOAD_EDIT[Ubah nilai yang diperbolehkan]
    SET_ACTION -- "Template notifikasi" --> SET_TEMPLATE[Tampilkan daftar template]
    SET_TEMPLATE --> SET_TEMPLATE_EDIT[Ubah subjek dan isi template]
    SET_UPLOAD_EDIT --> SET_VALIDATE
    SET_TEMPLATE_EDIT --> SET_VALIDATE{"Pengaturan valid?"}
    SET_VALIDATE -- "Tidak" --> SET_ERROR[Tampilkan field yang harus diperbaiki]
    SET_ERROR --> SET_ACTION
    SET_VALIDATE -- "Ya" --> SET_REAUTH[Autentikasi ulang Administrator]
    SET_REAUTH --> SET_SAVE[Simpan perubahan]
    SET_SAVE --> SET_HISTORY[Catat nilai lama, nilai baru, pelaksana, dan waktu]
    SET_HISTORY --> SET_END([Perubahan tersedia])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class SET_START,SET_END startend;
    class SET_UPLOAD,SET_UPLOAD_EDIT,SET_TEMPLATE,SET_TEMPLATE_EDIT,SET_REAUTH,SET_SAVE,SET_HISTORY process;
    class SET_ACTION,SET_VALIDATE decision;
    class SET_ERROR exception;
```

Feature flag, timeout, rate limit, konfigurasi infrastruktur, dan rollback tidak dikelola melalui flow Administrator ini.

## 7. UA-05 — Mengelola Paket, Langganan, dan Billing

### 7.1 Katalog, Kuota, Pembayaran, dan Koreksi

```mermaid
flowchart TD
    BILL_START([Administrator membuka Paket dan Billing]) --> BILL_ACTION{"Pilih aktivitas"}
    BILL_ACTION -- "Katalog" --> BILL_CATALOG[Melihat paket, harga, dan fitur]
    BILL_CATALOG --> BILL_EDIT[Ubah harga atau fitur]
    BILL_EDIT --> BILL_VALID{"Data paket valid?"}
    BILL_VALID -- "Tidak" --> BILL_ERROR[Tolak penyimpanan]
    BILL_VALID -- "Ya" --> BILL_SAVE[Simpan dan catat perubahan katalog]
    BILL_ACTION -- "Kuota" --> BILL_QUOTA[Atur kuota dokumen, kursi, dan ketentuan add-on]
    BILL_QUOTA --> BILL_QUOTA_VALID{"Nilai kuota valid?"}
    BILL_QUOTA_VALID -- "Tidak" --> BILL_ERROR
    BILL_QUOTA_VALID -- "Ya" --> BILL_SAVE
    BILL_ACTION -- "Pembayaran" --> BILL_PAYMENT[Melihat status transaksi dan referensi payment gateway]
    BILL_PAYMENT --> BILL_PAYMENT_STATE{"Gagal atau tertunda?"}
    BILL_PAYMENT_STATE -- "Ya" --> BILL_REVIEW[Catat status tanpa aktivasi hak akses]
    BILL_PAYMENT_STATE -- "Tidak" --> BILL_HISTORY[Perbarui histori pembayaran terverifikasi]
    BILL_ACTION -- "Status langganan" --> BILL_SUB[Melihat masa aktif dan histori perubahan]
    BILL_SUB --> BILL_HISTORY
    BILL_ACTION -- "Koreksi" --> BILL_REASON[Input alasan dan referensi koreksi]
    BILL_REASON --> BILL_REAUTH[Autentikasi ulang]
    BILL_REAUTH --> BILL_APPROVAL{"Administrator kedua menyetujui?"}
    BILL_APPROVAL -- "Tidak" --> BILL_REJECT[Catat koreksi ditolak]
    BILL_APPROVAL -- "Ya" --> BILL_APPLY[Terapkan koreksi paket, kuota, atau masa aktif secara terbatas]
    BILL_APPLY --> BILL_AUDIT[Catat koreksi pada audit log]
    BILL_SAVE --> BILL_END([Kembali ke daftar])
    BILL_REVIEW --> BILL_END
    BILL_HISTORY --> BILL_END
    BILL_REJECT --> BILL_END
    BILL_AUDIT --> BILL_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class BILL_START,BILL_END startend;
    class BILL_CATALOG,BILL_EDIT,BILL_SAVE,BILL_QUOTA,BILL_PAYMENT,BILL_REVIEW,BILL_HISTORY,BILL_SUB,BILL_REASON,BILL_REAUTH,BILL_APPLY,BILL_AUDIT process;
    class BILL_ACTION,BILL_VALID,BILL_QUOTA_VALID,BILL_PAYMENT_STATE,BILL_APPROVAL decision;
    class BILL_ERROR,BILL_REJECT exception;
```

### 7.2 Refund Terotorisasi

```mermaid
flowchart TD
    REF_START([Pengajuan refund berstatus Submitted]) --> REF_OPEN[Administrator membuka transaksi, alasan, dan bukti]
    REF_OPEN --> REF_REVIEW[Ubah status menjadi Under Review]
    REF_REVIEW --> REF_VALIDATE[Validasi kelayakan dan nilai refund]
    REF_VALIDATE --> REF_IMPACT[Tentukan dampak paket, masa aktif, kursi, kuota, atau add-on]
    REF_IMPACT --> REF_REAUTH[Autentikasi ulang Administrator]
    REF_REAUTH --> REF_SECOND{"Administrator kedua menyetujui?"}
    REF_SECOND -- "Tidak" --> REF_REJECT[Status Rejected tanpa perubahan akses]
    REF_SECOND -- "Ya" --> REF_APPROVED[Status Approved lalu Processing]
    REF_APPROVED --> REF_GATEWAY[Proses pengembalian melalui metode pembayaran asal]
    REF_GATEWAY --> REF_RESULT{"Pengembalian berhasil?"}
    REF_RESULT -- "Tidak" --> REF_FAILED[Status Failed tanpa penyesuaian ganda]
    REF_RESULT -- "Ya" --> REF_REFUNDED[Status Refunded]
    REF_REFUNDED --> REF_ADJUST[Sesuaikan akses tepat satu kali]
    REF_ADJUST --> REF_NOTIFY[Kirim hasil ke User Induk]
    REF_REJECT --> REF_NOTIFY
    REF_FAILED --> REF_NOTIFY
    REF_NOTIFY --> REF_END([Selesai])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class REF_START,REF_END startend;
    class REF_OPEN,REF_REVIEW,REF_VALIDATE,REF_IMPACT,REF_REAUTH,REF_APPROVED,REF_GATEWAY,REF_REFUNDED,REF_ADJUST,REF_NOTIFY process;
    class REF_SECOND,REF_RESULT decision;
    class REF_REJECT,REF_FAILED exception;
```

Status refund: `Submitted`, `Under Review`, `Approved`, `Rejected`, `Processing`, `Refunded`, dan `Failed`.

## 8. UA-06 — Mengelola Akun, Organisasi, dan Enterprise/Custom

### 8.1 Detail Akun, Pemulihan, dan Status

```mermaid
flowchart TD
    ACCOUNT_START([Administrator membuka detail akun atau organisasi]) --> ACCOUNT_VIEW[Tampilkan status akun, membership, profil organisasi, paket, dan masa aktif]
    ACCOUNT_VIEW --> ACCOUNT_ACTION{"Pilih tindakan"}
    ACCOUNT_ACTION -- "Kirim reset password" --> ACCOUNT_VERIFY[Verifikasi permintaan pemulihan]
    ACCOUNT_VERIFY --> ACCOUNT_LINK[Kirim tautan reset aman]
    ACCOUNT_ACTION -- "Reset MFA" --> ACCOUNT_IDENTITY[Verifikasi identitas pengguna]
    ACCOUNT_IDENTITY --> ACCOUNT_ID_OK{"Identitas terverifikasi?"}
    ACCOUNT_ID_OK -- "Tidak" --> ACCOUNT_REJECT[Tolak reset MFA]
    ACCOUNT_ID_OK -- "Ya" --> ACCOUNT_MFA[Cabut faktor lama dan kirim instruksi aktivasi baru]
    ACCOUNT_ACTION -- "Suspend / reaktivasi" --> ACCOUNT_REASON[Pilih alasan perubahan status]
    ACCOUNT_REASON --> ACCOUNT_REAUTH[Autentikasi ulang]
    ACCOUNT_REAUTH --> ACCOUNT_STATUS[Perbarui status akun]
    ACCOUNT_STATUS --> ACCOUNT_NOTIFY[Kirim pemberitahuan kepada pengguna]
    ACCOUNT_ACTION -- "Kunci darurat" --> ACCOUNT_EMERGENCY[Konfirmasi kondisi darurat]
    ACCOUNT_EMERGENCY --> ACCOUNT_LOCK[Kunci akun dan cabut seluruh sesi serta token]
    ACCOUNT_LOCK --> ACCOUNT_LOG[Catat alasan dan pelaksana]
    ACCOUNT_LINK --> ACCOUNT_END([Kembali ke detail])
    ACCOUNT_REJECT --> ACCOUNT_END
    ACCOUNT_MFA --> ACCOUNT_END
    ACCOUNT_NOTIFY --> ACCOUNT_END
    ACCOUNT_LOG --> ACCOUNT_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class ACCOUNT_START,ACCOUNT_END startend;
    class ACCOUNT_VIEW,ACCOUNT_VERIFY,ACCOUNT_LINK,ACCOUNT_IDENTITY,ACCOUNT_MFA,ACCOUNT_REASON,ACCOUNT_REAUTH,ACCOUNT_STATUS,ACCOUNT_NOTIFY,ACCOUNT_EMERGENCY,ACCOUNT_LOCK,ACCOUNT_LOG process;
    class ACCOUNT_ACTION,ACCOUNT_ID_OK decision;
    class ACCOUNT_REJECT exception;
```

Administrator hanya mengirim tautan reset; password baru tetap dibuat oleh pemilik akun.

### 8.2 Pengajuan Enterprise/Custom

```mermaid
flowchart TD
    CUSTOM_START([Pengajuan User Induk berstatus Submitted]) --> CUSTOM_LIST[Administrator membuka daftar dan detail pengajuan]
    CUSTOM_LIST --> CUSTOM_REVIEW[Ubah status menjadi Under Review]
    CUSTOM_REVIEW --> CUSTOM_NEEDS[Tinjau dokumen, pengguna, periode, entitas, SSO, lokasi data, dan dukungan]
    CUSTOM_NEEDS --> CUSTOM_FEASIBLE{"Kebutuhan dapat dipenuhi?"}
    CUSTOM_FEASIBLE -- "Perlu perbaikan" --> CUSTOM_REVISION[Status Revision Requested dan kirim catatan]
    CUSTOM_REVISION -. "status update" .-> CUSTOM_RESUBMIT[User Induk memperbaiki dan mengirim ulang]
    CUSTOM_RESUBMIT --> CUSTOM_REVIEW
    CUSTOM_FEASIBLE -- "Tidak" --> CUSTOM_REJECT[Status Rejected dan kirim alasan]
    CUSTOM_FEASIBLE -- "Ya" --> CUSTOM_QUOTE[Susun quotation harga, kuota, fitur, dan periode]
    CUSTOM_QUOTE --> CUSTOM_QUOTED[Status Quoted]
    CUSTOM_QUOTED -. "keputusan User Induk" .-> CUSTOM_ACCEPT{"Quotation diterima?"}
    CUSTOM_ACCEPT -- "Tidak" --> CUSTOM_CLOSED[Status Closed]
    CUSTOM_ACCEPT -- "Ya" --> CUSTOM_INVOICE[Buat tagihan; status Awaiting Payment]
    CUSTOM_INVOICE --> CUSTOM_PAYMENT[Memantau verifikasi pembayaran]
    CUSTOM_PAYMENT --> CUSTOM_PAID{"Pembayaran terverifikasi?"}
    CUSTOM_PAID -- "Tidak" --> CUSTOM_WAIT[Tetap Awaiting Payment atau catat kegagalan]
    CUSTOM_PAID -- "Ya" --> CUSTOM_CONFIG[Terapkan konfigurasi sesuai quotation]
    CUSTOM_CONFIG --> CUSTOM_ACTIVE[Status Active]
    CUSTOM_ACTIVE --> CUSTOM_NOTIFY[Kirim notifikasi aktivasi kepada User Induk]
    CUSTOM_NOTIFY --> CUSTOM_END([Selesai])
    CUSTOM_REJECT --> CUSTOM_END
    CUSTOM_CLOSED --> CUSTOM_END
    CUSTOM_WAIT --> CUSTOM_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class CUSTOM_START,CUSTOM_END startend;
    class CUSTOM_LIST,CUSTOM_REVIEW,CUSTOM_NEEDS,CUSTOM_REVISION,CUSTOM_RESUBMIT,CUSTOM_QUOTE,CUSTOM_QUOTED,CUSTOM_INVOICE,CUSTOM_PAYMENT,CUSTOM_WAIT,CUSTOM_CONFIG,CUSTOM_ACTIVE,CUSTOM_NOTIFY process;
    class CUSTOM_FEASIBLE,CUSTOM_ACCEPT,CUSTOM_PAID decision;
    class CUSTOM_REJECT,CUSTOM_CLOSED exception;
```

Status Enterprise/Custom: `Submitted`, `Under Review`, `Revision Requested`, `Quoted`, `Awaiting Payment`, `Active`, `Rejected`, dan `Closed`.

## 9. UA-07 — Mengelola Keamanan dan Insiden

```mermaid
flowchart TD
    SEC_START([Administrator membuka Keamanan]) --> SEC_AUDIT[Filter dan tinjau audit log aktivitas sensitif]
    SEC_AUDIT --> SEC_ALERT[Tinjau login berisiko dan indikasi akses lintas tenant]
    SEC_ALERT --> SEC_DANGER{"Aktivitas dikonfirmasi berbahaya?"}
    SEC_DANGER -- "Tidak" --> SEC_CLOSE[Catat hasil peninjauan]
    SEC_DANGER -- "Ya" --> SEC_TARGET[Pilih sesi atau token terdampak]
    SEC_TARGET --> SEC_REVOKE[Cabut akses]
    SEC_REVOKE --> SEC_REVOKE_OK{"Pencabutan berhasil?"}
    SEC_REVOKE_OK -- "Tidak" --> SEC_FAILED[Catat kegagalan dan eskalasi internal]
    SEC_REVOKE_OK -- "Ya" --> SEC_LOG[Catat pencabutan]
    SEC_LOG --> SEC_INCIDENT[Buat dan klasifikasikan insiden internal]
    SEC_INCIDENT --> SEC_IMPACT[Catat dampak, mitigasi, dan pelanggan terdampak]
    SEC_IMPACT --> SEC_SEVERITY{"Memerlukan respons darurat?"}
    SEC_SEVERITY -- "Ya" --> SEC_EMERGENCY[Aktifkan respons insiden darurat]
    SEC_SEVERITY -- "Tidak" --> SEC_NOTIFY
    SEC_EMERGENCY --> SEC_NOTIFY[Kirim notifikasi kepada pelanggan terdampak]
    SEC_NOTIFY --> SEC_END([Insiden tercatat dan ditangani])
    SEC_CLOSE --> SEC_END
    SEC_FAILED --> SEC_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class SEC_START,SEC_END startend;
    class SEC_AUDIT,SEC_ALERT,SEC_CLOSE,SEC_TARGET,SEC_REVOKE,SEC_LOG,SEC_INCIDENT,SEC_IMPACT,SEC_EMERGENCY,SEC_NOTIFY process;
    class SEC_DANGER,SEC_REVOKE_OK,SEC_SEVERITY decision;
    class SEC_FAILED exception;
```

## 10. UA-08 — Mengelola File dan Integrasi

```mermaid
flowchart TD
    FILE_START([Administrator membuka File dan Integrasi]) --> FILE_ACTION{"Pilih aktivitas"}
    FILE_ACTION -- "Karantina" --> FILE_LIST[Tampilkan file, alasan karantina, dan hasil pemindaian]
    FILE_LIST --> FILE_DECISION{"Keputusan pemeriksaan"}
    FILE_DECISION -- "Berbahaya" --> FILE_VALIDATE[Validasi hasil pemindaian]
    FILE_VALIDATE --> FILE_DELETE[Hapus file dari karantina]
    FILE_DELETE --> FILE_DELETE_OK{"Penghapusan berhasil?"}
    FILE_DELETE_OK -- "Tidak" --> FILE_ERROR[Catat kegagalan]
    FILE_DELETE_OK -- "Ya" --> FILE_LOG[Catat penghapusan]
    FILE_DECISION -- "Aman" --> FILE_REAUTH[Autentikasi ulang]
    FILE_REAUTH --> FILE_SECOND{"Administrator kedua menyetujui pelepasan?"}
    FILE_SECOND -- "Tidak" --> FILE_REJECT[Catat pelepasan ditolak]
    FILE_SECOND -- "Ya" --> FILE_RELEASE[Lepaskan file dan catat keputusan]
    FILE_ACTION -- "Integrasi" --> FILE_STATUS[Tampilkan status email, pembayaran, storage, malware scanner, dan AI]
    FILE_STATUS --> FILE_HISTORY[Tampilkan histori kegagalan integrasi]
    FILE_HISTORY --> FILE_RETRY{"Jalankan ulang proses gagal?"}
    FILE_RETRY -- "Tidak" --> FILE_END
    FILE_RETRY -- "Ya" --> FILE_IDEMPOTENT[Pastikan operasi idempoten]
    FILE_IDEMPOTENT --> FILE_RUN[Jalankan ulang proses]
    FILE_RUN --> FILE_RUN_OK{"Proses ulang berhasil?"}
    FILE_RUN_OK -- "Tidak" --> FILE_ERROR
    FILE_RUN_OK -- "Ya" --> FILE_RESULT[Catat hasil proses ulang]
    FILE_LOG --> FILE_END([Kembali ke Dashboard])
    FILE_REJECT --> FILE_END
    FILE_RELEASE --> FILE_END
    FILE_RESULT --> FILE_END
    FILE_ERROR --> FILE_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class FILE_START,FILE_END startend;
    class FILE_LIST,FILE_VALIDATE,FILE_DELETE,FILE_LOG,FILE_REAUTH,FILE_RELEASE,FILE_STATUS,FILE_HISTORY,FILE_IDEMPOTENT,FILE_RUN,FILE_RESULT process;
    class FILE_ACTION,FILE_DECISION,FILE_DELETE_OK,FILE_SECOND,FILE_RETRY,FILE_RUN_OK decision;
    class FILE_ERROR,FILE_REJECT exception;
```

Proses ulang tidak boleh mengirim email, membuat transaksi, menambah kuota, atau menghasilkan efek bisnis yang sama lebih dari satu kali.

## 11. UA-09 — Mengelola Backup, Retensi, dan Penghapusan

```mermaid
flowchart TD
    DATA_START([Administrator membuka Siklus Data]) --> DATA_ACTION{"Pilih aktivitas"}
    DATA_ACTION -- "Status backup" --> DATA_BACKUP[Tampilkan waktu dan status backup terakhir]
    DATA_BACKUP --> DATA_BACKUP_OK{"Backup berhasil?"}
    DATA_BACKUP_OK -- "Tidak" --> DATA_BACKUP_FAIL[Catat dan tampilkan kegagalan backup]
    DATA_BACKUP_OK -- "Ya" --> DATA_END
    DATA_ACTION -- "Uji restore" --> DATA_SELECT[Pilih backup uji]
    DATA_SELECT --> DATA_ISOLATED[Jalankan restore pada lingkungan terisolasi]
    DATA_ISOLATED --> DATA_VALIDATE[Validasi kelengkapan dan kemampuan pemulihan]
    DATA_VALIDATE --> DATA_TEST_OK{"Uji berhasil?"}
    DATA_TEST_OK -- "Tidak" --> DATA_TEST_FAIL[Catat kegagalan uji]
    DATA_TEST_OK -- "Ya" --> DATA_TEST_LOG[Catat hasil uji tanpa menimpa produksi]
    DATA_ACTION -- "Retensi" --> DATA_RETENTION[Tampilkan data dalam retensi 30 hari dan tanggal penghapusan]
    DATA_RETENTION --> DATA_DELETE[Tampilkan status penghapusan terjadwal]
    DATA_DELETE --> DATA_DELETE_FAIL{"Ada data gagal dihapus?"}
    DATA_DELETE_FAIL -- "Ya" --> DATA_DELETE_ERROR[Catat kegagalan untuk tindak lanjut]
    DATA_DELETE_FAIL -- "Tidak" --> DATA_EVIDENCE[Buat bukti penghapusan]
    DATA_EVIDENCE --> DATA_RECORD[Catat identitas permintaan, waktu, dan hasil tanpa menyimpan isi data]
    DATA_ACTION -- "Pemulihan retensi" --> DATA_WINDOW{"Masih dalam masa retensi?"}
    DATA_WINDOW -- "Tidak" --> DATA_RECOVERY_REJECT[Tolak pemulihan]
    DATA_WINDOW -- "Ya" --> DATA_TICKET[Validasi tiket, autentikasi ulang, dan persetujuan Administrator kedua]
    DATA_TICKET --> DATA_APPROVED{"Persetujuan lengkap?"}
    DATA_APPROVED -- "Tidak" --> DATA_RECOVERY_REJECT
    DATA_APPROVED -- "Ya" --> DATA_RECOVER[Jalankan pemulihan terotorisasi]
    DATA_RECOVER --> DATA_RECOVER_OK{"Pemulihan berhasil?"}
    DATA_RECOVER_OK -- "Tidak" --> DATA_RECOVERY_FAIL[Catat kegagalan]
    DATA_RECOVER_OK -- "Ya" --> DATA_RECOVERY_LOG[Catat hasil pemulihan]
    DATA_BACKUP_FAIL --> DATA_END
    DATA_TEST_FAIL --> DATA_END
    DATA_TEST_LOG --> DATA_END
    DATA_DELETE_ERROR --> DATA_END
    DATA_RECORD --> DATA_END
    DATA_RECOVERY_REJECT --> DATA_END
    DATA_RECOVERY_FAIL --> DATA_END
    DATA_RECOVERY_LOG --> DATA_END([Kembali ke Dashboard])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class DATA_START,DATA_END startend;
    class DATA_BACKUP,DATA_SELECT,DATA_ISOLATED,DATA_VALIDATE,DATA_TEST_LOG,DATA_RETENTION,DATA_DELETE,DATA_EVIDENCE,DATA_RECORD,DATA_TICKET,DATA_RECOVER,DATA_RECOVERY_LOG process;
    class DATA_ACTION,DATA_BACKUP_OK,DATA_TEST_OK,DATA_DELETE_FAIL,DATA_WINDOW,DATA_APPROVED,DATA_RECOVER_OK decision;
    class DATA_BACKUP_FAIL,DATA_TEST_FAIL,DATA_DELETE_ERROR,DATA_RECOVERY_REJECT,DATA_RECOVERY_FAIL exception;
```

Restore pengujian selalu dilakukan pada lingkungan terisolasi dan tidak boleh menimpa data produksi.

## 12. Handoff dan Status Lintas Aktor

| Proses | Pemicu dari User Induk | Aktivitas Administrator | Status/Hasil ke User Induk |
| --- | --- | --- | --- |
| Enterprise/Custom | `UI-08.1.2.4` | `UA-06.5` | Revisi, quotation, tagihan, aktivasi, penolakan, atau penutupan |
| Refund | `UI-08.8` | `UA-05.4` | Status refund dan dampak akses |
| Pembayaran | Transaksi paket atau add-on | `UA-05.3` | Status transaksi tercermin pada riwayat pembayaran |
| Koreksi paket/kuota | Tiket koreksi sah | `UA-05.E1` | Paket, kuota, dan histori diperbarui setelah dual approval |
| Reset password | Permintaan pemulihan terverifikasi | `UA-06.2` | Tautan reset dikirim; Administrator tidak menetapkan password |
| Reset MFA | Permintaan setelah verifikasi identitas | `UA-06.3` | Faktor lama dicabut dan instruksi aktivasi baru dikirim |
| Suspend/reaktivasi | Keputusan operasional yang sah | `UA-06.4` | Pengguna menerima alasan dan status akun terbaru |

Garis lintas aktor merupakan handoff dan status update, bukan relasi UML `include` atau `extend`.

## 13. Aturan Bisnis

- MFA wajib sebelum Administrator memperoleh akses portal.
- Dashboard hanya menampilkan data operasional sesuai kewenangan dan pendapatan agregat.
- Setiap tindakan Administrator memiliki identitas pelaksana, waktu, alasan bila diperlukan, dan audit log.
- Tindakan berisiko tinggi memerlukan autentikasi ulang.
- Koreksi billing, refund, pelepasan file karantina, dan pemulihan data memerlukan Administrator kedua.
- Administrator tidak dapat menetapkan password pengguna atau mengubah keanggotaan perusahaan.
- Enterprise/Custom tidak aktif sebelum quotation diterima dan pembayaran terverifikasi.
- Refund yang ditolak tidak mengubah akses; refund berhasil menyesuaikan akses tepat satu kali.
- Retry integrasi wajib idempoten.
- Audit log dan bukti penghapusan tidak menyimpan kembali isi data pelanggan.
- Retensi data perusahaan setelah langganan berakhir mengikuti kebijakan 30 hari dan siklus backup maksimum 90 hari.

## 14. Referensi

- [Business Flow Overview](./00-Business-Flow-Overview.md)
- [Authentication](./01-Authentication.md)
- [Membership and Organization](./05-Membership-Organization.md)
- [Subscription and Billing](./06-Subscription-Billing.md)
- [Use Case Administrator](../02-Use-Case/Admin.md)
- [Use Case User Induk](../02-Use-Case/Organization-Owner.md)
- [Non-Functional Requirements](../01-Requirements/Non-Functional-Requirements.md)
