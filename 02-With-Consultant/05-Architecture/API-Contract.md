# API Contract — TP Docs With Consultant

## 1. Tujuan dan Cakupan

Dokumen ini mendefinisikan REST API TP Docs untuk:

- publik dan autentikasi;
- User Induk;
- Karyawan;
- Konsultan;
- Administrator;
- organisasi dan membership;
- subscription, billing, payment, refund, dan withdrawal;
- Local File, transaksi, FAR, benchmarking, AI, dan document generation;
- katalog serta workflow konsultasi;
- webhook dan internal service.

Prefix MVP:

```text
/api
```

URL versioning belum digunakan. Breaking change memerlukan strategi versioning tersendiri.

## 2. Prinsip Kontrak

1. Browser hanya memanggil NestJS.
2. Frontend tidak memanggil database, payment provider, Resend, Gemini, FastAPI, atau vLLM secara langsung.
3. Backend tidak mempercayai `organizationId`, role, harga, status, atau ownership dari frontend tanpa validasi.
4. Semua mutation sensitif menggunakan audit log.
5. Payment, generation, promotion, dan webhook menggunakan idempotency.
6. File privat diunggah dan di-stream melalui NestJS setelah validasi tenant, assignment, dan hak akses; browser tidak mengakses PostgreSQL langsung.
7. DTO Konsultan tidak memuat nominal billing.
8. Response tidak memuat password hash, secret, token provider, atau data finansial mentah.
9. Resource tenant lain dikembalikan sebagai `404` atau `403` sesuai kebijakan anti-enumeration.
10. Tanggal menggunakan ISO-8601 UTC.

## 3. Authentication dan Header

### 3.1 Header Umum

| Header | Wajib | Fungsi |
| --- | --- | --- |
| `Authorization: Bearer <token>` | Ya untuk endpoint terlindungi | Access token/session |
| `Content-Type: application/json` | Mutation JSON | Format payload |
| `Idempotency-Key` | Payment, generation, request, promotion | Mencegah operasi ganda |
| `X-Correlation-Id` | Opsional dari aplikasi pemanggil | Trace request; server membuat jika kosong |
| `If-Match` | Update optimistic | Version/ETag resource |
| `Accept-Language` | Opsional | Locale pesan aman |

Aplikasi pemanggil tidak boleh menggunakan header role atau tenant sebagai sumber otorisasi.

### 3.2 Authentication Assurance

- User Induk: MFA wajib.
- Karyawan: MFA opsional pada versi awal.
- Konsultan: MFA wajib.
- Administrator: MFA wajib dan re-authentication untuk tindakan berisiko tinggi.

## 4. Response Convention

### 4.1 Success

```json
{
  "data": {},
  "meta": {
    "correlationId": "cor_...",
    "timestamp": "2026-11-20T10:00:00Z"
  }
}
```

### 4.2 Collection

```json
{
  "data": [],
  "pagination": {
    "page": 1,
    "pageSize": 25,
    "total": 120,
    "totalPages": 5
  },
  "meta": {
    "correlationId": "cor_..."
  }
}
```

### 4.3 Error

```json
{
  "error": {
    "code": "ASSIGNMENT_ACCESS_DENIED",
    "message": "Anda tidak memiliki akses ke resource ini.",
    "details": [],
    "correlationId": "cor_..."
  }
}
```

Error tidak mengungkap secret, stack trace, query, provider response mentah, atau keberadaan tenant lain.

## 5. Query Convention

| Parameter | Default | Keterangan |
| --- | --- | --- |
| `page` | 1 | Nomor halaman |
| `pageSize` | 25 | Maksimum sesuai kebijakan endpoint |
| `sort` | Endpoint-specific | Field allowlist |
| `order` | `desc` | `asc` atau `desc` |
| `search` | kosong | Pencarian yang telah disanitasi |
| `status` | kosong | Filter enum |
| `from`, `to` | kosong | Rentang tanggal ISO |
| `include` | kosong | Relasi allowlist, bukan arbitrary ORM include |

## 6. Matriks Akses Ringkas

| Domain | User Induk | Karyawan | Konsultan | Administrator |
| --- | --- | --- | --- | --- |
| Profil pribadi | Sendiri | Sendiri | Sendiri | Sendiri |
| Organisasi | Kelola | Baca terbatas | Tidak | Metadata operasional |
| Anggota | Kelola | Tidak | Tidak | Tidak mengubah membership pelanggan |
| Local File | Buatan sendiri, dibagikan Karyawan, atau diambil alih | Buatan sendiri atau ditugaskan aktif | Assignment konsultasi | Tidak otomatis |
| Subscription | Kelola; lihat kuota agregat | Lihat kuota dokumen agregat, tanpa billing | Tidak | Operasional |
| Konsultasi | Dokumen yang dapat diakses | Request miliknya pada dokumen buatan sendiri/ditugaskan | Assignment miliknya | Kelola platform |
| Harga/payment | Ya | Payment miliknya | Tidak | Sesuai kewenangan |
| Katalog konsultasi | Baca | Baca | Tidak | Kelola |
| Working copy approval | Request sendiri; ambil alih bila hak pemohon dicabut | Request sendiri dengan penugasan dan hak ubah aktif | Menyerahkan hasil, tidak menyetujui | Operasional |

## 7. Public dan System Endpoints

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/health/live` | Tidak | Liveness aplikasi |
| GET | `/api/health/ready` | Internal/monitor | Readiness dependency utama |
| GET | `/api/public/plans` | Tidak | Daftar paket aktif |
| GET | `/api/public/consultation-services` | Tidak/opsional | Ringkasan jasa aktif tanpa data internal |
| GET | `/api/public/maintenance` | Tidak | Status maintenance publik |
| GET | `/api/public/config` | Tidak | Konfigurasi publik aman |

## 8. Authentication dan Identity

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| POST | `/api/auth/register` | Tidak | Registrasi User Induk dan perusahaan |
| POST | `/api/auth/verify-email` | Token | Verifikasi email |
| POST | `/api/auth/resend-verification` | Tidak | Kirim ulang verifikasi |
| POST | `/api/auth/login` | Tidak | Login tahap password |
| POST | `/api/auth/mfa/challenge` | Partial | Verifikasi MFA login |
| POST | `/api/auth/mfa/setup` | Ya | Membuat MFA method |
| POST | `/api/auth/mfa/verify-setup` | Ya | Mengaktifkan MFA |
| POST | `/api/auth/mfa/recovery` | Partial | Login dengan recovery code |
| GET | `/api/auth/mfa/methods` | Ya | Daftar MFA method milik sendiri |
| DELETE | `/api/auth/mfa/methods/{methodId}` | Re-auth | Cabut MFA method |
| POST | `/api/auth/forgot-password` | Tidak | Meminta reset password |
| POST | `/api/auth/reset-password` | Token | Membuat password baru |
| POST | `/api/auth/change-password` | Re-auth | Mengubah password |
| POST | `/api/auth/refresh` | Refresh token | Rotasi session |
| POST | `/api/auth/logout` | Ya | Cabut sesi saat ini |
| POST | `/api/auth/logout-all` | Re-auth | Cabut seluruh sesi sendiri |
| GET | `/api/auth/sessions` | Ya | Daftar sesi sendiri |
| DELETE | `/api/auth/sessions/{sessionId}` | Ya | Cabut sesi tertentu |

### 8.1 Login Response

```ts
type LoginResponse =
  | { status: 'MFA_REQUIRED'; challengeId: string; expiresAt: string }
  | { status: 'AUTHENTICATED'; accessToken: string; expiresAt: string; user: SafeUser };
```

## 9. Profil Pengguna

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/me` | Ya | Profil dan role efektif |
| PATCH | `/api/me` | Ya | Update nama, telepon, jabatan, locale |
| POST | `/api/me/email-change` | Re-auth | Memulai perubahan email |
| POST | `/api/me/email-change/verify` | Token | Verifikasi email baru |
| POST | `/api/me/avatar` | Ya | Upload avatar melalui NestJS dan validasi file |
| DELETE | `/api/me/avatar` | Ya | Hapus avatar |
| GET | `/api/me/notifications` | Ya | Notifikasi sendiri |
| PATCH | `/api/me/notifications/{notificationId}/read` | Ya | Tandai dibaca |
| POST | `/api/me/notifications/read-all` | Ya | Tandai semua dibaca |
| GET | `/api/me/audit-events` | Ya | Aktivitas akun sendiri yang diizinkan |
| POST | `/api/me/export` | Re-auth | Meminta ekspor data pribadi |
| POST | `/api/me/delete-request` | Re-auth | Meminta penghapusan akun sesuai aturan |

## 10. Organization dan Membership

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/organizations` | Ya | Organisasi yang dapat diakses |
| GET | `/api/organizations/{organizationId}` | Member | Detail organisasi |
| PATCH | `/api/organizations/{organizationId}` | Owner | Update profil dasar |
| GET | `/api/organizations/{organizationId}/profile` | Member | Profil lengkap |
| PUT | `/api/organizations/{organizationId}/profile` | Owner | Simpan profil |
| GET | `/api/organizations/{organizationId}/members` | Owner | Daftar anggota |
| GET | `/api/organizations/{organizationId}/members/{membershipId}` | Owner | Detail membership |
| DELETE | `/api/organizations/{organizationId}/members/{membershipId}` | Owner | Hapus Karyawan |
| GET | `/api/organizations/{organizationId}/invitations` | Owner | Undangan tertunda |
| POST | `/api/organizations/{organizationId}/invitations` | Owner | Kirim undangan Karyawan |
| POST | `/api/organizations/{organizationId}/invitations/{invitationId}/resend` | Owner | Kirim ulang |
| DELETE | `/api/organizations/{organizationId}/invitations/{invitationId}` | Owner | Batalkan undangan |
| GET | `/api/invitations/{token}` | Tidak | Validasi metadata undangan aman |
| POST | `/api/invitations/{token}/accept` | Ya | Terima undangan |
| POST | `/api/invitations/{token}/reject` | Ya | Tolak undangan |
| GET | `/api/organizations/{organizationId}/seats` | Owner | Kursi terpakai/tersedia |
| GET | `/api/organizations/{organizationId}/document-assignments` | Owner | Semua penugasan |
| POST | `/api/organizations/{organizationId}/document-assignments` | Owner | Tugaskan dokumen |
| DELETE | `/api/organizations/{organizationId}/document-assignments/{assignmentId}` | Owner | Cabut penugasan |

## 11. Plans, Subscription, Quota, dan Billing

### Diskon Event Opsional

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | /api/discounts/active | Pengguna | Estimasi harga normal atau promo untuk produk |
| GET | /api/admin/discount-campaigns | Admin | Daftar dan penggunaan event |
| POST | /api/admin/discount-campaigns | Admin | Buat draft |
| PATCH | /api/admin/discount-campaigns/{campaignId} | Admin | Ubah draft dan target |
| POST | /api/admin/discount-campaigns/{campaignId}/activate | Admin + re-auth | Aktifkan |
| POST | /api/admin/discount-campaigns/{campaignId}/deactivate | Admin + re-auth | Nonaktifkan |

Checkout dan quotation memakai endpoint yang ada. Backend mengembalikan originalAmountMinor, discountAmountMinor bila ada, totalAmountMinor, currency, dan discountCampaignId opsional. Nominal browser diabaikan. Xendit menerima total snapshot; retry memakai snapshot sama; refund memakai nilai dibayar.


### 11.1 Plans dan Subscription

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/plans` | Ya | Paket aktif |
| GET | `/api/plans/{planCode}` | Ya | Detail paket |
| GET | `/api/organizations/{organizationId}/subscription` | Owner | Paket dan masa aktif |
| POST | `/api/organizations/{organizationId}/subscription/checkout` | Owner | Checkout paket |
| POST | `/api/organizations/{organizationId}/subscription/upgrade` | Owner | Upgrade paket |
| POST | `/api/organizations/{organizationId}/subscription/cancel` | Owner | Batalkan renewal sesuai kebijakan |
| GET | `/api/organizations/{organizationId}/quotas` | Owner/Member aktif | Kuota dokumen agregat untuk keduanya; rincian kursi/billing hanya Owner |
| GET | `/api/organizations/{organizationId}/quota-ledger` | Owner | Histori kuota; dokumen privat hanya berupa agregat tanpa ID atau metadata |
| POST | `/api/organizations/{organizationId}/addons/documents/checkout` | Owner | Beli dokumen tambahan |
| POST | `/api/organizations/{organizationId}/addons/seats/checkout` | Owner | Beli kursi tambahan |
| GET | `/api/organizations/{organizationId}/addons` | Owner | Daftar add-on |

### 11.2 Payment dan Refund

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/organizations/{organizationId}/payments` | Owner | Riwayat pembayaran organisasi |
| GET | `/api/organizations/{organizationId}/payments/{paymentId}` | Owner/payer | Detail payment aman |
| GET | `/api/organizations/{organizationId}/payments/{paymentId}/sessions` | Owner/payer | Histori hosted checkout session dan expiry |
| POST | `/api/organizations/{organizationId}/payments/{paymentId}/retry` | Owner/payer | Buat attempt baru bila diizinkan |
| GET | `/api/organizations/{organizationId}/refunds` | Owner/payer | Daftar refund |
| POST | `/api/organizations/{organizationId}/refunds` | Owner/payer | Ajukan refund |
| GET | `/api/organizations/{organizationId}/refunds/{refundId}` | Owner/payer | Status refund |
| POST | `/api/organizations/{organizationId}/refunds/{refundId}/evidence` | Owner/payer | Upload bukti melalui NestJS setelah validasi refund dan file |
| DELETE | `/api/organizations/{organizationId}/refunds/{refundId}` | Requester | Batalkan sebelum review bila diizinkan |

### 11.2.1 Hosted Checkout Response

```ts
type HostedCheckoutResponse = {
  paymentId: string;
  sessionId: string;
  status: 'PENDING';
  paymentUrl: string;
  expiresAt: string;
};
```

`paymentUrl` hanya URL checkout hosted. Return ke halaman sukses tidak mengubah status menjadi `PAID`; frontend melakukan polling detail payment sampai verified webhook atau rekonsiliasi memperbarui status.

### 11.3 Enterprise/Custom

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| POST | `/api/organizations/{organizationId}/custom-plan-requests` | Owner | Ajukan Enterprise/Custom |
| GET | `/api/organizations/{organizationId}/custom-plan-requests` | Owner | Daftar pengajuan |
| GET | `/api/organizations/{organizationId}/custom-plan-requests/{requestId}` | Owner | Detail/status |
| PATCH | `/api/organizations/{organizationId}/custom-plan-requests/{requestId}` | Owner | Perbaiki saat revision requested |
| POST | `/api/organizations/{organizationId}/custom-plan-requests/{requestId}/resubmit` | Owner | Kirim ulang |
| POST | `/api/organizations/{organizationId}/custom-plan-requests/{requestId}/accept-quotation` | Owner | Terima quotation |
| POST | `/api/organizations/{organizationId}/custom-plan-requests/{requestId}/reject-quotation` | Owner | Tolak quotation |
| POST | `/api/organizations/{organizationId}/custom-plan-requests/{requestId}/payments` | Owner | Bayar quotation |

## 12. File Management

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| POST | `/api/files` | Ya | Upload `multipart/form-data`, maksimum 50 MB |
| GET | `/api/files/{fileId}` | Resource permission | Metadata dan scan status |
| GET | `/api/files/{fileId}/content` | Resource permission | Stream konten; mendukung HTTP Range dan `206` untuk PDF |
| DELETE | `/api/files/{fileId}` | Resource owner | Hapus logical sesuai aturan |
| GET | `/api/files/{fileId}/processing-status` | Resource permission | Scan/extraction status |

Upload memakai field file serta metadata `purpose`, organization context, dan target resource. Backend mengautentikasi pengguna, memvalidasi role, tenant, assignment Local File jika berlaku, MIME, ekstensi, ukuran maksimal 50 MB, checksum, dan malware scan, lalu menyimpan konten sebagai chunk `bytea` 1 MiB di PostgreSQL tanpa memuat seluruh file ke memori. File baru berstatus lengkap setelah seluruh chunk dan checksum tervalidasi. Preview dan download melalui NestJS mengulang pemeriksaan akses pada setiap request; akses file dicatat pada audit log. Respons konten mengirim `Content-Type`, `Content-Length`, `Content-Disposition`, dan checksum sebagai `ETag`.

## 13. Local File Core

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/organizations/{organizationId}/local-files` | Member | Daftar sesuai permission |
| POST | `/api/organizations/{organizationId}/local-files` | Owner/Member aktif | Buat Local File dengan kuota organisasi; buatan Karyawan dimulai privat |
| GET | `/api/local-files/{localFileId}` | Resource access | Detail dan progress sesuai pembuat, penugasan, berbagi, atau pengambilalihan |
| PATCH | `/api/local-files/{localFileId}` | Resource write | Update metadata |
| DELETE | `/api/local-files/{localFileId}` | Resource owner | Hapus sesuai retensi |
| POST | `/api/local-files/{localFileId}/duplicate` | Resource access + kuota | Duplikasi dengan privasi awal sesuai pembuat baru |
| GET | `/api/local-files/{localFileId}/owner-sharing` | Creator/Shared owner | Status berbagi dengan User Induk |
| PUT | `/api/local-files/{localFileId}/owner-sharing` | Employee creator aktif | Beri atau cabut akses lihat/ubah User Induk; langsung berlaku dan diaudit |
| GET | `/api/local-files/{localFileId}/sections` | Assigned | Progress section |
| PATCH | `/api/local-files/{localFileId}/sections/{sectionCode}` | Assigned | Update status section |
| POST | `/api/local-files/{localFileId}/validate` | Assigned | Validasi kelengkapan |
| GET | `/api/local-files/{localFileId}/history` | Assigned | Histori perubahan |
| GET | `/api/local-files/{localFileId}/audit-events` | Resource owner | Audit dokumen sesuai akses pembuat atau pengambilalihan |

## 14. Profil Perusahaan, Kepemilikan, dan Bisnis

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/local-files/{id}/company-profile` | Assigned | Profil perusahaan |
| PUT | `/api/local-files/{id}/company-profile` | Assigned | Profil perusahaan |
| GET | `/api/local-files/{id}/shareholders` | Assigned | Daftar pemegang saham/tambah |
| POST | `/api/local-files/{id}/shareholders` | Assigned | Daftar pemegang saham/tambah |
| PATCH | `/api/local-files/{id}/shareholders/{shareholderId}` | Assigned | Ubah/hapus pemegang saham |
| DELETE | `/api/local-files/{id}/shareholders/{shareholderId}` | Assigned | Ubah/hapus pemegang saham |
| GET | `/api/local-files/{id}/management` | Assigned | Komisaris/direksi |
| POST | `/api/local-files/{id}/management` | Assigned | Komisaris/direksi |
| PATCH | `/api/local-files/{id}/management/{managementId}` | Assigned | Ubah/hapus pengurus |
| DELETE | `/api/local-files/{id}/management/{managementId}` | Assigned | Ubah/hapus pengurus |
| GET | `/api/local-files/{id}/business-profile` | Assigned | Uraian bisnis dan strategi |
| PUT | `/api/local-files/{id}/business-profile` | Assigned | Uraian bisnis dan strategi |
| GET | `/api/local-files/{id}/business-process-steps` | Assigned | Langkah proses |
| POST | `/api/local-files/{id}/business-process-steps` | Assigned | Langkah proses |
| PATCH | `/api/local-files/{id}/business-process-steps/{stepId}` | Assigned | Ubah/hapus langkah |
| DELETE | `/api/local-files/{id}/business-process-steps/{stepId}` | Assigned | Ubah/hapus langkah |
| POST | `/api/local-files/{id}/business-process-steps/reorder` | Assigned | Ubah urutan |
| GET | `/api/local-files/{id}/financial-summary` | Assigned | Ringkasan keuangan |
| PUT | `/api/local-files/{id}/financial-summary` | Assigned | Ringkasan keuangan |
| GET | `/api/local-files/{id}/affiliates` | Assigned | Pihak afiliasi |
| POST | `/api/local-files/{id}/affiliates` | Assigned | Pihak afiliasi |
| PATCH | `/api/local-files/{id}/affiliates/{affiliateId}` | Assigned | Ubah/hapus afiliasi |
| DELETE | `/api/local-files/{id}/affiliates/{affiliateId}` | Assigned | Ubah/hapus afiliasi |
| GET | `/api/local-files/{id}/competitors` | Assigned | Pesaing utama |
| POST | `/api/local-files/{id}/competitors` | Assigned | Pesaing utama |
| PATCH | `/api/local-files/{id}/competitors/{competitorId}` | Assigned | Ubah/hapus pesaing |
| DELETE | `/api/local-files/{id}/competitors/{competitorId}` | Assigned | Ubah/hapus pesaing |
| GET | `/api/local-files/{id}/industry-overview` | Assigned | Gambaran industri |
| PUT | `/api/local-files/{id}/industry-overview` | Assigned | Gambaran industri |
| POST | `/api/local-files/{id}/industry-overview/files` | Assigned | Kaitkan file Word/PDF |
| DELETE | `/api/local-files/{id}/industry-overview/files/{fileId}` | Assigned | Lepas file |
| GET | `/api/local-files/{id}/non-financial-events` | Assigned | Kejadian nonkeuangan |
| POST | `/api/local-files/{id}/non-financial-events` | Assigned | Kejadian nonkeuangan |
| PATCH | `/api/local-files/{id}/non-financial-events/{eventId}` | Assigned | Ubah/hapus kejadian |
| DELETE | `/api/local-files/{id}/non-financial-events/{eventId}` | Assigned | Ubah/hapus kejadian |

## 15. Transaksi dan Import

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/local-files/{id}/transaction-template` | Assigned | Download template |
| GET | `/api/local-files/{id}/transaction-tsv-template` | Assigned | Format TSV |
| POST | `/api/local-files/{id}/transaction-imports` | Assigned | Mulai import Excel/TSV |
| GET | `/api/local-files/{id}/transaction-imports/{importId}` | Assigned | Status import |
| GET | `/api/local-files/{id}/transaction-imports/{importId}/errors` | Assigned | Error baris |
| POST | `/api/local-files/{id}/transaction-imports/{importId}/commit` | Assigned | Commit hasil valid |
| DELETE | `/api/local-files/{id}/transaction-imports/{importId}` | Assigned | Batalkan import |
| GET | `/api/local-files/{id}/affiliate-transactions` | Assigned | Daftar/tambah transaksi afiliasi |
| POST | `/api/local-files/{id}/affiliate-transactions` | Assigned | Daftar/tambah transaksi afiliasi |
| PATCH | `/api/local-files/{id}/affiliate-transactions/{transactionId}` | Assigned | Ubah/hapus |
| DELETE | `/api/local-files/{id}/affiliate-transactions/{transactionId}` | Assigned | Ubah/hapus |
| GET | `/api/local-files/{id}/independent-transactions` | Assigned | Daftar/tambah transaksi independen |
| POST | `/api/local-files/{id}/independent-transactions` | Assigned | Daftar/tambah transaksi independen |
| PATCH | `/api/local-files/{id}/independent-transactions/{transactionId}` | Assigned | Ubah/hapus |
| DELETE | `/api/local-files/{id}/independent-transactions/{transactionId}` | Assigned | Ubah/hapus |
| GET | `/api/local-files/{id}/transaction-summary` | Assigned | Summary auto-generated |
| POST | `/api/local-files/{id}/transaction-summary/recalculate` | Assigned | Hitung ulang |

## 16. FAR dan Metode TP

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/local-files/{id}/far-analysis` | Assigned | Root FAR |
| PUT | `/api/local-files/{id}/far-analysis` | Assigned | Root FAR |
| GET | `/api/local-files/{id}/far/functions` | Assigned | Fungsi |
| POST | `/api/local-files/{id}/far/functions` | Assigned | Fungsi |
| PATCH | `/api/local-files/{id}/far/functions/{functionId}` | Assigned | Ubah/hapus |
| DELETE | `/api/local-files/{id}/far/functions/{functionId}` | Assigned | Ubah/hapus |
| GET | `/api/local-files/{id}/far/assets` | Assigned | Aset |
| POST | `/api/local-files/{id}/far/assets` | Assigned | Aset |
| PATCH | `/api/local-files/{id}/far/assets/{assetId}` | Assigned | Ubah/hapus |
| DELETE | `/api/local-files/{id}/far/assets/{assetId}` | Assigned | Ubah/hapus |
| GET | `/api/local-files/{id}/far/risks` | Assigned | Risiko |
| POST | `/api/local-files/{id}/far/risks` | Assigned | Risiko |
| PATCH | `/api/local-files/{id}/far/risks/{riskId}` | Assigned | Ubah/hapus |
| DELETE | `/api/local-files/{id}/far/risks/{riskId}` | Assigned | Ubah/hapus |
| GET | `/api/local-files/{id}/tp-method-selection` | Assigned | Metode terpilih, tested party, PLI, alasan |
| PUT | `/api/local-files/{id}/tp-method-selection` | Assigned | Metode terpilih, tested party, PLI, alasan |
| GET | `/api/local-files/{id}/tp-method-evaluations` | Assigned | Evaluasi metode |
| POST | `/api/local-files/{id}/tp-method-evaluations` | Assigned | Evaluasi metode |
| PATCH | `/api/local-files/{id}/tp-method-evaluations/{evaluationId}` | Assigned | Ubah/hapus evaluasi |
| DELETE | `/api/local-files/{id}/tp-method-evaluations/{evaluationId}` | Assigned | Ubah/hapus evaluasi |

## 17. Benchmarking dan Pustaka Pembanding

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/local-files/{id}/benchmarking` | Assigned | Root benchmarking |
| PUT | `/api/local-files/{id}/benchmarking` | Assigned | Root benchmarking |
| POST | `/api/local-files/{id}/benchmarking/files` | Assigned | Upload/tautkan PDF |
| GET | `/api/local-files/{id}/benchmark-search-steps` | Assigned | Langkah pencarian |
| POST | `/api/local-files/{id}/benchmark-search-steps` | Assigned | Langkah pencarian |
| PATCH | `/api/local-files/{id}/benchmark-search-steps/{stepId}` | Assigned | Ubah/hapus |
| DELETE | `/api/local-files/{id}/benchmark-search-steps/{stepId}` | Assigned | Ubah/hapus |
| GET | `/api/local-files/{id}/benchmark-search-results` | Assigned | Hasil perusahaan |
| POST | `/api/local-files/{id}/benchmark-search-results` | Assigned | Hasil perusahaan |
| PATCH | `/api/local-files/{id}/benchmark-search-results/{resultId}` | Assigned | Ubah/hapus |
| DELETE | `/api/local-files/{id}/benchmark-search-results/{resultId}` | Assigned | Ubah/hapus |
| GET | `/api/local-files/{id}/pli-selection` | Assigned | Pilih satu PLI |
| PUT | `/api/local-files/{id}/pli-selection` | Assigned | Pilih satu PLI |
| GET | `/api/local-files/{id}/comparables` | Assigned | Pembanding final |
| POST | `/api/local-files/{id}/comparables` | Assigned | Pembanding final |
| PATCH | `/api/local-files/{id}/comparables/{comparableId}` | Assigned | Ubah/hapus |
| DELETE | `/api/local-files/{id}/comparables/{comparableId}` | Assigned | Ubah/hapus |
| GET | `/api/local-files/{id}/interquartile-range` | Assigned | Min/Q1/median/Q2/Q3/max |
| POST | `/api/local-files/{id}/interquartile-range/recalculate` | Assigned | Hitung ulang |
| GET | `/api/organizations/{organizationId}/comparison-library` | Member | Pustaka read-only |
| POST | `/api/organizations/{organizationId}/comparison-library/{comparableId}/create-local-file` | Owner | Buat dokumen baru |

## 18. AI Business Process Generation

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| POST | `/api/local-files/{localFileId}/ai/business-process` | Assigned | Membuat generation |
| GET | `/api/ai-generations/{generationId}` | Tenant owner | Polling status/result |
| POST | `/api/ai-generations/{generationId}/accept` | Assigned | Terima hasil ke draft |
| POST | `/api/ai-generations/{generationId}/expire` | Assigned/System | Tandai tidak digunakan |
| GET | `/api/local-files/{localFileId}/ai-generations` | Assigned | Histori generation |

### 18.1 Create AI Generation

```ts
type CreateBusinessProcessGenerationRequest = {
  description: string;
  processSteps?: Array<{
    id: string;
    sequence: number;
    activity: string;
    actorOrDepartment?: string;
    description?: string;
  }>;
};
```

Response `202 Accepted` mengembalikan `generationId`, `status: QUEUED`, dan polling URL.

## 19. Review, Validation, dan Document Generation

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/local-files/{id}/review-summary` | Assigned | Preview ringkasan |
| GET | `/api/local-files/{id}/warnings` | Assigned | Warning rentang/data |
| POST | `/api/local-files/{id}/warnings/recalculate` | Assigned | Hitung ulang |
| POST | `/api/local-files/{id}/generations` | Assigned | Buat job DOCX/PDF |
| GET | `/api/document-generations/{generationId}` | Resource access | Status generation |
| GET | `/api/local-files/{id}/document-versions` | Resource access | Histori version |
| GET | `/api/local-files/{id}/document-versions/compare?fromVersionId={a}&toVersionId={b}` | Resource access | Diff baca saja dari dua snapshot hasil generate milik Local File yang sama; ringkasan bagian berubah dan nilai sebelum/sesudah |
| GET | `/api/document-versions/{versionId}` | Resource permission | Metadata version |
| GET | `/api/document-versions/{versionId}/download?format=docx\|pdf` | Resource permission | Stream DOCX/PDF setelah validasi paket, tenant, pembuat/penugasan/berbagi/pengambilalihan |
| POST | `/api/document-versions/{versionId}/clone` | Resource access + kuota | Clone sebagai dokumen baru dengan privasi awal sesuai pembuat baru |

Respons perbandingan memuat `fromVersionId`, `toVersionId`, `changedSections`, serta daftar perubahan per bagian (`path`, `before`, `after`, `changeType`). Nilai sensitif hanya dikirim kepada pengguna yang berhak membaca kedua versi. `400` untuk dua ID sama atau versi dari Local File berbeda; `404` untuk versi yang tidak ada atau tidak boleh diakses; `422` bila snapshot/schema tidak dapat dibandingkan. Hasil perbandingan tidak disimpan sebagai versi baru dan tidak memodifikasi dokumen. Pagination pada daftar History Versi menggunakan urutan `version_number DESC` dan cursor stabil.

## 20. Katalog Konsultasi untuk Pengguna

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/consultation-services` | User organisasi | Service version aktif |
| GET | `/api/consultation-services/{serviceId}` | User organisasi | Detail layanan aman |
| GET | `/api/document-versions/{versionId}/consultation-eligibility` | Resource access | Cek kelayakan berdasarkan pembuat, penugasan, berbagi, atau pengambilalihan |

Response katalog hanya memuat data display. Field internal, histori harga, margin, dan konfigurasi Admin tidak dikirim.

## 21. Consultation Request — User Induk dan Karyawan

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| POST | `/api/document-versions/{versionId}/consultation-requests` | Resource write | Membuat request pada dokumen yang boleh diubah |
| GET | `/api/organizations/{organizationId}/consultation-requests` | Resource scoped | Daftar request tanpa metadata request dokumen privat bagi User Induk yang belum diberi akses |
| GET | `/api/consultation-requests/{requestId}` | Owner/Requester | Detail dan status |
| PATCH | `/api/consultation-requests/{requestId}` | Requester | Update draft/revision requested |
| POST | `/api/consultation-requests/{requestId}/submit` | Requester | Kirim request quotation |
| POST | `/api/consultation-requests/{requestId}/cancel` | Requester/Owner | Batalkan bila diizinkan |
| GET | `/api/consultation-requests/{requestId}/quotation` | Owner/Requester | Quotation aktif |
| POST | `/api/consultation-requests/{requestId}/accept-quotation` | Owner/Requester allowed | Terima |
| POST | `/api/consultation-requests/{requestId}/reject-quotation` | Owner/Requester allowed | Tolak |
| POST | `/api/consultation-requests/{requestId}/payments` | Owner/Requester | Buat payment attempt |
| GET | `/api/consultation-requests/{requestId}/payments` | Owner/Requester | Histori attempt |
| POST | `/api/consultation-requests/{requestId}/refunds` | Owner/Payer | Ajukan refund |
| GET | `/api/consultation-requests/{requestId}/timeline` | Owner/Requester | Timeline status aman |
| GET | `/api/consultation-requests/{requestId}/eta` | Owner/Requester | ETA dan SLA events aman |
| GET | `/api/consultation-requests/{requestId}/review-summary` | Owner/Requester | Delivery/summary |
| GET | `/api/consultation-requests/{requestId}/comments` | Owner/Requester | Thread komentar |
| POST | `/api/consultation-comments/{commentId}/replies` | Owner/Requester | Balas thread |
| POST | `/api/consultation-comments/{commentId}/mark-addressed` | Owner/Requester | Tandai ditindaklanjuti |
| POST | `/api/consultation-requests/{requestId}/resubmit` | Owner/Requester | Ajukan version baru |
| GET | `/api/consultation-requests/{requestId}/revision-balance` | Owner/Requester | Revisi terpakai/tersisa |
| POST | `/api/consultation-requests/{requestId}/additional-review` | Owner/Requester | Buat order tambahan |
| GET | `/api/consultation-requests/{requestId}/workspace-deliveries` | Owner/Requester dengan hak dokumen | Draft Penyusunan Penuh; requester dengan hak sebagai pembuat atau penugasan aktif |
| GET | `/api/consultation-requests/{requestId}/workspace-deliveries/{deliveryId}` | Owner/Requester | Detail preview/change set sesuai akses request |
| POST | `/api/consultation-requests/{requestId}/workspace-deliveries/{deliveryId}/feedback` | Requester dengan hak ubah/Owner pengambil alih | Minta revisi |
| POST | `/api/consultation-requests/{requestId}/workspace-deliveries/{deliveryId}/approve` | Requester dengan hak ubah/Owner pengambil alih + re-auth | Setujui delivery tertentu |
| POST | `/api/consultation-requests/{requestId}/workspace/promote` | Requester dengan hak ubah/Owner pengambil alih + idempotency | Terapkan sebagai versi baru |

Pada feedback, approval, dan promotion, backend memvalidasi identitas `requested_by_user_id`, organisasi, Local File, hak pembuat atau penugasan Karyawan beserta hak ubah, delivery, dan versi workspace. User Induk hanya mengambil alih keputusan request Karyawan bila hak ubah pemohon telah dicabut; alasan dan aktor dicatat dalam audit. Promotion memvalidasi ulang hak dan approval untuk delivery yang sama, menggunakan transaksi dan idempotency agar retry mengembalikan versi hasil yang sama. Konsultan dan Karyawan tanpa hak ubah ditolak.

`Resource access` berarti pembuat dengan membership aktif, Karyawan dengan penugasan aktif pada dokumen buatan User Induk, User Induk pada dokumen buatannya atau yang dibagikan Karyawan, serta User Induk penerima pengambilalihan. `Resource write` mencakup hak yang sama untuk pengisian dan generate; `Resource owner` berarti pembuat aktif atau User Induk setelah pengambilalihan. Setiap endpoint Local File, file biner, versi, request konsultasi, quotation, payment, komentar, delivery, dan working copy memeriksa hak dokumen saat dibaca. Konsultan hanya memperoleh akses melalui assignment konsultasi yang sah. Pencabutan berbagi atau penugasan langsung menutup akses terkait.

Histori payment request berasal dari `consultation_request_payments` berurut `attempt_number`; status tiap attempt dibaca dari `payments`. Retry menambah attempt, tidak menimpa yang lama. Webhook yang sah mengaktifkan request sekali melalui lock dan idempotency. Detail delivery Review membaca isi komentar dari `review_delivery_comments` yang dibekukan pada waktu publish, sedangkan detail delivery Penyusunan Penuh membaca snapshot dari `consultation_workspace_deliveries`; perubahan komentar atau workspace berikutnya tidak mengubah respons delivery lama.

### 21.1 Create Consultation Request

```ts
type CreateConsultationRequest = {
  serviceId: string;
  notes?: string;
  requirements?: {
    requestedScope?: string[];
    entityCount?: number;
    specialNeeds?: string;
  };
};
```

Harga tidak terdapat pada payload.

## 22. Portal Konsultan

### 22.1 Profil dan Availability

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/consultant/profile` | Consultant | Profil sendiri |
| PATCH | `/api/consultant/profile` | Consultant | Biodata yang diizinkan |
| GET | `/api/consultant/availability` | Consultant | Daftar availability |
| POST | `/api/consultant/availability` | Consultant | Tambah periode |
| PATCH | `/api/consultant/availability/{availabilityId}` | Consultant | Update periode |
| DELETE | `/api/consultant/availability/{availabilityId}` | Consultant | Hapus periode |
| POST | `/api/consultant/profile-change-requests` | Consultant | Ajukan expertise/capacity change |
| GET | `/api/consultant/profile-change-requests` | Consultant | Status pengajuan |

### 22.2 Dashboard dan Assignment

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/consultant/dashboard` | Consultant | Ringkasan workload |
| GET | `/api/consultant/assignments` | Consultant | Daftar assignment sendiri |
| GET | `/api/consultant/assignments/{assignmentId}` | Consultant assignee | Detail scope/SLA |
| POST | `/api/consultant/assignments/{assignmentId}/accept` | Consultant assignee | Terima assignment |
| POST | `/api/consultant/assignments/{assignmentId}/decline` | Consultant assignee | Tolak dan alasan |
| POST | `/api/consultant/assignments/{assignmentId}/confirm-eta` | Consultant assignee | Konfirmasi ETA |
| POST | `/api/consultant/assignments/{assignmentId}/request-eta-change` | Consultant assignee | Usulkan ETA |
| POST | `/api/consultant/assignments/{assignmentId}/request-user-data` | Consultant assignee | Minta kelengkapan data kepada pengguna dan jeda SLA sesuai aturan |

### 22.3 Viewer, Comment, dan Review

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/consultant/requests/{requestId}/viewer` | Active assignment | Metadata viewer |
| GET | `/api/consultant/requests/{requestId}/viewer/sections` | Active assignment | Daftar section |
| GET | `/api/consultant/requests/{requestId}/viewer/sections/{sectionCode}` | Active assignment | Blocks dan thread |
| POST | `/api/consultant/requests/{requestId}/comments` | Active review round | Buat komentar |
| GET | `/api/consultant/comments/{commentId}` | Active assignment | Detail thread |
| PATCH | `/api/consultant/comments/{commentId}` | Author + unlocked | Edit draft komentar |
| DELETE | `/api/consultant/comments/{commentId}` | Author + draft | Hapus draft |
| POST | `/api/consultant/comments/{commentId}/replies` | Active assignment | Balas thread |
| PATCH | `/api/consultant/comment-replies/{replyId}` | Author + unlocked | Edit reply |
| POST | `/api/consultant/comments/{commentId}/resolve` | Active assignment | Resolve |
| POST | `/api/consultant/comments/{commentId}/reopen` | Active assignment | Reopen |
| GET | `/api/consultant/requests/{requestId}/review-checklist` | Active assignment | Checklist publish |
| POST | `/api/consultant/requests/{requestId}/deliver-review` | Active assignment | Publish immutable delivery |
| GET | `/api/consultant/requests/{requestId}/rounds` | Active assignment | Histori round |
| POST | `/api/consultant/requests/{requestId}/rounds/{roundId}/start` | Active assignment | Mulai re-review |

### 22.4 Working Copy Penyusunan Penuh

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/consultant/requests/{requestId}/workspace` | Active full-prep assignment | Working copy metadata |
| GET | `/api/consultant/workspaces/{workspaceId}/sections/{sectionCode}` | Active assignment | Data section |
| PATCH | `/api/consultant/workspaces/{workspaceId}/sections/{sectionCode}` | Active assignment + If-Match | Edit section |
| GET | `/api/consultant/workspaces/{workspaceId}/change-sets` | Active assignment | Histori perubahan |
| POST | `/api/consultant/workspaces/{workspaceId}/preview` | Active assignment | Buat preview draft |
| GET | `/api/consultant/workspace-previews/{previewId}` | Active assignment | Status/hasil preview |
| POST | `/api/consultant/workspaces/{workspaceId}/deliveries` | Active assignment | Serahkan draft |
| GET | `/api/consultant/workspaces/{workspaceId}/deliveries` | Active assignment | Histori delivery |
| GET | `/api/consultant/workspaces/{workspaceId}/feedback` | Active assignment | Feedback pemohon berwenang atau User Induk pengambil alih |
| POST | `/api/consultant/workspaces/{workspaceId}/close-response` | Active assignment | Tandai feedback ditangani |

## 23. Administrator — Dashboard dan Operasional

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/admin/dashboard` | Admin | Statistik agregat |
| GET | `/api/admin/dashboard/users` | Admin | Statistik pengguna |
| GET | `/api/admin/dashboard/organizations` | Admin | Statistik organisasi |
| GET | `/api/admin/dashboard/subscriptions` | Admin | Distribusi/status paket |
| GET | `/api/admin/dashboard/payments` | Admin | Status dan pendapatan agregat |
| GET | `/api/admin/dashboard/consultations` | Admin | Statistik konsultasi |
| GET | `/api/admin/reports/operational` | Admin | Laporan operasional |
| POST | `/api/admin/reports/operational/export` | Admin + re-auth | Ekspor aman |
| GET | `/api/admin/system/status` | Admin | Status aplikasi/integrasi |
| GET | `/api/admin/integrations` | Admin | Status integration |
| POST | `/api/admin/maintenance/enable` | Admin + re-auth | Aktifkan maintenance |
| POST | `/api/admin/maintenance/disable` | Admin + re-auth | Nonaktifkan |
| GET | `/api/admin/maintenance` | Admin | Status/pemberitahuan |
| PATCH | `/api/admin/settings/upload-policy` | Admin + re-auth | Tipe/ukuran upload |
| GET | `/api/admin/settings/notification-templates/{templateCode}` | Admin | Template notifikasi |
| PATCH | `/api/admin/settings/notification-templates/{templateCode}` | Admin | Template notifikasi |
| GET | `/api/admin/settings/history` | Admin | Histori perubahan |

## 24. Administrator — Plans, Billing, dan Accounts

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/admin/plans` | Admin | Daftar/buat paket |
| POST | `/api/admin/plans` | Admin | Daftar/buat paket |
| GET | `/api/admin/plans/{planId}` | Admin | Detail/update paket |
| PATCH | `/api/admin/plans/{planId}` | Admin | Detail/update paket |
| POST | `/api/admin/plans/{planId}/activate` | Admin + re-auth | Aktifkan version |
| POST | `/api/admin/plans/{planId}/deactivate` | Admin + re-auth | Nonaktifkan |
| GET | `/api/admin/plans/{planId}/quotas` | Admin | Kuota dokumen/kursi |
| PATCH | `/api/admin/plans/{planId}/quotas` | Admin | Kuota dokumen/kursi |
| GET | `/api/admin/payments` | Admin | Payment operasional |
| GET | `/api/admin/payments/{paymentId}` | Admin | Metadata payment |
| GET | `/api/admin/refunds` | Admin | Queue refund |
| GET | `/api/admin/refunds/{refundId}` | Admin | Detail refund |
| POST | `/api/admin/refunds/{refundId}/review` | Admin | Mulai review |
| POST | `/api/admin/refunds/{refundId}/approve` | Admin + second approval | Setujui |
| POST | `/api/admin/refunds/{refundId}/reject` | Admin + reason | Tolak |
| POST | `/api/admin/refunds/{refundId}/process` | Admin + idempotency | Proses provider |
| GET | `/api/admin/withdrawals` | Admin | Daftar settlement/withdrawal |
| POST | `/api/admin/withdrawals` | Admin + dual approval | Ajukan pencairan bila didukung |
| GET | `/api/admin/withdrawals/{withdrawalId}` | Admin | Status payout, ETA, dan failure code aman |
| GET | `/api/admin/payment-disputes` | Admin | Daftar dispute dan tenggat tindakan |
| GET | `/api/admin/payment-disputes/{disputeId}` | Admin | Detail dispute serta evidence requirement |
| POST | `/api/admin/payment-disputes/{disputeId}/evidences` | Admin + re-auth | Upload evidence privat melalui NestJS |
| POST | `/api/admin/payment-disputes/{disputeId}/submit-evidence` | Admin + re-auth | Kirim evidence sesuai ketentuan provider |
| POST | `/api/admin/payment-disputes/{disputeId}/accept` | Admin + dual approval | Terima dispute bila kebijakan mengizinkan |
| GET | `/api/admin/users` | Admin | Daftar metadata pengguna |
| GET | `/api/admin/users/{userId}` | Admin | Detail status/membership |
| POST | `/api/admin/users/{userId}/send-password-reset` | Admin | Kirim reset link |
| POST | `/api/admin/users/{userId}/reset-mfa` | Admin + verified identity | Reset MFA |
| POST | `/api/admin/users/{userId}/suspend` | Admin + reason | Suspend |
| POST | `/api/admin/users/{userId}/reactivate` | Admin + reason | Reaktivasi |
| POST | `/api/admin/users/{userId}/revoke-sessions` | Admin + re-auth | Cabut sesi |
| GET | `/api/admin/organizations` | Admin | Daftar metadata organisasi |
| GET | `/api/admin/organizations/{organizationId}` | Admin | Detail operasional tanpa isi dokumen |

## 25. Administrator — Enterprise/Custom

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/admin/custom-plan-requests` | Admin | Queue pengajuan |
| GET | `/api/admin/custom-plan-requests/{requestId}` | Admin | Detail metadata |
| POST | `/api/admin/custom-plan-requests/{requestId}/start-review` | Admin | Under Review |
| POST | `/api/admin/custom-plan-requests/{requestId}/request-revision` | Admin | Minta revisi |
| POST | `/api/admin/custom-plan-requests/{requestId}/quotations` | Admin | Buat quotation |
| POST | `/api/admin/custom-plan-requests/{requestId}/approve` | Admin | Setujui pengajuan |
| POST | `/api/admin/custom-plan-requests/{requestId}/reject` | Admin | Tolak |
| POST | `/api/admin/custom-plan-requests/{requestId}/activate` | Admin + paid | Aktivasi kontrak |

## 26. Administrator — Consultant Management

### 26.1 Akun Konsultan

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/admin/consultants` | Admin | Daftar Konsultan |
| POST | `/api/admin/consultants/invitations` | Admin | Kirim undangan |
| GET | `/api/admin/consultants/invitations` | Admin | Histori/status undangan |
| POST | `/api/admin/consultants/invitations/{invitationId}/resend` | Admin | Kirim ulang |
| DELETE | `/api/admin/consultants/invitations/{invitationId}` | Admin | Batalkan |
| GET | `/api/admin/consultants/{consultantId}` | Admin | Profil/workload |
| PATCH | `/api/admin/consultants/{consultantId}` | Admin | Expertise/capacity |
| POST | `/api/admin/consultants/{consultantId}/activate` | Admin | Aktifkan |
| POST | `/api/admin/consultants/{consultantId}/suspend` | Admin + reason | Suspend dan revoke access |
| POST | `/api/admin/consultants/{consultantId}/reactivate` | Admin | Reaktivasi |
| GET | `/api/admin/consultants/{consultantId}/assignments` | Admin | Workload/history |
| GET | `/api/admin/consultant-profile-change-requests` | Admin | Pengajuan perubahan |
| POST | `/api/admin/consultant-profile-change-requests/{requestId}/approve` | Admin | Setujui |
| POST | `/api/admin/consultant-profile-change-requests/{requestId}/reject` | Admin | Tolak |

### 26.2 Katalog Konsultasi

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/admin/consultation-services` | Admin | Semua layanan/version |
| POST | `/api/admin/consultation-services` | Admin | Buat layanan/version awal |
| GET | `/api/admin/consultation-services/{serviceId}` | Admin | Detail dan histori |
| PATCH | `/api/admin/consultation-services/{serviceId}` | Admin | Buat version perubahan |
| POST | `/api/admin/consultation-services/{serviceId}/activate` | Admin + re-auth | Aktifkan current version |
| POST | `/api/admin/consultation-services/{serviceId}/deactivate` | Admin + re-auth | Nonaktifkan order baru |
| GET | `/api/admin/consultation-services/{serviceId}/versions` | Admin | Histori version |

### 26.3 Request, Quotation, Assignment, dan SLA

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/admin/consultation-requests` | Admin | Queue request |
| GET | `/api/admin/consultation-requests/{requestId}` | Admin | Metadata/status |
| POST | `/api/admin/consultation-requests/{requestId}/start-review` | Admin | Review quotation |
| POST | `/api/admin/consultation-requests/{requestId}/request-revision` | Admin | Minta kelengkapan |
| POST | `/api/admin/consultation-requests/{requestId}/reject` | Admin | Tolak |
| POST | `/api/admin/consultation-requests/{requestId}/quotations` | Admin | Buat quotation |
| GET | `/api/admin/consultation-requests/{requestId}/quotations` | Admin | Histori quotation |
| POST | `/api/admin/consultation-requests/{requestId}/assignments` | Admin | Assign Konsultan |
| POST | `/api/admin/consultation-assignments/{assignmentId}/reassign` | Admin | Reassign |
| POST | `/api/admin/consultation-assignments/{assignmentId}/revoke` | Admin + reason | Cabut |
| POST | `/api/admin/consultation-assignments/{assignmentId}/eta-change/approve` | Admin | Setujui ETA |
| POST | `/api/admin/consultation-assignments/{assignmentId}/eta-change/reject` | Admin | Tolak ETA |
| GET | `/api/admin/consultation-requests/{requestId}/sla-events` | Admin | Timeline SLA |
| POST | `/api/admin/consultation-requests/{requestId}/close` | Admin | Tutup sesuai state |

## 27. Administrator — Security, Files, Backup, dan Audit

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/admin/audit-logs` | Admin | Audit terfilter |
| GET | `/api/admin/security/suspicious-logins` | Admin | Login mencurigakan |
| POST | `/api/admin/security/sessions/{sessionId}/revoke` | Admin | Cabut sesi |
| GET | `/api/admin/files/quarantine` | Admin | File dikarantina |
| DELETE | `/api/admin/files/quarantine/{fileId}` | Admin + re-auth | Hapus file berbahaya |
| POST | `/api/admin/files/quarantine/{fileId}/release` | Dual approval | Lepaskan setelah pemeriksaan |
| GET | `/api/admin/backups` | Admin | Status backup |
| POST | `/api/admin/backups/restore-tests` | Admin + approval | Uji restore non-production |
| GET | `/api/admin/retention` | Admin | Data dalam retensi |
| GET | `/api/admin/deletion-jobs` | Admin | Status penghapusan |
| GET | `/api/admin/integration-events` | Admin | Kegagalan integrasi |
| POST | `/api/admin/integration-events/{eventId}/retry` | Admin | Retry aman/idempotent |

## 28. Webhook Endpoints

| Method | Endpoint | Verifikasi | Fungsi |
| --- | --- | --- | --- |
| POST | `/api/webhooks/payments/xendit` | Verification token/signature dan provider ID | Payment, refund, dispute, atau payout event |
| POST | `/api/webhooks/email/resend` | Signature provider | Delivery/bounce/complaint |
| POST | `/api/webhooks/file-processing` | Internal signature bila digunakan | Scan/processing callback |
| POST | `/api/webhooks/malware-scanner` | Service credential | Hasil scan |

Aturan:

- Simpan event ID provider.
- Verifikasi sebelum parsing bisnis.
- Gunakan database lock/idempotency.
- Kembalikan 2xx untuk event duplikat yang telah sukses.
- Jangan log payload rahasia.
- Webhook tidak menggunakan session browser.

## 29. Internal Worker dan AI Service

Endpoint berikut tidak dipublikasikan ke internet:

| Method | Endpoint | Caller | Fungsi |
| --- | --- | --- | --- |
| POST | `/internal/ai/business-process/generate` | AI Worker | Generate fase |
| GET | `/internal/health/live` | Monitor internal | Liveness FastAPI |
| GET | `/internal/health/ready` | Worker/monitor | Model readiness |
| POST | `/internal/documents/convert-pdf` | Document Worker bila service terpisah | Convert DOCX |
| POST | `/internal/files/scan` | File Worker | Malware scan |
| POST | `/internal/notifications/send` | Worker internal bila service dipisah | Kirim email |

Komunikasi memakai private network dan service credential. Browser tidak menerima URL internal.

## 30. Payload Penting Konsultasi

### 30.1 Service Input

```ts
type ConsultationServiceInput = {
  code: string;
  serviceType: 'TP_REVIEW' | 'FULL_PREPARATION' | 'ADDITIONAL_REVIEW';
  name: string;
  description: string;
  pricingMode: 'FIXED' | 'STARTING_FROM' | 'QUOTATION';
  currency: string;
  displayPriceMinor?: number;
  minimumPriceMinor?: number;
  defaultSlaBusinessDays: number;
  reReviewSlaBusinessDays?: number;
  includedRevisionRounds: number;
};
```

### 30.2 Comment Input

```ts
type CreateReviewCommentRequest = {
  roundId: string;
  anchor: {
    documentVersionId: string;
    previewBlockId: string;
    sectionCode: string;
    blockKey: string;
    startOffset: number;
    endOffset: number;
    selectedText: string;
    selectedTextHash: string;
  };
  category: string;
  severity: 'INFO' | 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  body: string;
};
```

### 30.3 Deliver Review

```ts
type DeliverReviewRequest = {
  roundId: string;
  conclusion: string;
  outcome: 'COMPLETED' | 'REQUIRES_RESUBMISSION';
  expectedCommentVersions: Array<{ commentId: string; version: number }>;
};
```

### 30.4 ETA Change

```ts
type EtaChangeRequest = {
  proposedDueAt: string;
  reason: string;
};
```

### 30.5 Workspace Section Update

```ts
type UpdateWorkspaceSectionRequest = {
  expectedWorkspaceVersion: number;
  changes: Array<{
    operation: 'ADD' | 'REPLACE' | 'REMOVE';
    path: string;
    value?: unknown;
  }>;
};
```

## 31. HTTP Status

| HTTP | Penggunaan |
| --- | --- |
| 200 | Read/update sukses |
| 201 | Resource dibuat |
| 202 | Job asynchronous diterima |
| 204 | Delete/action tanpa body |
| 400 | Payload tidak dapat diproses |
| 401 | Tidak terautentikasi |
| 403 | Tidak memiliki permission |
| 404 | Resource tidak ditemukan/tidak boleh dienumerasi |
| 409 | State/version/idempotency conflict |
| 410 | Token/resource kedaluwarsa |
| 413 | File terlalu besar |
| 415 | Media type tidak didukung |
| 422 | Validasi domain |
| 429 | Rate limit |
| 500 | Internal safe error |
| 502/503 | Dependency tidak tersedia |

## 32. Error Codes Utama

### 32.1 Identity dan Tenant

- `AUTH_INVALID_CREDENTIALS`
- `AUTH_MFA_REQUIRED`
- `AUTH_MFA_INVALID`
- `AUTH_SESSION_EXPIRED`
- `ACCOUNT_SUSPENDED`
- `TENANT_ACCESS_DENIED`
- `DOCUMENT_ASSIGNMENT_REQUIRED`

### 32.2 Billing

- `PAYMENT_NOT_FOUND`
- `PAYMENT_AMOUNT_MISMATCH`
- `PAYMENT_NOT_PAID`
- `PAYMENT_ALREADY_PROCESSED`
- `PAYMENT_SESSION_EXPIRED`
- `PAYMENT_PROVIDER_MISMATCH`
- `PAYMENT_WEBHOOK_INVALID`
- `REFUND_NOT_ELIGIBLE`
- `REFUND_ALREADY_PROCESSED`
- `DISPUTE_ACTION_REQUIRED`
- `DISPUTE_EVIDENCE_INVALID`
- `PAYOUT_DISABLED`
- `PAYOUT_DESTINATION_NOT_ALLOWED`
- `QUOTATION_EXPIRED`

### 32.3 Local File dan Generation

- `LOCAL_FILE_NOT_READY`
- `DOCUMENT_QUOTA_EXHAUSTED`
- `DOCUMENT_GENERATION_FAILED`
- `DOCUMENT_VERSION_NOT_FOUND`
- `AI_TIMEOUT`
- `AI_RATE_LIMITED`
- `AI_OUTPUT_INVALID`
- `AI_MODEL_UNAVAILABLE`
- `AI_CAPACITY_FULL`

### 32.4 Consultation

- `CONSULTATION_SERVICE_INACTIVE`
- `CONSULTATION_REQUEST_STATE_INVALID`
- `CONSULTATION_REVISION_EXHAUSTED`
- `ASSIGNMENT_ACCESS_DENIED`
- `ASSIGNMENT_ALREADY_DECIDED`
- `CONSULTANT_CAPACITY_FULL`
- `ETA_CHANGE_PENDING`
- `REVIEW_ROUND_NOT_ACTIVE`
- `PREVIEW_NOT_READY`
- `COMMENT_ANCHOR_INVALID`
- `COMMENT_VERSION_CONFLICT`
- `COMMENT_LOCKED`
- `DELIVERY_ALREADY_PUBLISHED`
- `WORKSPACE_VERSION_CONFLICT`
- `WORKSPACE_NOT_APPROVED`
- `PROMOTION_ALREADY_COMPLETED`
- `ACCESS_REVOKED`

## 33. Rate Limiting

| Endpoint group | Kebijakan awal |
| --- | --- |
| Login/reset/MFA | Ketat per IP dan account hash |
| Upload intent | Per user/organization |
| AI generation | Per paket dan organisasi |
| Payment creation | Per request dan idempotency key |
| Comment/reply | Per assignment dan user |
| Report export | Per Administrator |
| Webhook | Provider-aware; jangan memblokir retry sah |

Nilai aktual disimpan sebagai konfigurasi dan ditentukan melalui pengujian.

## 34. Audit Requirements

Mutation berikut wajib mencatat audit:

- perubahan akun, MFA, sesi, role, dan status;
- organisasi, membership, invitation, dan document assignment;
- paket, kuota, payment, refund, dan withdrawal;
- Local File, upload, import, generate, download, dan delete;
- katalog konsultasi dan quotation;
- assignment, ETA, pause/resume, dan access revoke;
- viewer access sensitif;
- komentar, status, delivery, resubmission;
- working copy, approval, dan promotion;
- tindakan Administrator.

Audit tidak menyimpan password, token, full document text, comment body penuh, NPWP lengkap, atau provider secret.

## 35. Contract Testing dan Acceptance

- Seluruh endpoint terlindungi menolak token invalid.
- Tenant A tidak dapat mengakses resource tenant B.
- Karyawan tanpa hak sebagai pembuat atau penerima document assignment aktif ditolak.
- Konsultan tanpa active assignment ditolak.
- DTO Konsultan tidak memiliki field harga/payment.
- Harga checkout tidak dapat dimanipulasi frontend.
- Webhook duplikat idempotent.
- AI/document generation retry tidak menggandakan job atau kuota.
- Comment anchor dan version conflict tervalidasi.
- Working copy update menggunakan optimistic concurrency.
- Promotion tanpa approval ditolak.
- Admin tidak dapat menetapkan password pengguna.
- Response tidak mengandung secret atau stack trace.
- OpenAPI generated contract sesuai tabel endpoint ini.
- Integration test memverifikasi status code dan error code utama.

## 36. Referensi

- [System Architecture](./System-Architecture.md)
- [Technology Stack](./Technology-Stack.md)
- [Consultation Architecture](./Consultation-Architecture.md)
- [Document Review Architecture](./Document-Review-Architecture.md)
- [AI Processing Architecture](./AI-Processing-Architecture.md)
- [Document Generation Architecture](./Document-Generation-Architecture.md)
- [ERD](../04-Database/ERD.dbml)
- [Functional Requirements](../01-Requirements/Functional-Requirements.md)
