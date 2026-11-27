# Membership and Organization Business Flow — TP Docs (With Consultant)

## 1. Tujuan dan Cakupan

Dokumen ini menjelaskan hubungan akun pribadi dengan organisasi perusahaan, mulai dari pembentukan organisasi oleh User Induk, pengiriman undangan, penerimaan membership oleh Karyawan, pemantauan akses, penugasan dokumen oleh User Induk, hingga pencabutan akses.

Aktor yang terlibat:

- **User Induk** sebagai pemilik organisasi dan pengelola anggota.
- **Karyawan** sebagai pengguna tambahan yang menerima akses melalui undangan.
- **Sistem TP Docs** sebagai pengelola token undangan, membership, kursi, dan masa akses.

Administrator tidak mengundang, menghapus, atau menugaskan Karyawan. Setiap pengguna memiliki akun dan password sendiri; akun tidak dibagikan.

Referensi use case: `UI-08.2` sampai `UI-08.5`, `UK-01`, dan `UK-02`.

## 2. Pembentukan Organisasi, User Induk, dan Hak Multi-Pengguna

Pengguna pertama yang membuat perusahaan tetap dicatat sebagai **User Induk** pada seluruh paket. Namun, status User Induk tidak otomatis memberikan hak untuk mengundang Karyawan. Hak multi-pengguna hanya aktif apabila organisasi menggunakan Professional atau Enterprise/Custom dengan kursi yang tersedia.

```mermaid
flowchart TD
    ORG_START([Akun perusahaan berhasil didaftarkan]) --> ORG_PROFILE[User mengisi profil perusahaan]
    ORG_PROFILE --> ORG_CREATE[Sistem membuat Organization]
    ORG_CREATE --> ORG_OWNER[Sistem menetapkan pendaftar sebagai User Induk]
    ORG_OWNER --> ORG_MEMBERSHIP[Buat membership User Induk berstatus Active]
    ORG_MEMBERSHIP --> ORG_PLAN[Hubungkan organisasi dengan paket dan masa aktif]
    ORG_PLAN --> ORG_SEAT[Hitung kursi User Induk sebagai satu kursi terpakai]
    ORG_SEAT --> ORG_MULTI{"Paket mendukung multi-pengguna?"}
    ORG_MULTI -- "Free/Trial atau Basic" --> ORG_SINGLE[Gunakan organisasi sebagai pengguna tunggal]
    ORG_SINGLE --> ORG_LOCK[Nonaktifkan fitur undangan dan add-on kursi]
    ORG_MULTI -- "Professional atau Enterprise/Custom" --> ORG_LIMIT[Aktifkan pengelolaan anggota sesuai jumlah kursi]
    ORG_LOCK --> ORG_READY([Organisasi siap digunakan])
    ORG_LIMIT --> ORG_READY

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    class ORG_START,ORG_READY startend;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    class ORG_PROFILE,ORG_CREATE,ORG_OWNER,ORG_MEMBERSHIP,ORG_PLAN,ORG_SEAT,ORG_SINGLE,ORG_LOCK,ORG_LIMIT process;
    class ORG_MULTI decision;
```

User pertama yang menyelesaikan registrasi dan profil perusahaan menjadi User Induk. Pada Free/Trial dan Basic, User Induk tetap dapat menggunakan fitur perusahaan sesuai paket, tetapi menu undangan Karyawan tidak dapat digunakan. Pengalihan kepemilikan organisasi belum menjadi bagian dari flow versi ini.

## 3. Flow User Induk Mengundang Karyawan

Undangan hanya tersedia pada paket Professional dan Enterprise/Custom sesuai jumlah kursi. Free/Trial dan Basic hanya mendukung satu User Induk.

```mermaid
flowchart TD
    INV_START([User Induk membuka Kelola Anggota]) --> INV_ACCESS[Sistem tampilkan kursi terpakai dan tersedia]
    INV_ACCESS --> INV_PLAN{"Paket mendukung multi-pengguna?"}
    INV_PLAN -- "Tidak" --> INV_UPGRADE[Arahkan User Induk untuk meningkatkan paket]
    INV_PLAN -- "Ya" --> INV_SEAT{"Kursi tersedia?"}
    INV_SEAT -- "Tidak" --> INV_ADDON[Professional: tawarkan pembelian add-on kursi]
    INV_ADDON --> INV_PAID{"Pembayaran add-on berhasil?"}
    INV_PAID -- "Tidak" --> INV_STOP[Undangan belum dapat dikirim]
    INV_PAID -- "Ya" --> INV_REFRESH[Tambahkan kursi sampai akhir periode paket]
    INV_REFRESH --> INV_EMAIL
    INV_SEAT -- "Ya" --> INV_EMAIL[User Induk input email calon Karyawan]
    INV_EMAIL --> INV_VALIDATE{"Email valid dan belum menjadi anggota organisasi?"}
    INV_VALIDATE -- "Tidak" --> INV_ERROR[Tampilkan alasan dan minta perbaikan]
    INV_ERROR --> INV_EMAIL
    INV_VALIDATE -- "Ya" --> INV_OTHER{"Akun menjadi anggota perusahaan lain?"}
    INV_OTHER -- "Ya" --> INV_REJECT[Tolak sesuai aturan satu perusahaan per Karyawan]
    INV_OTHER -- "Tidak" --> INV_TOKEN[Sistem membuat token khusus email tujuan]
    INV_TOKEN --> INV_PENDING[Simpan Invitation berstatus Pending dan masa kedaluwarsa]
    INV_PENDING --> INV_SEND[Kirim link undangan melalui layanan email sistem]
    INV_SEND --> INV_RESULT{"Email berhasil dikirim?"}
    INV_RESULT -- "Tidak" --> INV_FAILED[Catat kegagalan; undangan dapat dikirim ulang]
    INV_RESULT -- "Ya" --> INV_LIST[Tampilkan undangan pada daftar tertunda]
    INV_LIST --> INV_END([Menunggu keputusan calon Karyawan])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class INV_START,INV_END startend;
    class INV_ACCESS,INV_UPGRADE,INV_ADDON,INV_REFRESH,INV_EMAIL,INV_TOKEN,INV_PENDING,INV_SEND,INV_LIST process;
    class INV_PLAN,INV_SEAT,INV_PAID,INV_VALIDATE,INV_OTHER,INV_RESULT decision;
    class INV_STOP,INV_ERROR,INV_REJECT,INV_FAILED exception;
```

## 4. Flow Karyawan Menerima Undangan

Karyawan menerima link, bukan akun atau password dari User Induk. Jika calon Karyawan belum memiliki akun, registrasi dan verifikasi email dilakukan dari konteks undangan agar link tetap dapat dilanjutkan.

```mermaid
flowchart TD
    EMP_START([Calon Karyawan membuka link undangan]) --> EMP_TOKEN{"Token tersedia, belum digunakan, dan belum kedaluwarsa?"}
    EMP_TOKEN -- "Tidak" --> EMP_INVALID[Tampilkan undangan tidak valid atau kedaluwarsa]
    EMP_TOKEN -- "Ya" --> EMP_ACCOUNT{"Sudah memiliki akun TP Docs?"}
    EMP_ACCOUNT -- "Tidak" --> EMP_REGISTER[Registrasi akun pribadi menggunakan email tujuan]
    EMP_REGISTER --> EMP_VERIFY[Verifikasi email akun]
    EMP_VERIFY --> EMP_CONTINUE[Lanjutkan kembali konteks undangan]
    EMP_ACCOUNT -- "Ya" --> EMP_LOGIN[Login atau lanjutkan dengan akun aktif]
    EMP_CONTINUE --> EMP_MATCH
    EMP_LOGIN --> EMP_MATCH{"Email akun sama dengan email tujuan?"}
    EMP_MATCH -- "Tidak" --> EMP_MISMATCH[Tolak penerimaan oleh akun lain]
    EMP_MATCH -- "Ya" --> EMP_STATUS{"Organisasi, paket, dan kursi masih tersedia?"}
    EMP_STATUS -- "Tidak" --> EMP_UNAVAILABLE[Tampilkan akses belum dapat diaktifkan]
    EMP_STATUS -- "Ya" --> EMP_DECISION{"Bersedia bergabung?"}
    EMP_DECISION -- "Tidak" --> EMP_REJECT[Ubah Invitation menjadi Rejected]
    EMP_DECISION -- "Ya" --> EMP_MEMBER[Buat Membership Karyawan berstatus Active]
    EMP_MEMBER --> EMP_USE[Ubah token menjadi Used]
    EMP_USE --> EMP_SEAT[Tambahkan jumlah kursi terpakai]
    EMP_SEAT --> EMP_ACCESS[Wariskan fitur dan tanggal berakhir paket perusahaan]
    EMP_ACCESS --> EMP_END([Karyawan berhasil bergabung])

    classDef startend fill:#dcfce7,stroke:#16a34a,color:#052e16,stroke-width:2px;
    classDef process fill:#f0fdf4,stroke:#22c55e,color:#052e16;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class EMP_START,EMP_END startend;
    class EMP_REGISTER,EMP_VERIFY,EMP_CONTINUE,EMP_LOGIN,EMP_MEMBER,EMP_USE,EMP_SEAT,EMP_ACCESS process;
    class EMP_TOKEN,EMP_ACCOUNT,EMP_MATCH,EMP_STATUS,EMP_DECISION decision;
    class EMP_INVALID,EMP_MISMATCH,EMP_UNAVAILABLE,EMP_REJECT exception;
```

## 5. Flow Undangan Tertunda

```mermaid
flowchart TD
    PEND_START([User Induk membuka Undangan Tertunda]) --> PEND_LIST[Tampilkan email, pengirim, waktu kirim, dan kedaluwarsa]
    PEND_LIST --> PEND_ACTION{"Pilih tindakan"}
    PEND_ACTION -- "Kirim ulang" --> PEND_VALID{"Undangan masih dapat digunakan?"}
    PEND_VALID -- "Ya" --> PEND_RESEND[Kirim ulang link yang masih berlaku]
    PEND_VALID -- "Tidak" --> PEND_NEW[Batalkan token lama dan buat token baru]
    PEND_NEW --> PEND_RESEND
    PEND_RESEND --> PEND_LOG[Perbarui waktu pengiriman dan audit log]
    PEND_ACTION -- "Batalkan" --> PEND_CANCEL[Ubah status menjadi Cancelled dan nonaktifkan token]
    PEND_ACTION -- "Tidak ada" --> PEND_END([Kembali ke daftar anggota])
    PEND_LOG --> PEND_END
    PEND_CANCEL --> PEND_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    class PEND_START,PEND_END startend;
    class PEND_LIST,PEND_RESEND,PEND_NEW,PEND_LOG,PEND_CANCEL process;
    class PEND_ACTION,PEND_VALID decision;
```

## 6. Flow Penugasan Dokumen oleh User Induk

Penugasan merupakan aktivitas User Induk pada `UI-08.5`. Flow ini tidak menambahkan aktivitas baru pada use case Karyawan.

```mermaid
flowchart TD
    ASSIGN_START([User Induk memilih Local File buatannya]) --> ASSIGN_MEMBER[Pilih Karyawan aktif]
    ASSIGN_MEMBER --> ASSIGN_VALIDATE{"Membership aktif dan paket masih berlaku?"}
    ASSIGN_VALIDATE -- "Tidak" --> ASSIGN_DENY[Penugasan ditolak]
    ASSIGN_VALIDATE -- "Ya" --> ASSIGN_SAVE[Simpan Document Assignment]
    ASSIGN_SAVE --> ASSIGN_SCOPE[Batasi hubungan penugasan pada dokumen terpilih]
    ASSIGN_SCOPE --> ASSIGN_LOG[Catat pemberi tugas, anggota, dokumen, dan waktu]
    ASSIGN_LOG --> ASSIGN_END([Penugasan tersimpan])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class ASSIGN_START,ASSIGN_END startend;
    class ASSIGN_MEMBER,ASSIGN_SAVE,ASSIGN_SCOPE,ASSIGN_LOG process;
    class ASSIGN_VALIDATE decision;
    class ASSIGN_DENY exception;
```

## 7. Flow Status Akses Karyawan

```mermaid
flowchart TD
    ACCESS_START([Karyawan melihat status akses]) --> ACCESS_COMPANY[Tampilkan perusahaan induk]
    ACCESS_COMPANY --> ACCESS_PLAN[Tampilkan paket yang diwarisi]
    ACCESS_PLAN --> ACCESS_MEMBER[Tampilkan status Membership]
    ACCESS_MEMBER --> ACCESS_PERIOD[Tampilkan tanggal berakhir akses]
    ACCESS_PERIOD --> ACCESS_ACTIVE{"Membership dan paket aktif?"}
    ACCESS_ACTIVE -- "Ya" --> ACCESS_AVAILABLE[Akses paket perusahaan tetap tersedia]
    ACCESS_ACTIVE -- "Tidak" --> ACCESS_BASIC[Cabut fitur berbayar; akun pribadi tetap tersedia]
    ACCESS_AVAILABLE --> ACCESS_END([Selesai])
    ACCESS_BASIC --> ACCESS_END

    classDef startend fill:#dcfce7,stroke:#16a34a,color:#052e16,stroke-width:2px;
    classDef process fill:#f0fdf4,stroke:#22c55e,color:#052e16;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    class ACCESS_START,ACCESS_END startend;
    class ACCESS_COMPANY,ACCESS_PLAN,ACCESS_MEMBER,ACCESS_PERIOD,ACCESS_AVAILABLE,ACCESS_BASIC process;
    class ACCESS_ACTIVE decision;
```

## 8. Flow Menghapus Karyawan

```mermaid
flowchart TD
    REMOVE_START([User Induk memilih Karyawan aktif]) --> REMOVE_CONFIRM[Konfirmasi pencabutan akses]
    REMOVE_CONFIRM --> REMOVE_OK{"Penghapusan dikonfirmasi?"}
    REMOVE_OK -- "Tidak" --> REMOVE_END([Tidak ada perubahan])
    REMOVE_OK -- "Ya" --> REMOVE_MEMBER[Ubah Membership menjadi Revoked]
    REMOVE_MEMBER --> REMOVE_SESSION[Cabut akses organisasi dan sesi terkait]
    REMOVE_SESSION --> REMOVE_ASSIGN[Nonaktifkan seluruh Document Assignment anggota]
    REMOVE_ASSIGN --> REMOVE_TAKEOVER[Alihkan akses Local File privat buatan Karyawan kepada User Induk; catat audit]
    REMOVE_TAKEOVER --> REMOVE_SEAT[Bebaskan satu kursi organisasi]
    REMOVE_SEAT --> REMOVE_KEEP[Pertahankan dokumen dan histori perubahan sebagai milik perusahaan]
    REMOVE_KEEP --> REMOVE_ACCOUNT[Pertahankan akun pribadi Karyawan]
    REMOVE_ACCOUNT --> REMOVE_NOTIFY[Kirim pemberitahuan pencabutan akses]
    REMOVE_NOTIFY --> REMOVE_DONE([Karyawan berhasil dikeluarkan])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    class REMOVE_START,REMOVE_END,REMOVE_DONE startend;
    class REMOVE_CONFIRM,REMOVE_MEMBER,REMOVE_SESSION,REMOVE_ASSIGN,REMOVE_TAKEOVER,REMOVE_SEAT,REMOVE_KEEP,REMOVE_ACCOUNT,REMOVE_NOTIFY process;
    class REMOVE_OK decision;
```

## 9. Status Membership dan Invitation

### 9.1 Status Invitation

| Status | Arti |
| --- | --- |
| `Pending` | Undangan telah dibuat dan menunggu keputusan penerima |
| `Used` | Undangan diterima dan membership berhasil dibuat |
| `Rejected` | Calon Karyawan menolak bergabung |
| `Cancelled` | Undangan dibatalkan oleh User Induk |
| `Expired` | Masa berlaku token berakhir |

### 9.2 Status Membership

| Status | Arti |
| --- | --- |
| `Active` | Anggota terhubung dan akses mengikuti paket perusahaan |
| `Pending` | Proses aktivasi belum selesai |
| `Revoked` | Akses dicabut oleh User Induk |
| `Expired` | Akses berbayar berakhir karena paket perusahaan tidak aktif |

## 10. Aturan Bisnis

- Hanya User Induk yang dapat mengelola anggota perusahaan.
- Peran User Induk dimiliki oleh pembuat organisasi pada seluruh paket, tetapi hak multi-pengguna mengikuti paket aktif.
- User Induk Free/Trial dan Basic tidak dapat mengirim undangan atau membeli add-on kursi.
- Administrator tidak dapat mengundang, menghapus, atau menugaskan Karyawan.
- Professional menyediakan tiga pengguna total: satu User Induk dan maksimal dua Karyawan sebelum add-on kursi.
- Enterprise/Custom mengikuti kuota kursi pada quotation dan kontrak.
- Satu akun Karyawan hanya dapat menjadi anggota satu perusahaan pada satu waktu.
- Link undangan hanya berlaku bagi email tujuan dan tidak dapat digunakan akun lain.
- Token yang digunakan, dibatalkan, atau kedaluwarsa tidak dapat dipakai kembali.
- Masa akses Karyawan selalu mengikuti tanggal berakhir paket perusahaan.
- Karyawan tidak memiliki masa Professional tersendiri.
- Berakhirnya paket perusahaan mencabut fitur berbayar Karyawan tanpa menghapus akun pribadi.
- Penghapusan Karyawan membebaskan kursi, tetapi tidak menghapus dokumen atau histori perusahaan.
- Pencabutan membership pembuat menyerahkan akses Local File privat dan request konsultasi yang terkait kepada User Induk dengan audit. Sebelum pengambilalihan, User Induk tidak melihat metadata atau isi dokumen privat.
- User Induk hanya menugaskan Local File buatannya. Berbagi dokumen buatan Karyawan kepada User Induk diubah hanya oleh pembuat aktif dan tidak dapat diteruskan kepada Karyawan lain.
- Seluruh undangan, penerimaan, pembatalan, penugasan, dan pencabutan akses dicatat pada audit log.

## 11. Hubungan Organisasi TP Docs dengan Xendit

`Organization` pada TP Docs adalah batas tenant, pemilik dokumen, langganan, anggota, dan request konsultasi. Organization **bukan** sub-account Xendit karena perusahaan pelanggan bertindak sebagai pembeli layanan, bukan merchant penerima dana.

- pembayaran paket, add-on, Review Konsultan TP, dan Penyusunan Penuh masuk ke akun merchant TP Docs;
- Konsultan merupakan akun internal platform dan bukan penerima payout langsung melalui request pelanggan;
- membership User Induk/Karyawan serta assignment Konsultan tidak dikelola Xendit;
- satu customer Xendit bertipe `BUSINESS` boleh dipetakan ke satu organisasi sebagai identitas billing;
- `organization_id` tetap menjadi sumber kebenaran tenant; provider customer ID hanya referensi integrasi;
- NPWP, isi Local File, komentar review, dan data transaksi perpajakan tidak dikirim ke Xendit.

```mermaid
flowchart LR
    ORG[Organization TP Docs] --> CUSTOMER[Xendit BUSINESS Customer opsional]
    ORG --> MEMBERS[Owner dan Employee Membership]
    ORG --> DOCS[Local File]
    ORG --> ORDERS[Subscription dan Consultation Orders]
    ORDERS --> SESSIONS[Xendit Payment Sessions]

    CONSULTANT[Konsultan Internal] --> ASSIGNMENT[Consultation Assignment]
    ASSIGNMENT --> ORDERS
    CONSULTANT -. "Bukan sub-account atau payout recipient otomatis" .-> CUSTOMER
```

XenPlatform Accounts tidak digunakan pada MVP. Fitur tersebut baru relevan apabila TP Docs kelak menjadi marketplace dan secara kontraktual membagi dana kepada Konsultan atau organisasi lain.

Referensi Xendit:

- [Customers API — Get Customer](https://docs.xendit.co/apidocs/get-customer-id)
- [XenPlatform Accounts V3 Migration Guide](https://docs.xendit.co/docs/xenplatform-v3-api-migration-guide)

## 12. Referensi

- [Business Flow Overview](./00-Business-Flow-Overview.md)
- [Authentication](./01-Authentication.md)
- [Subscription and Billing](./06-Subscription-Billing.md)
- [Use Case User Induk](../02-Use-Case/Organization-Owner.md)
- [Use Case Karyawan](../02-Use-Case/Employee.md)

---

## Ekstensi Business Flow Layanan Konsultan

Flow membership organisasi telah dijelaskan lengkap pada bagian sebelumnya. Konsultan tetap merupakan role platform dan bukan membership organisasi.

### Batas Konsultan

- Konsultan tidak dibuat sebagai `membership` organisasi.
- Akses berasal dari `consultation_assignment` aktif.
- Assignment membawa organization, request, document version, scope, serta expiry.
- Konsultan tidak menggunakan seat Professional atau Enterprise/Custom pelanggan.
- Penghapusan Karyawan tidak menghapus consultation request, tetapi akses Karyawan terhadap request dievaluasi ulang.
