# Subscription and Billing Business Flow — TP Docs (By System)

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

## 14. Implementasi Pembayaran Xendit

### 14.1 Produk Xendit yang Digunakan

TP Docs menggunakan **Payment Session dengan mode `PAYMENT_LINK`** untuk paket reguler, upgrade, add-on, serta pembayaran quotation Enterprise/Custom. Session memakai `session_type: PAY` dan `capture_method: AUTOMATIC`. Browser hanya diarahkan ke `payment_link_url`; secret API key tidak pernah dikirim ke frontend.

Pada MVP, perpanjangan dilakukan melalui checkout baru. Penyimpanan metode pembayaran (`allow_save_payment_method`) dan auto-renew belum diaktifkan sampai consent, kanal, penanganan kegagalan recurring, dan kebijakan penghentian telah divalidasi.

| Kebutuhan TP Docs | Implementasi Xendit |
| --- | --- |
| Paket Basic/Professional | Payment Session `PAYMENT_LINK` |
| Upgrade paket | Session baru dengan snapshot selisih harga yang telah dihitung backend |
| Add-on dokumen/kursi | Session baru dengan purpose dan order terpisah |
| Enterprise/Custom | Session dibuat setelah quotation diterima |
| Identitas billing | Customer bertipe `BUSINESS` opsional per organisasi |
| Refund | Refund V3 terhadap `payment_id` asal |
| Pencairan dana | Bukan refund; lihat batas payout pada bagian 14.6 |

### 14.2 Checkout dan Aktivasi

```mermaid
sequenceDiagram
    actor Owner as User Induk
    participant Web as Next.js
    participant API as NestJS Billing
    participant DB as PostgreSQL
    participant X as Xendit
    participant Hook as Verified Webhook

    Owner->>Web: Konfirmasi paket atau add-on
    Web->>API: Buat checkout + Idempotency-Key
    API->>DB: Validasi owner, harga, quota, dan quotation
    API->>DB: Buat payment PENDING + snapshot order
    API->>X: POST /sessions (PAY, PAYMENT_LINK, AUTOMATIC)
    X-->>API: payment_session_id, payment_link_url, expires_at
    API->>DB: Simpan payment session ACTIVE
    API-->>Web: paymentUrl dan expiry
    Web-->>Owner: Redirect ke hosted checkout Xendit
    X-->>Hook: Event/status pembayaran
    Hook->>Hook: Verifikasi autentisitas dan deduplikasi event
    Hook->>DB: Lock payment dan cocokkan amount/currency/reference
    Hook->>DB: Payment PAID + aktivasi entitlement satu transaksi
    API-->>Web: Status terbaru saat halaman hasil dipolling
```

Return URL hanya untuk pengalaman pengguna dan **bukan** bukti pembayaran. Paket, quota, kursi, atau add-on hanya aktif setelah webhook valid atau rekonsiliasi server-to-server mengonfirmasi pembayaran berhasil.

`reference_id` dibuat unik oleh TP Docs dan tidak menggunakan email atau nama perusahaan. Metadata provider hanya memuat ID internal aman seperti `paymentId`, `organizationId`, dan `purpose`; data Local File tidak dikirim.

### 14.3 Pemetaan Status

| Status provider | Status internal | Dampak |
| --- | --- | --- |
| Session `ACTIVE` | `PENDING` | Menunggu tindakan pengguna |
| Payment Request `REQUIRES_ACTION` | `PENDING` | Tampilkan instruksi/redirect dari provider |
| Sedang diproses | `PROCESSING` | Jangan membuat session pengganti otomatis |
| Pembayaran berhasil | `PAID` | Terapkan entitlement tepat satu kali |
| Gagal | `FAILED` | Tidak mengubah paket atau quota |
| Kedaluwarsa | `EXPIRED` | Checkout ulang membuat payment/session baru |
| Dibatalkan | `CANCELLED` | Tidak mengubah entitlement |
| Refund `PENDING` | Refund `PROCESSING` | Akses belum disesuaikan |
| Refund `SUCCEEDED` | Refund `REFUNDED` | Sesuaikan akses tepat satu kali |
| Refund `FAILED` | Refund `FAILED` | Pertahankan akses dan tandai tindak lanjut |

Status provider disimpan sebagai metadata integrasi. Domain TP Docs tetap menggunakan enum internal agar perubahan API provider tidak langsung mengubah aturan bisnis.

### 14.4 Webhook, Idempotensi, dan Rekonsiliasi

- webhook masuk melalui endpoint NestJS khusus dan memverifikasi token/signature sesuai konfigurasi Xendit;
- raw payload hanya digunakan untuk verifikasi, lalu log menyimpan hash dan metadata aman;
- deduplication key memakai event ID provider; untuk event yang dapat berulang, key mencakup resource ID, status, alasan, dan waktu pembaruan;
- amount, currency, `reference_id`, serta provider payment ID harus cocok dengan snapshot internal;
- database lock mencegah dua worker mengaktifkan subscription yang sama;
- event duplikat mengembalikan respons sukses tanpa mengulang quota ledger atau aktivasi;
- job rekonsiliasi memeriksa payment `PENDING/PROCESSING` yang melewati batas waktu melalui API Xendit;
- redirect sukses dari browser tidak pernah langsung menandai payment `PAID`.

### 14.5 Refund dan Dispute

Refund dibuat terhadap pembayaran asal menggunakan reference ID unik. TP Docs menyimpan `refund_id` provider dan memetakan status `PENDING`, `SUCCEEDED`, atau `FAILED`. Pengurangan quota, kursi, atau masa aktif baru dilakukan setelah status provider `SUCCEEDED` dan memakai idempotency key terpisah.

Pembayaran kartu dapat menghasilkan dispute. Event `dispute.action_required` dapat dikirim lebih dari sekali dan mempunyai `due_date`; karena itu sistem menyimpan kasus dispute, status reason, tenggat, serta evidence yang diizinkan. Dispute tidak otomatis diperlakukan sebagai refund. Administrator menerima alert dan tindakan finansial mengikuti keputusan provider.

### 14.6 Settlement, Withdrawal, dan Payout

Refund mengembalikan dana ke metode pembayaran pelanggan. Payout adalah transfer keluar ke rekening penerima dan bukan mekanisme refund.

Untuk MVP:

- settlement dana merchant dipantau sebagai laporan/rekonsiliasi Xendit;
- TP Docs tidak membuat payout kepada User Induk atau Karyawan;
- tombol pencairan di portal Admin hanya boleh diaktifkan jika akun Xendit, rekening perusahaan, izin API, limit, dan alur dual approval telah tervalidasi;
- bila payout diaktifkan, hanya rekening perusahaan yang telah di-allowlist yang dapat dipilih;
- status payout seperti `SUCCEEDED` atau `FAILED` disinkronkan melalui provider ID dan failure code;
- operasi payout wajib re-authentication, persetujuan Administrator kedua, idempotency, serta audit log.

### 14.7 Mata Uang dan Pelanggan Luar Negeri

Database menyimpan nominal dalam minor unit dan currency ISO-4217. Kanal, negara, currency, biaya lintas negara, settlement currency, serta ketersediaan refund harus diperiksa dari konfigurasi akun Xendit sebelum ditampilkan. MVP tidak melakukan konversi mata uang otomatis. Apabila harga tersedia dalam currency lain, TP Docs menggunakan `plan_prices` atau quotation dalam currency tersebut; rate konversi tidak dihitung sendiri saat checkout.

### 14.8 Referensi Xendit

- [Subscription with Initial Payment — Payment Session](https://docs.xendit.co/docs/subscription-with-initial-payment)
- [Payment Request Details](https://docs.xendit.co/apidocs/get-payment-request)
- [Get Payment Details](https://docs.xendit.co/docs/retrieving-authorization-and-decline-details)
- [Refund V3](https://docs.xendit.co/apidocs/get-a-refund)
- [Payout V3](https://docs.xendit.co/apidocs/get-payout-v3-by-id)
- [Manage Disputes via API](https://docs.xendit.co/docs/manage-disputes-via-api)

## 15. Referensi

- [Business Flow Overview](./00-Business-Flow-Overview.md)
- [Membership and Organization](./05-Membership-Organization.md)
- [Use Case User Induk](../02-Use-Case/Organization-Owner.md)
- [Use Case Karyawan](../02-Use-Case/Employee.md)
- [Use Case Administrator](../02-Use-Case/Admin.md)
- [Functional Requirements](../01-Requirements/Functional-Requirements.md)
- [Non-Functional Requirements](../01-Requirements/Non-Functional-Requirements.md)
