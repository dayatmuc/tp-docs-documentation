# Authentication Business Flow — TP Docs (With Consultant)

## 1. Tujuan dan Cakupan

Dokumen ini menjelaskan flow autentikasi lengkap untuk **User Induk**, **Karyawan**, dan **Administrator**, meliputi registrasi, verifikasi email, login, MFA, lupa password, pengelolaan sesi, serta logout.

User Induk dan Karyawan menggunakan mekanisme autentikasi akun yang sama. Perbedaannya:

| Aspek                           | User Induk                                     | Karyawan                                               | Administrator                           |
| ------------------------------- | ---------------------------------------------- | ------------------------------------------------------ | --------------------------------------- |
| Sumber akun                     | Registrasi publik sekaligus membuat perusahaan | Registrasi biasa atau registrasi melalui link undangan | Dibuat melalui proses internal platform |
| Data organisasi saat registrasi | Wajib                                          | Tidak membuat organisasi                               | Tidak berlaku                           |
| Verifikasi email                | Wajib                                          | Wajib                                                  | Wajib sebelum akun digunakan            |
| MFA                             | Wajib                                          | Opsional pada versi awal                               | Wajib                                   |
| Tujuan setelah login            | Dashboard User Induk                           | Status akses atau konteks link undangan                | Dashboard operasional Administrator     |
| Lupa password mandiri           | Ya                                             | Ya                                                     | Ya                                      |

## 2. Flow Autentikasi User Induk

### 2.1 Registrasi Perusahaan dan Aktivasi Akun

```mermaid
flowchart TD
    UI_START([Mulai Registrasi]) --> UI_PERSONAL[Input nama lengkap, email, password, dan konfirmasi password]
    UI_PERSONAL --> UI_COMPANY[Input nama perusahaan, NPWP, alamat, dan nomor telepon]
    UI_COMPANY --> UI_VALIDATE[Validasi kelengkapan, format email, format NPWP, dan kesamaan password]
    UI_VALIDATE --> UI_VALID{"Data registrasi valid?"}

    UI_VALID -- "Tidak" --> UI_ERROR[Tampilkan field dan alasan yang harus diperbaiki]
    UI_ERROR --> UI_PERSONAL
    UI_VALID -- "Ya" --> UI_EMAIL_USED{"Email sudah digunakan?"}
    UI_EMAIL_USED -- "Ya" --> UI_EXISTING[Arahkan ke login atau lupa password]
    UI_EXISTING --> UI_END_EXISTING([Registrasi tidak dilanjutkan])
    UI_EMAIL_USED -- "Tidak" --> UI_CREATE[Hasilkan akun User Induk dan perusahaan berstatus Pending Verification]
    UI_CREATE --> UI_SEND[Email sistem mengirim link verifikasi]
    UI_SEND --> UI_INBOX[User Induk membuka email verifikasi]
    UI_INBOX --> UI_LINK[User Induk membuka link verifikasi]
    UI_LINK --> UI_LINK_VALID{"Token valid dan belum kedaluwarsa?"}

    UI_LINK_VALID -- "Tidak" --> UI_RESEND[Meminta pengiriman ulang link verifikasi]
    UI_RESEND --> UI_SEND
    UI_LINK_VALID -- "Ya" --> UI_ACTIVATE[Aktifkan akun dan perusahaan]
    UI_ACTIVATE --> UI_MFA_SETUP[Daftarkan MFA wajib dan simpan recovery code]
    UI_MFA_SETUP --> UI_READY([Akun siap digunakan])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class UI_START,UI_END_EXISTING,UI_READY terminal;
    class UI_PERSONAL,UI_COMPANY,UI_VALIDATE,UI_ERROR,UI_EXISTING,UI_CREATE,UI_SEND,UI_INBOX,UI_LINK,UI_RESEND,UI_ACTIVATE,UI_MFA_SETUP process;
    class UI_VALID,UI_EMAIL_USED,UI_LINK_VALID decision;
```

### 2.2 Login, MFA, Sesi, dan Logout User Induk

```mermaid
flowchart TD
    UIL_START([Halaman Login]) --> UIL_INPUT[Input email dan password]
    UIL_INPUT --> UIL_CHECK[Validasi kredensial]
    UIL_CHECK --> UIL_VALID{"Kredensial valid?"}

    UIL_VALID -- "Tidak" --> UIL_FAILED[Catat kegagalan dan tampilkan pesan umum]
    UIL_FAILED --> UIL_LIMIT{"Percobaan gagal melewati batas?"}
    UIL_LIMIT -- "Ya" --> UIL_LOCK[Blokir sementara dan kirim notifikasi keamanan]
    UIL_LOCK --> UIL_END_LOCK([Login dihentikan sementara])
    UIL_LIMIT -- "Tidak" --> UIL_FORGOT{"Pilih lupa password?"}
    UIL_FORGOT -- "Tidak" --> UIL_INPUT
    UIL_FORGOT -- "Ya" --> UIL_RESET_LINK[Masuk ke flow lupa password]

    UIL_VALID -- "Ya" --> UIL_VERIFIED{"Email sudah terverifikasi?"}
    UIL_VERIFIED -- "Tidak" --> UIL_RESEND[Kirimi ulang link verifikasi]
    UIL_RESEND --> UIL_END_VERIFY([Menunggu verifikasi email])
    UIL_VERIFIED -- "Ya" --> UIL_MFA[Input kode MFA]
    UIL_MFA --> UIL_MFA_VALID{"Kode MFA valid?"}
    UIL_MFA_VALID -- "Tidak" --> UIL_MFA_RETRY[Ulangi kode atau gunakan recovery code]
    UIL_MFA_RETRY --> UIL_MFA
    UIL_MFA_VALID -- "Ya" --> UIL_SESSION[Buat sesi aman dan catat aktivitas login]
    UIL_SESSION --> UIL_DASHBOARD[Masuk ke Dashboard User Induk]

    UIL_DASHBOARD --> UIL_ACTION{"Kondisi sesi?"}
    UIL_ACTION -- "Masih aktif" --> UIL_DASHBOARD
    UIL_ACTION -- "Tidak aktif 30 menit" --> UIL_TIMEOUT[Akhiri sesi otomatis]
    UIL_ACTION -- "User memilih logout" --> UIL_LOGOUT[Cabut sesi dan token aktif]
    UIL_TIMEOUT --> UIL_LOGIN_AGAIN([Harus login kembali])
    UIL_LOGOUT --> UIL_END([Selesai])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class UIL_START,UIL_END_LOCK,UIL_END_VERIFY,UIL_LOGIN_AGAIN,UIL_END terminal;
    class UIL_INPUT,UIL_CHECK,UIL_FAILED,UIL_LOCK,UIL_RESET_LINK,UIL_RESEND,UIL_MFA,UIL_MFA_RETRY,UIL_SESSION,UIL_DASHBOARD,UIL_TIMEOUT,UIL_LOGOUT process;
    class UIL_VALID,UIL_LIMIT,UIL_FORGOT,UIL_VERIFIED,UIL_MFA_VALID,UIL_ACTION decision;
```

### 2.3 Lupa Password User Induk

```mermaid
flowchart TD
    UIF_START([Pilih Lupa Password]) --> UIF_EMAIL[Input email akun]
    UIF_EMAIL --> UIF_SUBMIT[Kirim permintaan reset]
    UIF_SUBMIT --> UIF_RESPONSE[Tampilkan respons umum tanpa mengungkap keberadaan akun]
    UIF_SUBMIT --> UIF_EXISTS{"Email terdaftar dan akun aktif?"}
    UIF_EXISTS -- "Tidak" --> UIF_END_GENERIC([Proses selesai tanpa mengirim link])
    UIF_EXISTS -- "Ya" --> UIF_SEND[Kirim link reset sekali pakai dengan masa berlaku terbatas]
    UIF_SEND --> UIF_OPEN[Pengguna membuka link reset]
    UIF_OPEN --> UIF_VALID{"Link valid, belum digunakan, dan belum kedaluwarsa?"}
    UIF_VALID -- "Tidak" --> UIF_INVALID[Tampilkan link tidak valid atau kedaluwarsa]
    UIF_INVALID --> UIF_EMAIL
    UIF_VALID -- "Ya" --> UIF_NEW[Input password baru dan konfirmasi password]
    UIF_NEW --> UIF_PASSWORD{"Password memenuhi kebijakan dan sesuai konfirmasi?"}
    UIF_PASSWORD -- "Tidak" --> UIF_NEW
    UIF_PASSWORD -- "Ya" --> UIF_SAVE[Simpan hash password baru dan tandai token telah digunakan]
    UIF_SAVE --> UIF_REVOKE[Cabut sesi aktif lainnya dan kirim notifikasi perubahan password]
    UIF_REVOKE --> UIF_LOGIN([Kembali ke login dan MFA])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class UIF_START,UIF_END_GENERIC,UIF_LOGIN terminal;
    class UIF_EMAIL,UIF_SUBMIT,UIF_RESPONSE,UIF_SEND,UIF_OPEN,UIF_INVALID,UIF_NEW,UIF_SAVE,UIF_REVOKE process;
    class UIF_EXISTS,UIF_VALID,UIF_PASSWORD decision;
```

## 3. Flow Autentikasi Karyawan

### 3.1 Registrasi atau Login melalui Link Undangan

```mermaid
flowchart TD
    UK_START([Menerima Link Undangan]) --> UK_OPEN[Buka link undangan]
    UK_OPEN --> UK_TOKEN{"Token valid dan belum kedaluwarsa?"}
    UK_TOKEN -- "Tidak" --> UK_INVALID[Tampilkan undangan salah, dibatalkan, digunakan, atau kedaluwarsa]
    UK_INVALID --> UK_END_INVALID([Tidak memperoleh membership])
    UK_TOKEN -- "Ya" --> UK_ACCOUNT{"Email tujuan sudah memiliki akun?"}

    UK_ACCOUNT -- "Tidak" --> UK_REGISTER[Input nama, email tujuan, password, dan konfirmasi password]
    UK_REGISTER --> UK_VALIDATE[Validasi data dan buat akun Pending Verification]
    UK_VALIDATE --> UK_SEND_VERIFY[Kirim link verifikasi email]
    UK_SEND_VERIFY --> UK_VERIFY[Calon Karyawan memverifikasi email]
    UK_VERIFY --> UK_VERIFY_VALID{"Verifikasi berhasil?"}
    UK_VERIFY_VALID -- "Tidak" --> UK_SEND_VERIFY
    UK_VERIFY_VALID -- "Ya" --> UK_RETURN[Kembali ke konteks undangan]

    UK_ACCOUNT -- "Ya" --> UK_LOGIN[Login menggunakan email dan password]
    UK_LOGIN --> UK_LOGIN_VALID{"Kredensial valid?"}
    UK_LOGIN_VALID -- "Tidak" --> UK_RETRY[Ulangi login atau jalankan lupa password]
    UK_RETRY --> UK_LOGIN
    UK_LOGIN_VALID -- "Ya" --> UK_RETURN

    UK_RETURN --> UK_EMAIL_MATCH{"Email akun sama dengan email tujuan undangan?"}
    UK_EMAIL_MATCH -- "Tidak" --> UK_MISMATCH[Tolak akses karena link tidak dapat digunakan akun lain]
    UK_MISMATCH --> UK_END_INVALID
    UK_EMAIL_MATCH -- "Ya" --> UK_DECISION{"Menerima undangan perusahaan?"}
    UK_DECISION -- "Tidak" --> UK_REJECT[Catat status undangan Rejected]
    UK_REJECT --> UK_END_REJECT([Selesai tanpa membership])
    UK_DECISION -- "Ya" --> UK_MEMBERSHIP[Aktifkan membership dan warisi akses paket perusahaan]
    UK_MEMBERSHIP --> UK_MFA{"MFA Karyawan telah diaktifkan?"}
    UK_MFA -- "Ya" --> UK_MFA_CHECK[Verifikasi kode MFA]
    UK_MFA_CHECK --> UK_ACCESS
    UK_MFA -- "Tidak" --> UK_ACCESS[Masuk ke status akses langganan]
    UK_ACCESS --> UK_END([Selesai])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class UK_START,UK_END_INVALID,UK_END_REJECT,UK_END terminal;
    class UK_OPEN,UK_INVALID,UK_REGISTER,UK_VALIDATE,UK_SEND_VERIFY,UK_VERIFY,UK_RETURN,UK_LOGIN,UK_RETRY,UK_MISMATCH,UK_REJECT,UK_MEMBERSHIP,UK_MFA_CHECK,UK_ACCESS process;
    class UK_TOKEN,UK_ACCOUNT,UK_VERIFY_VALID,UK_LOGIN_VALID,UK_EMAIL_MATCH,UK_DECISION,UK_MFA decision;
```

### 3.2 Login, Lupa Password, Sesi, dan Logout Karyawan

Flow login dan lupa password Karyawan mengikuti mekanisme User Induk pada bagian 2.2 dan 2.3 dengan perbedaan berikut:

```mermaid
flowchart TD
    UKL_START([Halaman Login]) --> UKL_INPUT[Input email dan password]
    UKL_INPUT --> UKL_VALID{"Kredensial dan email terverifikasi?"}
    UKL_VALID -- "Tidak" --> UKL_OPTION{"Lupa password atau email belum terverifikasi?"}
    UKL_OPTION -- "Lupa password" --> UKL_RESET[Jalankan reset password yang sama dengan User Induk]
    UKL_RESET --> UKL_INPUT
    UKL_OPTION -- "Belum terverifikasi" --> UKL_VERIFY[Kirim ulang link verifikasi email]
    UKL_VERIFY --> UKL_INPUT
    UKL_OPTION -- "Kredensial salah" --> UKL_INPUT

    UKL_VALID -- "Ya" --> UKL_MFA{"MFA Karyawan aktif?"}
    UKL_MFA -- "Ya" --> UKL_CODE[Input dan validasi kode MFA]
    UKL_CODE --> UKL_SESSION
    UKL_MFA -- "Tidak" --> UKL_SESSION[Buat sesi Karyawan]
    UKL_SESSION --> UKL_DESTINATION{"Ada konteks link undangan?"}
    UKL_DESTINATION -- "Ya" --> UKL_INVITATION[Lanjutkan validasi dan keputusan undangan]
    UKL_DESTINATION -- "Tidak" --> UKL_STATUS[Masuk ke halaman status akses langganan]

    UKL_INVITATION --> UKL_ACTION{"Sesi masih aktif?"}
    UKL_STATUS --> UKL_ACTION
    UKL_ACTION -- "Ya" --> UKL_STATUS
    UKL_ACTION -- "Timeout atau logout" --> UKL_REVOKE[Akhiri sesi Karyawan]
    UKL_REVOKE --> UKL_END([Selesai])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class UKL_START,UKL_END terminal;
    class UKL_INPUT,UKL_RESET,UKL_VERIFY,UKL_CODE,UKL_SESSION,UKL_INVITATION,UKL_STATUS,UKL_REVOKE process;
    class UKL_VALID,UKL_OPTION,UKL_MFA,UKL_DESTINATION,UKL_ACTION decision;
```

## 4. Flow Autentikasi Administrator

### 4.1 Login Administrator, MFA, dan Pemeriksaan Keamanan

```mermaid
flowchart TD
    UA_START([Portal Administrator]) --> UA_INPUT[Input email dan password Administrator]
    UA_INPUT --> UA_SUBMIT[Kirim kredensial login]
    UA_SUBMIT --> UA_VALID{"Kredensial valid dan akun aktif?"}

    UA_VALID -- "Tidak" --> UA_FAILED[Catat hasil login gagal]
    UA_FAILED --> UA_RISK{"Terindikasi brute force atau aktivitas mencurigakan?"}
    UA_RISK -- "Ya" --> UA_BLOCK[Blokir sementara, cabut sesi berisiko, dan kirim notifikasi keamanan]
    UA_BLOCK --> UA_END_BLOCK([Akses ditolak])
    UA_RISK -- "Tidak" --> UA_FORGOT{"Administrator memilih lupa password?"}
    UA_FORGOT -- "Tidak" --> UA_INPUT
    UA_FORGOT -- "Ya" --> UA_RESET[Masuk ke flow lupa password Administrator]

    UA_VALID -- "Ya" --> UA_MFA[Input kode MFA wajib]
    UA_MFA --> UA_MFA_VALID{"Kode MFA valid dan belum kedaluwarsa?"}
    UA_MFA_VALID -- "Tidak" --> UA_MFA_RETRY[Catat kegagalan dan minta kode MFA kembali]
    UA_MFA_RETRY --> UA_MFA
    UA_MFA_VALID -- "Ya" --> UA_DEVICE[Validasi perangkat, IP, lokasi, dan kebijakan sesi]
    UA_DEVICE --> UA_DEVICE_RISK{"Perangkat atau lokasi berisiko?"}
    UA_DEVICE_RISK -- "Ya" --> UA_BLOCK
    UA_DEVICE_RISK -- "Tidak" --> UA_LOG[Catat waktu, hasil login, perangkat, dan IP]
    UA_LOG --> UA_SESSION[Buat sesi Administrator]
    UA_SESSION --> UA_DASHBOARD[Masuk ke Dashboard Operasional]

    UA_DASHBOARD --> UA_STATE{"Kondisi sesi?"}
    UA_STATE -- "Aktif" --> UA_DASHBOARD
    UA_STATE -- "Tidak aktif 30 menit" --> UA_TIMEOUT[Akhiri sesi otomatis]
    UA_STATE -- "Logout" --> UA_LOGOUT[Cabut sesi dan token Administrator]
    UA_TIMEOUT --> UA_LOGIN_AGAIN([Harus login dan MFA kembali])
    UA_LOGOUT --> UA_END([Selesai])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class UA_START,UA_END_BLOCK,UA_LOGIN_AGAIN,UA_END terminal;
    class UA_INPUT,UA_SUBMIT,UA_FAILED,UA_BLOCK,UA_RESET,UA_MFA,UA_MFA_RETRY,UA_DEVICE,UA_LOG,UA_SESSION,UA_DASHBOARD,UA_TIMEOUT,UA_LOGOUT process;
    class UA_VALID,UA_RISK,UA_FORGOT,UA_MFA_VALID,UA_DEVICE_RISK,UA_STATE decision;
```

### 4.2 Lupa Password Administrator

```mermaid
flowchart TD
    UAF_START([Pilih Lupa Password]) --> UAF_EMAIL[Input email Administrator]
    UAF_EMAIL --> UAF_VALIDATE[Validasi akun Administrator secara internal]
    UAF_VALIDATE --> UAF_EXISTS{"Akun valid dan aktif?"}
    UAF_EXISTS -- "Tidak" --> UAF_GENERIC[Tampilkan respons umum tanpa mengungkap status akun]
    UAF_GENERIC --> UAF_END_GENERIC([Selesai])
    UAF_EXISTS -- "Ya" --> UAF_SEND[Kirim link reset sekali pakai dan berbatas waktu]
    UAF_SEND --> UAF_OPEN[Administrator membuka link]
    UAF_OPEN --> UAF_TOKEN{"Token valid, belum digunakan, dan belum kedaluwarsa?"}
    UAF_TOKEN -- "Tidak" --> UAF_INVALID[Tampilkan link tidak valid atau kedaluwarsa]
    UAF_INVALID --> UAF_START
    UAF_TOKEN -- "Ya" --> UAF_NEW[Input password baru dan konfirmasi]
    UAF_NEW --> UAF_POLICY{"Password valid dan sesuai konfirmasi?"}
    UAF_POLICY -- "Tidak" --> UAF_NEW
    UAF_POLICY -- "Ya" --> UAF_SAVE[Simpan password baru dan tandai token telah digunakan]
    UAF_SAVE --> UAF_REVOKE[Cabut seluruh sesi Administrator yang aktif]
    UAF_REVOKE --> UAF_NOTIFY[Kirim notifikasi keamanan dan catat audit log]
    UAF_NOTIFY --> UAF_LOGIN([Kembali ke login dan MFA wajib])

    classDef terminal fill:#1f6fd1,color:#fff,stroke:#1556a8,stroke-width:2px;
    classDef process fill:#438fe6,color:#fff,stroke:#2875c7,stroke-width:1.5px;
    classDef decision fill:#1761cf,color:#fff,stroke:#104ba3,stroke-width:2px;
    class UAF_START,UAF_END_GENERIC,UAF_LOGIN terminal;
    class UAF_EMAIL,UAF_VALIDATE,UAF_GENERIC,UAF_SEND,UAF_OPEN,UAF_INVALID,UAF_NEW,UAF_SAVE,UAF_REVOKE,UAF_NOTIFY process;
    class UAF_EXISTS,UAF_TOKEN,UAF_POLICY decision;
```

## 5. Aturan Autentikasi Bersama

1. Password tidak pernah disimpan dalam bentuk asli.
2. Respons lupa password tidak boleh mengungkap apakah suatu email terdaftar.
3. Link verifikasi dan reset password hanya dapat digunakan oleh penerima, memiliki masa berlaku terbatas, dan tidak dapat digunakan ulang.
4. Perubahan password mencabut sesi lain yang masih aktif.
5. MFA wajib untuk User Induk dan Administrator; MFA Karyawan opsional pada versi awal.
6. Sesi yang tidak aktif selama 30 menit berakhir otomatis.
7. Percobaan login gagal berulang dikenai rate limiting dan penguncian sementara.
8. Login, logout, reset password, perubahan MFA, kegagalan autentikasi, dan pemblokiran dicatat pada audit log tanpa menyimpan password atau token asli.
9. Link undangan Karyawan tetap terikat pada email tujuan ketika penerima harus melakukan registrasi atau login.
10. Administrator tidak dapat melihat atau menetapkan password User Induk maupun Karyawan.

## 6. Status dan Hasil Akhir

| Proses                               | Status / Hasil                                                                   |
| ------------------------------------ | -------------------------------------------------------------------------------- |
| Registrasi User Induk                | `Pending Verification` → `Active`                                                |
| Registrasi Karyawan melalui undangan | `Pending Verification` → akun aktif → membership aktif setelah undangan diterima |
| Undangan Karyawan                    | `Pending`, `Accepted`, `Rejected`, `Expired`, atau `Cancelled`                   |
| Login berhasil                       | Sesi aktif sesuai aktor dan hak akses                                            |
| Login mencurigakan                   | Percobaan diblokir dan dicatat                                                   |
| Reset password                       | Token digunakan satu kali, password diperbarui, sesi lama dicabut                |
| Session timeout                      | Sesi berakhir dan pengguna harus login kembali                                   |

## 7. Referensi

- [Use Case User Induk](../02-Use-Case/Organization-Owner.md): `UI-01`, `UI-02`, dan `UI-03`.
- [Use Case Karyawan](../02-Use-Case/Employee.md): `UK-01` dan `UK-02`.
- [Use Case Administrator](../02-Use-Case/Admin.md): `UA-01`.
- [Non-Functional Requirements](../01-Requirements/Non-Functional-Requirements.md): keamanan akun, MFA, sesi, rate limiting, password, dan audit log.

---

## Ekstensi Business Flow Layanan Konsultan

Autentikasi User Induk, Karyawan, dan Administrator telah dijelaskan lengkap pada bagian sebelumnya. Ekstensi berikut menambahkan autentikasi Konsultan.

### Onboarding Konsultan

```mermaid
flowchart TD
    ADMIN([Administrator]) --> EMAIL[Input email Konsultan]
    EMAIL --> EXISTS{Email sudah digunakan?}
    EXISTS -- Ya --> REJECT[Tolak atau gunakan akun yang diverifikasi sesuai kebijakan]
    EXISTS -- Tidak --> INVITE[Buat invitation token hash dan expiry]
    INVITE --> SEND[Kirim email undangan]
    SEND --> OPEN[Konsultan membuka link]
    OPEN --> VALID{Token valid?}
    VALID -- Tidak --> EXPIRED[Tampilkan invalid atau expired]
    VALID -- Ya --> PASSWORD[Konsultan membuat password sendiri]
    PASSWORD --> MFA[Aktifkan MFA wajib]
    MFA --> PROFILE[Admin melengkapi profil dan kapasitas]
    PROFILE --> ACTIVE[Akun Konsultan aktif]
```

Admin tidak menetapkan password. Reset password dilakukan pemilik akun melalui tautan aman. Suspend Konsultan mencabut seluruh session dan scoped assignment access.
