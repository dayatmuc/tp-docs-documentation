# Subscription and Billing Business Flow — TP Docs (With Consultant)

## 1. Tujuan dan Cakupan

Dokumen ini menjelaskan business flow paket langganan organisasi, pembayaran, masa aktif, kuota dokumen, kursi pengguna, add-on, Enterprise/Custom, histori transaksi, dan refund.

Aktor yang terlibat:

- **User Induk** memilih paket, melakukan pembayaran, membeli add-on, mengajukan Enterprise/Custom, dan mengajukan refund.
- **Administrator** memelihara katalog, memantau pembayaran, memproses Enterprise/Custom, dan memproses refund terotorisasi.
- **Sistem TP Docs** menghitung periode aktif, kuota, kursi, status transaksi, dan hak akses organisasi.

Karyawan hanya mewarisi akses dari perusahaan dan tidak dapat membeli paket, add-on, atau melihat informasi pembayaran.

Referensi use case: `UI-08.1` sampai `UI-08.8`, `UA-05`, `UA-06.5`, dan `UK-02`.

## 2. Katalog dan Aturan Paket

| Paket | Periode | Kuota dokumen | Kuota pengguna | Add-on dokumen | Add-on kursi |
| --- | --- | --- | --- | --- | --- |
| Free/Trial | Selama masa akun | 1 dokumen | 1 User Induk | Tidak tersedia | Tidak tersedia |
| Basic | 12 bulan sejak aktif | 5 dokumen | 1 User Induk | Tidak tersedia | Tidak tersedia |
| Professional | 12 bulan sejak aktif | 10 dokumen | 3 pengguna total | Rp500.000 per dokumen | Tersedia secara prorata |
| Enterprise/Custom | Sesuai quotation | Sesuai quotation | Sesuai quotation | Sesuai kontrak | Sesuai kontrak |

Periode Basic dan Professional dihitung dari tanggal pembayaran terverifikasi dan paket diaktifkan, bukan berdasarkan tahun kalender.

## 3. Flow Memilih Paket

```mermaid
flowchart TD
    PLAN_START([User Induk membuka halaman Langganan]) --> PLAN_VIEW[Tampilkan harga, kuota dokumen, pengguna, fitur, dan periode]
    PLAN_VIEW --> PLAN_SELECT{"Pilih paket"}
    PLAN_SELECT -- "Free/Trial" --> PLAN_FREE[Aktifkan 1 dokumen dan 1 User Induk]
    PLAN_FREE --> PLAN_FREE_RULE[Tidak dapat membeli add-on atau menggunakan multi-pengguna]
    PLAN_FREE_RULE --> PLAN_END([Paket tersedia])
    PLAN_SELECT -- "Basic" --> PLAN_BASIC[Siapkan pesanan Basic]
    PLAN_BASIC --> PLAN_CHECKOUT[Lanjut ke checkout paket reguler]
    PLAN_SELECT -- "Professional" --> PLAN_PRO[Siapkan pesanan Professional]
    PLAN_PRO --> PLAN_CHECKOUT
    PLAN_SELECT -- "Enterprise/Custom" --> PLAN_CUSTOM[Arahkan ke form kebutuhan khusus]
    PLAN_CUSTOM --> PLAN_CUSTOM_FLOW([Lanjut ke Flow Enterprise/Custom])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    class PLAN_START,PLAN_END,PLAN_CUSTOM_FLOW startend;
    class PLAN_VIEW,PLAN_FREE,PLAN_FREE_RULE,PLAN_BASIC,PLAN_PRO,PLAN_CHECKOUT,PLAN_CUSTOM process;
    class PLAN_SELECT decision;
```

## 4. Flow Checkout Paket Reguler

Flow checkout berlaku untuk Basic dan Professional. Enterprise/Custom tidak melakukan checkout otomatis sebelum quotation diterima.

```mermaid
flowchart TD
    PAY_START([Pesanan Basic atau Professional dibuat]) --> PAY_SUMMARY[Tampilkan paket, harga, masa 12 bulan, kuota, dan total]
    PAY_SUMMARY --> PAY_CONFIRM[User Induk mengonfirmasi pesanan]
    PAY_CONFIRM --> PAY_CREATE[Sistem membuat transaksi berstatus Pending]
    PAY_CREATE --> PAY_GATEWAY[Arahkan pembayaran melalui payment gateway]
    PAY_GATEWAY --> PAY_RESULT{"Status pembayaran"}
    PAY_RESULT -- "Pending" --> PAY_WAIT[Tampilkan instruksi dan batas waktu pembayaran]
    PAY_WAIT --> PAY_SYNC[Sistem memeriksa pembaruan status]
    PAY_SYNC --> PAY_RESULT
    PAY_RESULT -- "Failed / Expired" --> PAY_FAIL[Tandai transaksi gagal atau kedaluwarsa]
    PAY_FAIL --> PAY_RETRY[User Induk dapat membuat pembayaran baru]
    PAY_RESULT -- "Paid" --> PAY_VERIFY[Verifikasi referensi, nilai, dan keunikan transaksi]
    PAY_VERIFY --> PAY_VALID{"Pembayaran valid dan belum diproses?"}
    PAY_VALID -- "Tidak" --> PAY_HOLD[Tahan aktivasi untuk pemeriksaan]
    PAY_VALID -- "Ya" --> PAY_ACTIVATE[Aktifkan atau upgrade paket organisasi]
    PAY_ACTIVATE --> PAY_PERIOD[Tetapkan periode 12 bulan sejak aktivasi]
    PAY_PERIOD --> PAY_QUOTA[Terapkan kuota dokumen dan kursi paket]
    PAY_QUOTA --> PAY_HISTORY[Catat pembayaran dan perubahan langganan]
    PAY_HISTORY --> PAY_NOTIFY[Kirim konfirmasi kepada User Induk]
    PAY_NOTIFY --> PAY_END([Paket aktif])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class PAY_START,PAY_END startend;
    class PAY_SUMMARY,PAY_CONFIRM,PAY_CREATE,PAY_GATEWAY,PAY_WAIT,PAY_SYNC,PAY_RETRY,PAY_VERIFY,PAY_ACTIVATE,PAY_PERIOD,PAY_QUOTA,PAY_HISTORY,PAY_NOTIFY process;
    class PAY_RESULT,PAY_VALID decision;
    class PAY_FAIL,PAY_HOLD exception;
```

Aktivasi harus bersifat idempotent: satu transaksi pembayaran hanya boleh mengaktifkan paket atau menambah hak akses satu kali.

## 5. Flow Upgrade Paket Reguler

Flow ini digunakan ketika User Induk meningkatkan paket organisasi, termasuk dari Free/Trial ke Basic atau Professional serta dari Basic ke Professional. Upgrade ke Enterprise/Custom mengikuti flow pengajuan dan quotation pada bagian tersendiri.

```mermaid
flowchart TD
    UPGRADE_START([User Induk membuka paket dan masa aktif]) --> UPGRADE_CURRENT[Tampilkan paket aktif, periode, kuota, kursi, dan penggunaan]
    UPGRADE_CURRENT --> UPGRADE_SELECT{"Pilih paket tujuan"}
    UPGRADE_SELECT -- "Paket sama atau lebih rendah" --> UPGRADE_INVALID[Upgrade tidak diproses melalui flow ini]
    UPGRADE_SELECT -- "Enterprise/Custom" --> UPGRADE_CUSTOM[Arahkan ke form kebutuhan Enterprise/Custom]
    UPGRADE_SELECT -- "Basic atau Professional yang lebih tinggi" --> UPGRADE_CALCULATE[Sistem menghitung harga dan ketentuan upgrade]
    UPGRADE_CALCULATE --> UPGRADE_SUMMARY[Tampilkan biaya, waktu berlaku, perubahan kuota, kursi, dan fitur]
    UPGRADE_SUMMARY --> UPGRADE_CONFIRM{"User Induk mengonfirmasi?"}
    UPGRADE_CONFIRM -- "Tidak" --> UPGRADE_CANCEL([Upgrade dibatalkan])
    UPGRADE_CONFIRM -- "Ya" --> UPGRADE_ORDER[Buat transaksi upgrade berstatus Pending]
    UPGRADE_ORDER --> UPGRADE_PAYMENT[Proses pembayaran melalui payment gateway]
    UPGRADE_PAYMENT --> UPGRADE_PAID{"Pembayaran terverifikasi?"}
    UPGRADE_PAID -- "Tidak" --> UPGRADE_FAILED[Paket lama tetap aktif tanpa perubahan hak akses]
    UPGRADE_PAID -- "Ya" --> UPGRADE_APPLY[Terapkan paket tujuan sesuai ketentuan yang ditampilkan]
    UPGRADE_APPLY --> UPGRADE_ENTITLEMENT[Perbarui kuota dokumen, kursi, fitur, dan periode]
    UPGRADE_ENTITLEMENT --> UPGRADE_MULTI{"Professional aktif?"}
    UPGRADE_MULTI -- "Ya" --> UPGRADE_MEMBER[Aktifkan pengelolaan anggota dan add-on yang tersedia]
    UPGRADE_MULTI -- "Tidak" --> UPGRADE_HISTORY
    UPGRADE_MEMBER --> UPGRADE_HISTORY[Catat pembayaran dan perubahan paket]
    UPGRADE_HISTORY --> UPGRADE_NOTIFY[Kirim konfirmasi upgrade]
    UPGRADE_NOTIFY --> UPGRADE_END([Paket baru tersedia])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class UPGRADE_START,UPGRADE_CANCEL,UPGRADE_END startend;
    class UPGRADE_CURRENT,UPGRADE_CUSTOM,UPGRADE_CALCULATE,UPGRADE_SUMMARY,UPGRADE_ORDER,UPGRADE_PAYMENT,UPGRADE_APPLY,UPGRADE_ENTITLEMENT,UPGRADE_MEMBER,UPGRADE_HISTORY,UPGRADE_NOTIFY process;
    class UPGRADE_SELECT,UPGRADE_CONFIRM,UPGRADE_PAID,UPGRADE_MULTI decision;
    class UPGRADE_INVALID,UPGRADE_FAILED exception;
```

Ketika Basic berhasil ditingkatkan ke Professional, organisasi memperoleh kapasitas tiga pengguna total—satu User Induk dan maksimal dua Karyawan—serta dapat mulai mengirim undangan. Harga upgrade, perlakuan sisa masa Basic, dan tanggal periode baru harus ditampilkan sebelum pembayaran sesuai kebijakan upgrade yang dikonfigurasi.

## 6. Flow Kuota Dokumen

```mermaid
flowchart TD
    QUOTA_START([User Induk membuat dokumen baru]) --> QUOTA_STATUS[Ambil paket, periode aktif, dan status organisasi]
    QUOTA_STATUS --> QUOTA_ACTIVE{"Paket dapat digunakan?"}
    QUOTA_ACTIVE -- "Tidak" --> QUOTA_BLOCK[Tolak pembuatan dan arahkan ke langganan]
    QUOTA_ACTIVE -- "Ya" --> QUOTA_PACKAGE{"Sisa kuota paket tersedia?"}
    QUOTA_PACKAGE -- "Ya" --> QUOTA_CREATE[Buat dokumen baru]
    QUOTA_PACKAGE -- "Tidak" --> QUOTA_ADDON{"Sisa kuota add-on tersedia?"}
    QUOTA_ADDON -- "Ya" --> QUOTA_CREATE
    QUOTA_ADDON -- "Tidak" --> QUOTA_PLAN{"Paket aktif"}
    QUOTA_PLAN -- "Free/Trial" --> QUOTA_UPGRADE[Arahkan upgrade untuk dokumen berikutnya]
    QUOTA_PLAN -- "Basic" --> QUOTA_PRO[Arahkan upgrade ke Professional]
    QUOTA_PLAN -- "Professional" --> QUOTA_BUY[Tawarkan add-on satu dokumen]
    QUOTA_PLAN -- "Enterprise/Custom" --> QUOTA_CONTRACT[Tampilkan batas sesuai quotation]
    QUOTA_CREATE --> QUOTA_SUCCESS{"Dokumen berhasil dibuat?"}
    QUOTA_SUCCESS -- "Tidak" --> QUOTA_KEEP[Kuota tidak dikurangi]
    QUOTA_SUCCESS -- "Ya" --> QUOTA_DEDUCT[Kurangi kuota paket terlebih dahulu, lalu add-on]
    QUOTA_DEDUCT --> QUOTA_USAGE[Catat Document Quota Usage]
    QUOTA_USAGE --> QUOTA_END([Dokumen tersedia])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class QUOTA_START,QUOTA_END startend;
    class QUOTA_STATUS,QUOTA_CREATE,QUOTA_UPGRADE,QUOTA_PRO,QUOTA_BUY,QUOTA_CONTRACT,QUOTA_DEDUCT,QUOTA_USAGE process;
    class QUOTA_ACTIVE,QUOTA_PACKAGE,QUOTA_ADDON,QUOTA_PLAN,QUOTA_SUCCESS decision;
    class QUOTA_BLOCK,QUOTA_KEEP exception;
```

Kuota hanya digunakan setelah dokumen berhasil dibuat. Dokumen yang sudah ada tidak dihapus ketika kuota habis atau paket berakhir.

## 7. Flow Pembelian Add-on Dokumen Professional

```mermaid
flowchart TD
    DOC_START([User Induk memilih Tambah Kuota Dokumen]) --> DOC_PLAN{"Paket Professional aktif?"}
    DOC_PLAN -- "Tidak" --> DOC_DENY[Add-on dokumen tidak tersedia]
    DOC_PLAN -- "Ya" --> DOC_PRICE[Tampilkan Rp500.000 untuk satu dokumen dan masa berlaku]
    DOC_PRICE --> DOC_CONFIRM[Konfirmasi pembelian]
    DOC_CONFIRM --> DOC_TRANSACTION[Buat transaksi add-on berstatus Pending]
    DOC_TRANSACTION --> DOC_PAYMENT[Proses melalui payment gateway]
    DOC_PAYMENT --> DOC_RESULT{"Pembayaran terverifikasi?"}
    DOC_RESULT -- "Tidak" --> DOC_FAILED[Tidak menambah kuota]
    DOC_RESULT -- "Ya" --> DOC_IDEMPOTENT{"Transaksi belum pernah diterapkan?"}
    DOC_IDEMPOTENT -- "Tidak" --> DOC_DUPLICATE[Abaikan callback duplikat]
    DOC_IDEMPOTENT -- "Ya" --> DOC_ADD[Tambahkan tepat satu kuota dokumen]
    DOC_ADD --> DOC_EXPIRY[Samakan masa berlaku dengan akhir periode Professional]
    DOC_EXPIRY --> DOC_HISTORY[Catat histori add-on]
    DOC_HISTORY --> DOC_END([Kuota tambahan tersedia])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class DOC_START,DOC_END startend;
    class DOC_PRICE,DOC_CONFIRM,DOC_TRANSACTION,DOC_PAYMENT,DOC_ADD,DOC_EXPIRY,DOC_HISTORY process;
    class DOC_PLAN,DOC_RESULT,DOC_IDEMPOTENT decision;
    class DOC_DENY,DOC_FAILED,DOC_DUPLICATE exception;
```

Sisa add-on dokumen tidak dibawa ke periode langganan berikutnya.

## 8. Flow Pembelian Add-on Kursi Professional

```mermaid
flowchart TD
    SEAT_START([User Induk membutuhkan Karyawan tambahan]) --> SEAT_PLAN{"Professional aktif?"}
    SEAT_PLAN -- "Tidak" --> SEAT_DENY[Arahkan ke paket yang mendukung multi-pengguna]
    SEAT_PLAN -- "Ya" --> SEAT_PERIOD[Hitung sisa hari periode Professional]
    SEAT_PERIOD --> SEAT_PRORATE[Hitung harga kursi secara prorata]
    SEAT_PRORATE --> SEAT_CONFIRM[User Induk mengonfirmasi pembelian]
    SEAT_CONFIRM --> SEAT_PAYMENT[Proses pembayaran add-on kursi]
    SEAT_PAYMENT --> SEAT_VALID{"Pembayaran terverifikasi?"}
    SEAT_VALID -- "Tidak" --> SEAT_FAILED[Kursi tidak ditambahkan]
    SEAT_VALID -- "Ya" --> SEAT_ADD[Tambahkan satu kursi organisasi]
    SEAT_ADD --> SEAT_EXPIRY[Samakan masa berlaku dengan paket induk]
    SEAT_EXPIRY --> SEAT_HISTORY[Catat transaksi dan perubahan kursi]
    SEAT_HISTORY --> SEAT_INVITE[User Induk dapat melanjutkan undangan Karyawan]
    SEAT_INVITE --> SEAT_END([Kursi tersedia])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class SEAT_START,SEAT_END startend;
    class SEAT_PERIOD,SEAT_PRORATE,SEAT_CONFIRM,SEAT_PAYMENT,SEAT_ADD,SEAT_EXPIRY,SEAT_HISTORY,SEAT_INVITE process;
    class SEAT_PLAN,SEAT_VALID decision;
    class SEAT_DENY,SEAT_FAILED exception;
```

## 9. Flow Enterprise/Custom

Enterprise/Custom menggunakan form, review Administrator, dan quotation. Sistem tidak membuka Gmail atau meminta User Induk mengirim email secara manual.

```mermaid
flowchart TD
    CUSTOM_START([User Induk memilih Enterprise/Custom]) --> CUSTOM_FORM[Isi perusahaan, PIC, kontak, entitas, pengguna, dan dokumen per tahun]
    CUSTOM_FORM --> CUSTOM_FEATURE[Isi multi-entitas, SSO, lokasi data, dukungan, dan catatan]
    CUSTOM_FEATURE --> CUSTOM_CONSENT[Setujui persetujuan untuk dihubungi]
    CUSTOM_CONSENT --> CUSTOM_VALIDATE{"Form lengkap dan valid?"}
    CUSTOM_VALIDATE -- "Tidak" --> CUSTOM_FIX[Tampilkan field yang harus diperbaiki]
    CUSTOM_FIX --> CUSTOM_FORM
    CUSTOM_VALIDATE -- "Ya" --> CUSTOM_SUBMIT[Kirim permintaan]
    CUSTOM_SUBMIT --> CUSTOM_TICKET[Buat nomor tiket dan status Submitted]
    CUSTOM_TICKET --> CUSTOM_EMAIL[Kirim notifikasi sales dan konfirmasi pengguna]
    CUSTOM_EMAIL --> CUSTOM_REVIEW[Administrator mengubah status Under Review]
    CUSTOM_REVIEW --> CUSTOM_FEASIBLE{"Kebutuhan dapat dipenuhi?"}
    CUSTOM_FEASIBLE -- "Perlu revisi" --> CUSTOM_REVISION[Status Revision Requested]
    CUSTOM_REVISION --> CUSTOM_UPDATE[User Induk memperbaiki dan mengirim ulang]
    CUSTOM_UPDATE --> CUSTOM_REVIEW
    CUSTOM_FEASIBLE -- "Tidak" --> CUSTOM_REJECT[Status Rejected dan kirim alasan]
    CUSTOM_FEASIBLE -- "Ya" --> CUSTOM_QUOTE[Administrator menyusun quotation; status Quoted]
    CUSTOM_QUOTE --> CUSTOM_DECISION{"User Induk menerima quotation?"}
    CUSTOM_DECISION -- "Tidak" --> CUSTOM_CLOSED[Status Closed]
    CUSTOM_DECISION -- "Ya" --> CUSTOM_INVOICE[Buat tagihan; status Awaiting Payment]
    CUSTOM_INVOICE --> CUSTOM_PAY[User Induk melakukan pembayaran]
    CUSTOM_PAY --> CUSTOM_VERIFIED{"Pembayaran terverifikasi?"}
    CUSTOM_VERIFIED -- "Tidak" --> CUSTOM_WAIT[Tetap Awaiting Payment atau tangani pembayaran gagal]
    CUSTOM_WAIT --> CUSTOM_PAY
    CUSTOM_VERIFIED -- "Ya" --> CUSTOM_CONFIG[Administrator menerapkan konfigurasi sesuai quotation]
    CUSTOM_CONFIG --> CUSTOM_ACTIVE[Status Active dan kirim notifikasi aktivasi]
    CUSTOM_ACTIVE --> CUSTOM_END([Enterprise/Custom tersedia])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class CUSTOM_START,CUSTOM_END startend;
    class CUSTOM_FORM,CUSTOM_FEATURE,CUSTOM_CONSENT,CUSTOM_SUBMIT,CUSTOM_TICKET,CUSTOM_EMAIL,CUSTOM_REVIEW,CUSTOM_REVISION,CUSTOM_UPDATE,CUSTOM_QUOTE,CUSTOM_INVOICE,CUSTOM_PAY,CUSTOM_WAIT,CUSTOM_CONFIG,CUSTOM_ACTIVE process;
    class CUSTOM_VALIDATE,CUSTOM_FEASIBLE,CUSTOM_DECISION,CUSTOM_VERIFIED decision;
    class CUSTOM_FIX,CUSTOM_REJECT,CUSTOM_CLOSED exception;
```

Status Enterprise/Custom yang digunakan bersama oleh User Induk dan Administrator:

`Submitted`, `Under Review`, `Revision Requested`, `Quoted`, `Awaiting Payment`, `Active`, `Rejected`, dan `Closed`.

## 10. Flow Refund

```mermaid
flowchart TD
    REF_START([User Induk membuka Riwayat Pembayaran]) --> REF_SELECT[Pilih transaksi yang memenuhi syarat]
    REF_SELECT --> REF_POLICY[Tampilkan ketentuan, estimasi nilai, dan dampak refund]
    REF_POLICY --> REF_REASON[Isi kategori dan penjelasan alasan]
    REF_REASON --> REF_PROOF{"Perlu bukti pendukung?"}
    REF_PROOF -- "Ya" --> REF_UPLOAD[Unggah bukti pembayaran atau dokumen pendukung]
    REF_PROOF -- "Tidak" --> REF_IMPACT
    REF_UPLOAD --> REF_IMPACT[Konfirmasi dampak pada paket, periode, kursi, kuota, atau add-on]
    REF_IMPACT --> REF_SEND[Kirim pengajuan dan buat nomor referensi]
    REF_SEND --> REF_SUBMITTED[Status Submitted]
    REF_SUBMITTED -. "handoff" .-> REF_ADMIN[Administrator mengubah status Under Review]
    REF_ADMIN --> REF_VALIDATE[Validasi transaksi, alasan, bukti, nilai, dan dampak]
    REF_VALIDATE --> REF_APPROVAL[Meminta persetujuan Administrator kedua]
    REF_APPROVAL --> REF_ELIGIBLE{"Refund disetujui?"}
    REF_ELIGIBLE -- "Tidak" --> REF_REJECT[Status Rejected; paket dan kuota tidak berubah]
    REF_ELIGIBLE -- "Ya" --> REF_APPROVED[Status Approved lalu Processing]
    REF_APPROVED --> REF_GATEWAY[Proses ke metode pembayaran asal]
    REF_GATEWAY --> REF_SUCCESS{"Pengembalian berhasil?"}
    REF_SUCCESS -- "Tidak" --> REF_FAILED[Status Failed; tidak melakukan penyesuaian ganda]
    REF_SUCCESS -- "Ya" --> REF_ADJUST[Status Refunded dan sesuaikan akses tepat satu kali]
    REF_ADJUST --> REF_NOTIFY[Kirim hasil kepada User Induk]
    REF_REJECT --> REF_NOTIFY
    REF_FAILED --> REF_NOTIFY
    REF_NOTIFY --> REF_END([User Induk melihat hasil refund])

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    classDef exception fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    class REF_START,REF_END startend;
    class REF_SELECT,REF_POLICY,REF_REASON,REF_UPLOAD,REF_IMPACT,REF_SEND,REF_SUBMITTED,REF_ADMIN,REF_VALIDATE,REF_APPROVAL,REF_APPROVED,REF_GATEWAY,REF_ADJUST,REF_NOTIFY process;
    class REF_PROOF,REF_ELIGIBLE,REF_SUCCESS decision;
    class REF_REJECT,REF_FAILED exception;
```

Status refund yang sama digunakan pada kedua sisi:

`Submitted`, `Under Review`, `Approved`, `Rejected`, `Processing`, `Refunded`, dan `Failed`.

## 11. Flow Masa Aktif dan Berakhirnya Paket

```mermaid
flowchart TD
    EXP_START([Sistem memeriksa periode langganan]) --> EXP_DATE{"Tanggal akhir telah tercapai?"}
    EXP_DATE -- "Tidak" --> EXP_ACTIVE[Paket, kuota, add-on, dan membership tetap aktif]
    EXP_DATE -- "Ya" --> EXP_STATUS[Ubah status langganan menjadi Expired]
    EXP_STATUS --> EXP_CREATE[Blokir pembuatan dokumen baru dan pembelian add-on periode lama]
    EXP_CREATE --> EXP_MEMBER[Cabut fitur berbayar seluruh Karyawan]
    EXP_MEMBER --> EXP_KEEP[Pertahankan akun, dokumen, histori, dan data perusahaan]
    EXP_KEEP --> EXP_RECOVERY[Mulai masa pemulihan data sesuai kebijakan retensi]
    EXP_RECOVERY --> EXP_NOTIFY[Kirim pemberitahuan kepada User Induk]
    EXP_ACTIVE --> EXP_END([Selesai])
    EXP_NOTIFY --> EXP_END

    classDef startend fill:#dbeafe,stroke:#2563eb,color:#0f172a,stroke-width:2px;
    classDef process fill:#eff6ff,stroke:#3b82f6,color:#0f172a;
    classDef decision fill:#fef3c7,stroke:#d97706,color:#0f172a;
    class EXP_START,EXP_END startend;
    class EXP_ACTIVE,EXP_STATUS,EXP_CREATE,EXP_MEMBER,EXP_KEEP,EXP_RECOVERY,EXP_NOTIFY process;
    class EXP_DATE decision;
```

## 12. Riwayat dan Informasi Langganan

User Induk dapat melihat:

- Paket aktif dan tanggal berakhir.
- Kuota dokumen paket, dokumen terpakai, sisa kuota, dan tanggal reset.
- Kuota add-on dokumen dan masa berlakunya.
- Kursi terpakai dan tersedia.
- Transaksi paket dan add-on.
- Status pembayaran `Pending`, `Paid`, `Failed`, atau `Expired`.
- Status pengajuan Enterprise/Custom.
- Status dan hasil refund.

Perubahan katalog atau koreksi kuota oleh Administrator harus tercermin pada tampilan dan histori User Induk. Laporan atau histori tidak menampilkan kredensial payment gateway maupun data pembayaran mentah.

## 13. Aturan Bisnis

### Diskon Event Opsional

Admin mengatur sasaran, nilai, periode Asia/Jakarta, dan batas penggunaan opsional. Saat checkout paket baru, upgrade, perpanjangan, atau add-on, sistem memilih satu diskon terbesar yang aktif dan sesuai. Jika tidak ada, harga katalog berlaku. Quotation Enterprise/Custom membekukan potongan saat diterbitkan; invoice mengikuti total snapshot hingga quotation kedaluwarsa. Payment gagal melepas reservasi setelah TTL; redemption dicatat sekali saat PAID. Refund mengacu pada jumlah dibayar. Free/Trial tidak mendapat diskon.

~~~mermaid
flowchart TD
 P[Harga normal] --> E{Event aktif sesuai?}
 E -->|Tidak| N[Harga normal]
 E -->|Ya| D[Satu diskon terbesar]
 N --> S[Snapshot harga]
 D --> S
 S --> X[Checkout atau quotation]
 X --> V{Pembayaran berhasil?}
 V -->|Ya| R[Catat penggunaan promo]
 V -->|Tidak| F[Lepas reservasi saat TTL]
~~~


- Hanya User Induk yang dapat membeli paket, add-on, atau mengajukan refund.
- User Induk tetap menjadi pemilik organisasi pada Free/Trial dan Basic, tetapi tidak dapat mengundang Karyawan.
- Upgrade Basic ke Professional mengaktifkan hak multi-pengguna setelah pembayaran terverifikasi.
- Pembayaran upgrade yang gagal tidak mengubah paket, kuota, kursi, periode, atau hak akses sebelumnya.
- Karyawan mewarisi fitur dan masa aktif organisasi tanpa memiliki langganan terpisah.
- Basic dan Professional berlaku 12 bulan sejak paket diaktifkan.
- Sistem menggunakan kuota paket sebelum kuota add-on.
- Kuota berkurang hanya setelah dokumen baru berhasil dibuat.
- Add-on Professional berakhir bersama periode paket yang sedang berjalan.
- Sisa add-on tidak dibawa ke periode berikutnya.
- Pembayaran gagal tidak mengaktifkan paket dan tidak menambah kuota atau kursi.
- Callback pembayaran yang sama tidak boleh menerapkan hak akses lebih dari satu kali.
- Enterprise/Custom tidak aktif sebelum quotation diterima dan pembayaran terverifikasi.
- Refund yang ditolak tidak mengubah paket, periode, kursi, kuota, atau add-on.
- Refund yang gagal tetap tercatat dan tidak boleh menghasilkan pengembalian maupun penyesuaian akses ganda.
- Pengembalian dana dilakukan melalui metode pembayaran asal.
- Formula prorata kursi, kelayakan refund, batas waktu refund, dan formula nilai refund mengikuti kebijakan bisnis yang masih perlu divalidasi.

## 14. Referensi

- [Business Flow Overview](./00-Business-Flow-Overview.md)
- [Membership and Organization](./05-Membership-Organization.md)
- [Use Case User Induk](../02-Use-Case/Organization-Owner.md)
- [Use Case Karyawan](../02-Use-Case/Employee.md)
- [Use Case Administrator](../02-Use-Case/Admin.md)
- [Functional Requirements](../01-Requirements/Functional-Requirements.md)
- [Non-Functional Requirements](../01-Requirements/Non-Functional-Requirements.md)

---

## 15. Billing Layanan Konsultasi

### 15.1 Pemisahan Langganan Aplikasi dan Jasa Konsultasi

Langganan aplikasi dan jasa konsultasi merupakan dua produk yang berbeda:

| Aspek | Langganan Aplikasi | Jasa Konsultasi |
| --- | --- | --- |
| Objek pembelian | Paket, kursi, kuota, atau add-on | Layanan untuk satu document version |
| Pemilik | Organisasi | Organisasi melalui pemohon yang berwenang |
| Harga | Katalog paket | Katalog jasa atau quotation |
| Masa berlaku | Mengikuti periode paket | Mengikuti request dan SLA snapshot |
| Kuota revisi | Tidak berlaku | Mengikuti snapshot order konsultasi |
| Aktivasi | Setelah payment paket terverifikasi | Setelah payment konsultasi terverifikasi dan assignment dibuat |
| Dampak berakhir | Fitur paket dibatasi | Scoped access Konsultan ditutup setelah request selesai/dicabut |

Pembelian konsultasi tidak menambah kuota dokumen, kursi, atau masa aktif paket. Penggunaan layanan konsultasi juga tidak mengurangi kuota dokumen, kecuali pengguna membuat document version baru melalui proses generate normal yang memang menggunakan aturan kuota dokumen.

### 15.2 Kewenangan Billing Konsultasi

| Aktivitas | User Induk | Karyawan | Konsultan | Administrator |
| --- | --- | --- | --- | --- |
| Melihat katalog aktif | Ya | Ya | Tidak | Kelola |
| Melihat harga layanan | Ya | Ya | Tidak | Kelola |
| Membuat request | Semua dokumen organisasi | Hanya dokumen yang ditugaskan | Tidak | Tidak |
| Menerima quotation | Ya | Jika menjadi pemohon dan diizinkan | Tidak | Membuat quotation |
| Membayar | Ya | Ya, untuk request miliknya | Tidak | Memantau status |
| Melihat transaksi | Seluruh organisasi | Hanya transaksi konsultasi yang dibuatnya | Tidak | Metadata operasional |
| Mengajukan refund | Ya | Untuk transaksi yang dibuatnya | Tidak | Memutuskan dan memproses |
| Melihat nilai payment | Ya | Hanya transaksi miliknya | Tidak | Sesuai kewenangan |
| Mengubah harga/SLA/revisi | Tidak | Tidak | Tidak | Ya, melalui katalog/version baru |

Konsultan hanya menerima informasi bahwa request telah memenuhi syarat pembayaran. Nominal, metode pembayaran, quotation, refund, dan data finansial pengguna tidak ditampilkan pada portal Konsultan.

### 15.3 Lifecycle Katalog Jasa

```mermaid
flowchart TD
    ADMIN([Administrator]) --> CREATE[Buat atau ubah layanan]
    CREATE --> INPUT[Input nama, deskripsi, pricing mode, currency, SLA, dan revisi]
    INPUT --> MODE{Pricing mode}
    MODE -- FIXED --> FIXED[Input display price]
    MODE -- STARTING_FROM --> START[Input minimum price]
    MODE -- QUOTATION --> QUOTE[Harga final ditentukan melalui quotation]
    FIXED --> VALIDATE
    START --> VALIDATE
    QUOTE --> VALIDATE[Validasi konfigurasi]
    VALIDATE --> VALID{Valid?}
    VALID -- Tidak --> ERROR[Tampilkan field yang harus diperbaiki]
    ERROR --> INPUT
    VALID -- Ya --> VERSION[Buat service version immutable]
    VERSION --> ACTIVATE{Aktifkan?}
    ACTIVATE -- Tidak --> DRAFT[Status draft]
    ACTIVATE -- Ya --> ACTIVE[Service version aktif untuk order baru]
    ACTIVE --> CHANGE{Ada perubahan katalog?}
    CHANGE -- Ya --> NEWVERSION[Buat version baru]
    NEWVERSION --> ACTIVE
    CHANGE -- Tidak --> END([Selesai])
```

Aturan katalog:

- Harga, currency, SLA, dan revisi tidak ditulis permanen pada frontend.
- Layanan tidak dapat diaktifkan apabila konfigurasi wajib belum valid.
- Perubahan selalu membuat service version baru.
- Order lama tetap menggunakan snapshot dari version yang dipilih ketika request dibuat.
- Menonaktifkan layanan mencegah order baru tanpa membatalkan request yang sudah sah.
- Seluruh perubahan katalog memiliki identitas Administrator, waktu, alasan, dan audit log.

### 15.4 Mode Harga

| Mode | Tampilan kepada pengguna | Waktu menentukan harga final | Syarat checkout |
| --- | --- | --- | --- |
| `FIXED` | Harga tetap dari `displayPrice` | Saat service version dibuat | Request dan snapshot valid |
| `STARTING_FROM` | Teks “Mulai” dari `minimumPrice` | Setelah Admin membuat quotation | Quotation diterima |
| `QUOTATION` | Keterangan meminta quotation tanpa nominal checkout | Setelah kebutuhan ditinjau Admin | Quotation diterima |

`STARTING_FROM` tidak berarti pengguna membayar minimum price secara otomatis. Minimum price hanya informasi katalog; payment selalu memakai nilai quotation final yang diterima pengguna.

### 15.5 Pembentukan Snapshot Order

Ketika request dibuat, backend mengambil service version aktif dan menyimpan:

- ID layanan dan service version.
- Nama serta deskripsi layanan.
- Pricing mode.
- Currency.
- Harga tetap atau nilai quotation final.
- SLA review awal dan review ulang.
- Jumlah revisi yang termasuk.
- Document version sumber.
- Pemohon, membership, organisasi, dan waktu request.

Frontend tidak mengirim nominal sebagai sumber kebenaran. Nilai yang dikirim browser diabaikan atau diverifikasi ulang terhadap snapshot backend.

### 15.6 Checkout Layanan Fixed

```mermaid
sequenceDiagram
    actor User as User Induk/Karyawan
    participant Web as Next.js
    participant API as NestJS
    participant DB as PostgreSQL
    participant PG as Payment Gateway
    participant Admin as Administrator

    User->>Web: Pilih layanan dan document version
    Web->>API: Buat consultation request
    API->>API: Validasi tenant, assignment, version, dan service aktif
    API->>DB: Simpan request + service snapshot
    API->>PG: Buat invoice dari price snapshot
    PG-->>API: Invoice/payment reference
    API-->>Web: Tampilkan instruksi pembayaran
    PG->>API: Webhook payment
    API->>API: Verifikasi signature dan idempotency
    API->>DB: Payment=PAID, request=PENDING_ASSIGNMENT
    API-->>Admin: Notifikasi request siap ditugaskan
```

Jika payment gagal atau kedaluwarsa:

- request tidak dapat ditugaskan;
- Konsultan tidak memperoleh akses;
- snapshot order tidak berubah;
- pengguna dapat membuat payment attempt baru sesuai kebijakan;
- retry webhook tidak boleh membuat payment atau assignment ganda.

### 15.7 Quotation Penyusunan Penuh

```mermaid
flowchart TD
    USER([User berwenang]) --> REQUEST[Isi kebutuhan dan kirim request]
    REQUEST --> META[Sistem bentuk metadata kompleksitas aman]
    META --> REVIEW[Administrator meninjau kebutuhan]
    REVIEW --> NEED{Informasi cukup?}
    NEED -- Tidak --> REVISION[Status Revision Requested]
    REVISION --> UPDATE[User memperbarui kebutuhan]
    UPDATE --> REVIEW
    NEED -- Ya --> QUOTATION[Admin input amount, currency, terms, dan validity]
    QUOTATION --> QUOTED[Status Quoted]
    QUOTED --> DECISION{Keputusan pengguna}
    DECISION -- Tolak --> CLOSED[Status Closed atau Rejected]
    DECISION -- Kedaluwarsa --> EXPIRED[Quotation tidak dapat dibayar]
    DECISION -- Terima --> AWAIT[Status Awaiting Payment]
    AWAIT --> PAYMENT[Payment dari quotation snapshot]
    PAYMENT --> PAID{Terverifikasi?}
    PAID -- Tidak --> AWAIT
    PAID -- Ya --> ASSIGN[Status Pending Assignment]
```

Administrator menilai metadata kompleksitas tanpa otomatis membaca isi dokumen. Akses isi dokumen baru diberikan kepada Konsultan yang sah setelah payment dan assignment.

### 15.8 Status Payment Konsultasi

| Status | Makna | Dampak pada request |
| --- | --- | --- |
| `PENDING` | Invoice dibuat tetapi belum terverifikasi | Tetap `AWAITING_PAYMENT` |
| `PAID` | Webhook valid dan transaksi berhasil | Menjadi `PENDING_ASSIGNMENT` |
| `FAILED` | Provider menyatakan gagal | Tidak ada akses Konsultan |
| `EXPIRED` | Masa pembayaran habis | Memerlukan payment attempt baru |
| `REFUNDED` | Refund selesai | Akses dan status disesuaikan sekali |

Status payment hanya berubah melalui proses backend yang sah. Redirect browser setelah pembayaran bukan bukti keberhasilan.

### 15.9 Revisi dan Review Tambahan

- Review awal tidak dihitung sebagai revision round.
- Setiap resubmission yang diterima sebagai review ulang menggunakan satu revision round.
- `includedRevisionRounds` dan `usedRevisionRounds` dihitung dari snapshot request.
- Penggunaan revisi harus idempotent dan tidak bertambah dua kali karena retry.
- Jika kuota habis, sistem menampilkan layanan Review Tambahan yang aktif.
- Review Tambahan selalu membuat request, snapshot, payment, assignment, dan audit baru.
- Perubahan harga katalog berlaku untuk order tambahan baru, bukan request sebelumnya.

### 15.10 Refund Konsultasi

```mermaid
flowchart TD
    START([User membuka transaksi konsultasi]) --> ELIGIBLE{Transaksi dapat diajukan?}
    ELIGIBLE -- Tidak --> DENY[Tampilkan alasan tidak memenuhi syarat]
    ELIGIBLE -- Ya --> FORM[Isi alasan dan bukti bila diperlukan]
    FORM --> SUBMIT[Status Submitted]
    SUBMIT --> ADMIN[Administrator review transaksi, request, dan progress]
    ADMIN --> WORK{Pekerjaan sudah IN_PROGRESS?}
    WORK -- Tidak --> POLICY[Evaluasi refund penuh sesuai kebijakan]
    WORK -- Ya --> MANUAL[Review manual nilai dan dampak]
    POLICY --> APPROVAL
    MANUAL --> APPROVAL[Persetujuan Administrator kedua]
    APPROVAL --> DECISION{Disetujui?}
    DECISION -- Tidak --> REJECT[Status Rejected; akses tidak berubah]
    DECISION -- Ya --> PROCESS[Status Processing; kirim ke provider]
    PROCESS --> SUCCESS{Provider berhasil?}
    SUCCESS -- Tidak --> FAILED[Status Failed; tidak ada penyesuaian ganda]
    SUCCESS -- Ya --> REFUNDED[Status Refunded]
    REFUNDED --> ADJUST[Batalkan request/assignment dan cabut scoped access]
    ADJUST --> NOTIFY[Kirim hasil kepada pemohon dan User Induk]
```

Aturan refund:

- Refund mengacu pada payment konsultasi dan consultation request yang sama.
- Refund sebelum pekerjaan dimulai mengikuti kebijakan kelayakan bisnis.
- Setelah `IN_PROGRESS`, nilai dan kelayakan memerlukan review manual.
- Konsultan tidak melihat alasan finansial atau nilai refund.
- Assignment yang dibatalkan harus mencabut akses preview dan working copy.
- Refund gagal tidak menutup request atau mencabut akses secara otomatis.
- Provider call, status update, dan access adjustment memakai idempotency key terpisah.
- Satu payment tidak boleh menghasilkan refund berhasil lebih dari satu kali.

### 15.11 Sinkronisasi Status Antaraktor

| Kejadian | Status request | Tampilan pengguna | Tampilan Konsultan | Tindakan Admin |
| --- | --- | --- | --- | --- |
| Request fixed dibuat | `AWAITING_PAYMENT` | Invoice tersedia | Tidak terlihat | Memantau payment |
| Payment berhasil | `PENDING_ASSIGNMENT` | Menunggu assignment | Belum terlihat | Pilih Konsultan |
| Assignment dibuat | `ASSIGNED` | Konsultan ditugaskan | Assignment baru | Pantau keputusan |
| Assignment diterima | `IN_PROGRESS` | ETA/progress tersedia | Akses aktif | Pantau SLA |
| Menunggu data pengguna | `WAITING_FOR_CLIENT` | Tindakan diperlukan | SLA pause | Pantau aging |
| Review diterbitkan | `REVIEW_DELIVERED` | Komentar tersedia | Delivery terkunci | Pantau hasil |
| Perlu versi baru | `AWAITING_RESUBMISSION` | Perbaiki dan resubmit | Menunggu | Pantau SLA pause |
| Review ulang | `RE_REVIEW` | Sedang diperiksa | Round aktif | Pantau revisi |
| Pekerjaan selesai | `COMPLETED` | Hasil final tersedia | Read-only/ditutup | Arsip |
| Refund berjalan | `REFUND_PROCESSING` | Status refund | Tidak melihat detail | Proses refund |
| Refund selesai | `REFUNDED` | Konfirmasi refund | Akses dicabut | Audit penyesuaian |

### 15.12 Rekonsiliasi dan Audit Billing

Sistem melakukan rekonsiliasi antara payment internal, webhook provider, consultation request, quotation, refund, dan assignment. Kondisi berikut harus menghasilkan alert operasional:

- payment `PAID` tetapi request belum `PENDING_ASSIGNMENT`;
- request ditugaskan tetapi payment belum `PAID`;
- quotation kedaluwarsa tetapi invoice masih dapat dibayar;
- refund `REFUNDED` tetapi assignment masih aktif;
- amount atau currency provider berbeda dari snapshot;
- webhook duplikat menghasilkan perubahan status kedua;
- request selesai tetapi scoped access belum ditutup.

Audit log mencatat perubahan katalog, quotation, payment reference, status request, refund, access adjustment, dan identitas Administrator tanpa menyimpan data kartu atau credential provider.

### 15.13 Kriteria Penerimaan Billing Konsultasi

- Harga selalu berasal dari katalog/version atau quotation backend.
- Perubahan katalog tidak mengubah request lama.
- Karyawan tidak dapat membayar konsultasi untuk dokumen yang tidak ditugaskan.
- User Induk dapat melihat seluruh transaksi konsultasi organisasi.
- Konsultan tidak dapat melihat nominal maupun metode pembayaran.
- Assignment tidak dapat dibuat sebelum payment `PAID`.
- Webhook duplikat tidak membuat aktivasi atau assignment ganda.
- Quotation kedaluwarsa tidak dapat digunakan untuk checkout.
- Review tambahan selalu menghasilkan order baru setelah revisi habis.
- Refund berhasil mencabut akses tepat satu kali.
- Refund gagal tidak mengubah akses secara prematur.
- Seluruh perubahan finansial dan status memiliki audit trail.

## 16. Implementasi Pembayaran Xendit

### 16.1 Satu Jalur Pembayaran untuk Langganan dan Konsultasi

TP Docs menggunakan Xendit **Payment Session mode `PAYMENT_LINK`** dengan `session_type: PAY` dan `capture_method: AUTOMATIC`. Jalur yang sama dipakai untuk paket, add-on, Enterprise/Custom, Review Konsultan TP, Penyusunan Penuh, dan review tambahan; yang membedakan adalah order snapshot serta `payment_purpose` internal.

| Purpose internal | Sumber nominal | Resource yang aktif setelah `PAID` |
| --- | --- | --- |
| `SUBSCRIPTION` | Version harga paket | Subscription dan quota |
| `UPGRADE` | Snapshot perhitungan upgrade | Paket/entitlement baru |
| `DOCUMENT_ADDON` | Katalog add-on | Tambahan quota dokumen |
| `SEAT_ADDON` | Harga prorata backend | Tambahan kursi sampai akhir periode |
| `CUSTOM_PLAN` | Quotation diterima | Enterprise/Custom |
| `CONSULTATION` | Service version atau quotation | Request masuk `PENDING_ASSIGNMENT` |

Harga tidak pernah berasal dari payload frontend. Consultation request menyimpan snapshot nama layanan, versi katalog, currency, amount, SLA, dan revisi sebelum Payment Session dibuat.

### 16.2 Checkout Xendit

```mermaid
sequenceDiagram
    actor User as User Induk atau Karyawan berwenang
    participant Web as Next.js
    participant API as NestJS Billing
    participant DB as PostgreSQL
    participant X as Xendit
    participant Hook as Verified Webhook
    participant Consult as Consultation Module

    User->>Web: Konfirmasi order
    Web->>API: Checkout + Idempotency-Key
    API->>DB: Validasi actor, resource, harga, currency, dan state
    API->>DB: Simpan payment PENDING dan order snapshot
    API->>X: POST /sessions (PAYMENT_LINK)
    X-->>API: session ID, payment URL, expiry
    API-->>Web: Hosted payment URL
    X-->>Hook: Status pembayaran
    Hook->>Hook: Verifikasi dan deduplikasi
    Hook->>DB: Cocokkan reference, amount, currency, dan payment
    alt Langganan atau add-on
        Hook->>DB: Terapkan entitlement tepat satu kali
    else Layanan konsultasi
        Hook->>Consult: ConsultationPaymentPaid
        Consult->>DB: PENDING_ASSIGNMENT tepat satu kali
    end
```

Success/cancel return URL hanya mengembalikan pengguna ke TP Docs. Return URL tidak mengaktifkan paket maupun membuka request Konsultan. Sumber kebenaran adalah webhook valid atau hasil rekonsiliasi server-to-server.

### 16.3 Customer dan Data yang Dikirim

Satu organisasi dapat mempunyai satu customer Xendit bertipe `BUSINESS`. Organization bukan sub-account Xendit. Data yang dikirim dibatasi pada reference ID acak, nama perusahaan, email billing, negara, dan metadata ID internal. Isi Local File, NPWP, komentar Konsultan, hasil FAR, transaksi afiliasi, dan dokumen hasil generate tidak dikirim ke payment provider.

Konsultan tidak menerima informasi channel, nominal, provider payment ID, refund reason, atau detail payout. Konsultan hanya menerima perubahan status bisnis yang relevan, misalnya assignment telah dapat dikerjakan atau akses telah dicabut.

### 16.4 Status dan Rekonsiliasi

| Provider | Internal payment | Consultation request |
| --- | --- | --- |
| Session `ACTIVE` / `REQUIRES_ACTION` | `PENDING` | `AWAITING_PAYMENT` |
| Processing | `PROCESSING` | `AWAITING_PAYMENT` |
| Berhasil | `PAID` | `PENDING_ASSIGNMENT` |
| Gagal | `FAILED` | `AWAITING_PAYMENT` atau checkout ulang |
| Kedaluwarsa | `EXPIRED` | `AWAITING_PAYMENT` |
| Refund `PENDING` | Refund `PROCESSING` | `REFUND_PROCESSING` |
| Refund `SUCCEEDED` | Refund `REFUNDED` | `REFUNDED` dan akses dicabut |
| Refund `FAILED` | Refund `FAILED` | Status kerja tidak diubah otomatis |

Webhook diproses dengan verification, payload hash, dedupe key, database lock, dan idempotency. Rekonsiliasi menghasilkan alert bila payment `PAID` belum mengubah request menjadi `PENDING_ASSIGNMENT`, assignment ada sebelum `PAID`, nominal/currency berbeda, atau refund berhasil tetapi scoped access masih aktif.

### 16.5 Refund, Dispute, dan Payout

- refund selalu merujuk ke provider `payment_id` asal dan dikembalikan melalui mekanisme refund provider;
- penyesuaian subscription atau akses Konsultan dilakukan setelah refund `SUCCEEDED`, bukan saat diajukan;
- dispute kartu disimpan terpisah dari refund karena dapat meminta evidence dan memiliki due date;
- event dispute berulang dideduplicasi menggunakan dispute ID, status, status reason, dan updated time;
- payout bukan pembayaran Konsultan pada MVP dan bukan refund pelanggan;
- pembayaran jasa Konsultan merupakan pendapatan TP Docs; kompensasi Konsultan berada di luar alur order pelanggan sampai kebijakan bisnis dan legalnya ditetapkan;
- pencairan saldo perusahaan melalui Admin hanya dapat diaktifkan setelah rekening allowlist, limit, dual approval, dan dukungan API provider tervalidasi.

### 16.6 Perpanjangan dan Multi-Currency

MVP memakai checkout baru saat perpanjangan. Penyimpanan metode pembayaran dan auto-renew tidak diaktifkan sebelum consent serta mekanisme recurring selesai dirancang. Currency disimpan sebagai ISO-4217 dan amount sebagai minor unit. Dukungan pelanggan luar negeri mengikuti country, channel, settlement currency, dan refund capability yang aktif pada akun Xendit; TP Docs tidak melakukan konversi otomatis.

### 16.7 Referensi Xendit

- [Subscription with Initial Payment — Payment Session](https://docs.xendit.co/docs/subscription-with-initial-payment)
- [Payment Request Details](https://docs.xendit.co/apidocs/get-payment-request)
- [Refund V3](https://docs.xendit.co/apidocs/get-a-refund)
- [Payout V3](https://docs.xendit.co/apidocs/get-payout-v3-by-id)
- [Manage Disputes via API](https://docs.xendit.co/docs/manage-disputes-via-api)
- [Customers API](https://docs.xendit.co/apidocs/get-customer-id)

## 17. Referensi Layanan Konsultasi

- [Consultation Service Business Flow](./08-Consultation-Service.md)
- [Consultant Workflow](./09-Consultant-Workflow.md)
- [Use Case Konsultan](../02-Use-Case/Consultant.md)
- [Consultation Architecture](../05-Architecture/Consultation-Architecture.md)
- [API Contract](../05-Architecture/API-Contract.md)
