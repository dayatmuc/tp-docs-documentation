# API Contract — TP Docs By System
## Diskon Event Opsional

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | /api/discounts/active | Pengguna | Estimasi harga normal atau promo untuk produk |
| GET | /api/admin/discount-campaigns | Admin | Daftar dan penggunaan event |
| POST | /api/admin/discount-campaigns | Admin | Buat draft |
| PATCH | /api/admin/discount-campaigns/{campaignId} | Admin | Ubah draft dan target |
| POST | /api/admin/discount-campaigns/{campaignId}/activate | Admin + re-auth | Aktifkan |
| POST | /api/admin/discount-campaigns/{campaignId}/deactivate | Admin + re-auth | Nonaktifkan |

Checkout dan quotation memakai endpoint yang ada. Backend mengembalikan originalAmountMinor, discountAmountMinor bila ada, totalAmountMinor, currency, dan discountCampaignId opsional. Nominal browser diabaikan. Xendit menerima total snapshot; retry memakai snapshot sama; refund memakai nilai dibayar.


## 1. Tujuan dan Cakupan

Dokumen ini mendefinisikan REST API TP Docs untuk:

- publik dan autentikasi;
- User Induk;
- Karyawan;
- Administrator;
- organisasi dan membership;
- subscription, billing, payment, refund, dan withdrawal;
- Local File, transaksi, FAR, benchmarking, AI, dan document generation;
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
5. Payment, refund, document generation, dan webhook menggunakan idempotency.
6. File privat diunggah dan di-stream melalui NestJS setelah validasi tenant dan hak akses; browser tidak mengakses PostgreSQL langsung.
7. Response tidak memuat password hash, secret, token provider, atau data finansial mentah.
8. Resource tenant lain dikembalikan sebagai `404` atau `403` sesuai kebijakan anti-enumeration.
9. Tanggal menggunakan ISO-8601 UTC.

## 3. Authentication dan Header

### 3.1 Header Umum

| Header | Wajib | Fungsi |
| --- | --- | --- |
| `Authorization: Bearer <token>` | Ya untuk endpoint terlindungi | Access token/session |
| `Content-Type: application/json` | Mutation JSON | Format payload |
| `Idempotency-Key` | Checkout, payment, refund, AI, dan document generation | Mencegah operasi ganda |
| `X-Correlation-Id` | Opsional dari aplikasi pemanggil | Trace request; server membuat jika kosong |
| `If-Match` | Update optimistic | Version/ETag resource |
| `Accept-Language` | Opsional | Locale pesan aman |

Aplikasi pemanggil tidak boleh menggunakan header role atau tenant sebagai sumber otorisasi.

### 3.2 Authentication Assurance

- User Induk: MFA wajib.
- Karyawan: MFA opsional pada versi awal.
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

| Kemampuan | User Induk | Karyawan | Administrator |
| --- | --- | --- | --- |
| Profil pribadi | Akun sendiri | Akun sendiri | Akun Administrator sendiri |
| Profil organisasi | Kelola | Baca sesuai kebutuhan dokumen | Metadata operasional; tidak mengubah kepemilikan |
| Anggota organisasi | Kelola | Tidak | Tidak mengundang, menghapus, atau menugaskan anggota |
| Local File | Seluruh dokumen organisasi | Hanya dokumen yang ditugaskan | Tidak otomatis dapat membaca isi dokumen |
| Langganan dan kuota | Kelola | Tidak | Kelola katalog dan koreksi terotorisasi |
| Pembayaran dan refund | Kelola dan ajukan | Tidak | Proses operasional dengan audit dan approval |
| Enterprise/Custom | Mengajukan dan memutuskan quotation | Tidak | Meninjau, membuat quotation, dan mengaktifkan setelah pembayaran |
| Pengaturan platform | Tidak | Tidak | Sesuai kewenangan Administrator |

## 7. Public dan System Endpoints

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/health/live` | Tidak | Liveness aplikasi |
| GET | `/api/health/ready` | Internal/monitor | Readiness dependency utama |
| GET | `/api/public/plans` | Tidak | Daftar paket aktif |
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
| POST | `/api/me/avatar/upload-url` | Ya | Signed upload avatar |
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

### 11.1 Plans dan Subscription

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/plans` | Ya | Paket aktif |
| GET | `/api/plans/{planCode}` | Ya | Detail paket |
| GET | `/api/organizations/{organizationId}/subscription` | Owner | Paket dan masa aktif |
| POST | `/api/organizations/{organizationId}/subscription/checkout` | Owner | Checkout paket |
| POST | `/api/organizations/{organizationId}/subscription/upgrade` | Owner | Upgrade paket |
| POST | `/api/organizations/{organizationId}/subscription/cancel` | Owner | Batalkan renewal sesuai kebijakan |
| GET | `/api/organizations/{organizationId}/quotas` | Owner | Kuota dokumen/kursi |
| GET | `/api/organizations/{organizationId}/quota-ledger` | Owner | Histori perubahan kuota |
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
| POST | `/api/organizations/{organizationId}/refunds/{refundId}/evidence/upload-url` | Owner/payer | Upload bukti |
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

Upload memakai field file serta metadata `purpose`, organization context, dan target resource. Backend memvalidasi MIME, ekstensi, ukuran, checksum, lalu menyimpan konten sebagai chunk `bytea` 1 MiB. File baru berstatus lengkap setelah seluruh chunk dan checksum tervalidasi. Respons konten mengirim `Content-Type`, `Content-Length`, `Content-Disposition`, dan checksum sebagai `ETag`.

## 13. Local File Core

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/organizations/{organizationId}/local-files` | Member | Daftar sesuai permission |
| POST | `/api/organizations/{organizationId}/local-files` | Owner | Buat Local File dan memakai kuota setelah transaksi database berhasil |
| GET | `/api/local-files/{localFileId}` | Assigned | Detail dan progress |
| PATCH | `/api/local-files/{localFileId}` | Assigned | Update metadata |
| DELETE | `/api/local-files/{localFileId}` | Owner | Hapus sesuai retensi |
| POST | `/api/local-files/{localFileId}/duplicate` | Owner/Assigned | Duplikasi |
| GET | `/api/local-files/{localFileId}/sections` | Assigned | Progress section |
| PATCH | `/api/local-files/{localFileId}/sections/{sectionCode}` | Assigned | Update status section |
| POST | `/api/local-files/{localFileId}/validate` | Assigned | Validasi kelengkapan |
| GET | `/api/local-files/{localFileId}/history` | Assigned | Histori perubahan |
| GET | `/api/local-files/{localFileId}/audit-events` | Owner | Audit dokumen |

## 14. Profil Perusahaan, Kepemilikan, dan Bisnis

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/local-files/{id}/company-profile` | Assigned | Profil perusahaan |
| PUT | `/api/local-files/{id}/company-profile` | Assigned | Profil perusahaan |
| GET | `/api/local-files/{id}/shareholders` | Assigned | Daftar pemegang saham |
| POST | `/api/local-files/{id}/shareholders` | Assigned | Tambah pemegang saham |
| PATCH | `/api/local-files/{id}/shareholders/{shareholderId}` | Assigned | Ubah pemegang saham |
| DELETE | `/api/local-files/{id}/shareholders/{shareholderId}` | Assigned | Hapus pemegang saham |
| GET | `/api/local-files/{id}/management` | Assigned | Daftar komisaris dan direksi |
| POST | `/api/local-files/{id}/management` | Assigned | Tambah komisaris atau direksi |
| PATCH | `/api/local-files/{id}/management/{managementId}` | Assigned | Ubah data pengurus |
| DELETE | `/api/local-files/{id}/management/{managementId}` | Assigned | Hapus data pengurus |
| GET | `/api/local-files/{id}/business-profile` | Assigned | Uraian bisnis dan strategi |
| PUT | `/api/local-files/{id}/business-profile` | Assigned | Uraian bisnis dan strategi |
| GET | `/api/local-files/{id}/business-process-steps` | Assigned | Langkah proses |
| POST | `/api/local-files/{id}/business-process-steps` | Assigned | Langkah proses |
| PATCH | `/api/local-files/{id}/business-process-steps/{stepId}` | Assigned | Ubah langkah proses bisnis |
| DELETE | `/api/local-files/{id}/business-process-steps/{stepId}` | Assigned | Hapus langkah proses bisnis |
| POST | `/api/local-files/{id}/business-process-steps/reorder` | Assigned | Ubah urutan |
| GET | `/api/local-files/{id}/financial-summary` | Assigned | Ringkasan keuangan |
| PUT | `/api/local-files/{id}/financial-summary` | Assigned | Ringkasan keuangan |
| GET | `/api/local-files/{id}/affiliates` | Assigned | Pihak afiliasi |
| POST | `/api/local-files/{id}/affiliates` | Assigned | Pihak afiliasi |
| PATCH | `/api/local-files/{id}/affiliates/{affiliateId}` | Assigned | Ubah pihak afiliasi |
| DELETE | `/api/local-files/{id}/affiliates/{affiliateId}` | Assigned | Hapus pihak afiliasi |
| GET | `/api/local-files/{id}/competitors` | Assigned | Pesaing utama |
| POST | `/api/local-files/{id}/competitors` | Assigned | Pesaing utama |
| PATCH | `/api/local-files/{id}/competitors/{competitorId}` | Assigned | Ubah pesaing utama |
| DELETE | `/api/local-files/{id}/competitors/{competitorId}` | Assigned | Hapus pesaing utama |
| GET | `/api/local-files/{id}/industry-overview` | Assigned | Gambaran industri |
| PUT | `/api/local-files/{id}/industry-overview` | Assigned | Gambaran industri |
| POST | `/api/local-files/{id}/industry-overview/files` | Assigned | Kaitkan file Word/PDF |
| DELETE | `/api/local-files/{id}/industry-overview/files/{fileId}` | Assigned | Lepas file |
| GET | `/api/local-files/{id}/non-financial-events` | Assigned | Kejadian nonkeuangan |
| POST | `/api/local-files/{id}/non-financial-events` | Assigned | Kejadian nonkeuangan |
| PATCH | `/api/local-files/{id}/non-financial-events/{eventId}` | Assigned | Ubah kejadian nonkeuangan |
| DELETE | `/api/local-files/{id}/non-financial-events/{eventId}` | Assigned | Hapus kejadian nonkeuangan |

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
| GET | `/api/local-files/{id}/affiliate-transactions` | Assigned | Daftar transaksi afiliasi |
| POST | `/api/local-files/{id}/affiliate-transactions` | Assigned | Tambah transaksi afiliasi |
| PATCH | `/api/local-files/{id}/affiliate-transactions/{transactionId}` | Assigned | Ubah transaksi afiliasi |
| DELETE | `/api/local-files/{id}/affiliate-transactions/{transactionId}` | Assigned | Hapus transaksi afiliasi |
| GET | `/api/local-files/{id}/independent-transactions` | Assigned | Daftar transaksi independen |
| POST | `/api/local-files/{id}/independent-transactions` | Assigned | Tambah transaksi independen |
| PATCH | `/api/local-files/{id}/independent-transactions/{transactionId}` | Assigned | Ubah transaksi independen |
| DELETE | `/api/local-files/{id}/independent-transactions/{transactionId}` | Assigned | Hapus transaksi independen |
| GET | `/api/local-files/{id}/transaction-summary` | Assigned | Summary auto-generated |
| POST | `/api/local-files/{id}/transaction-summary/recalculate` | Assigned | Hitung ulang |

## 16. FAR dan Metode TP

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/local-files/{id}/far-analysis` | Assigned | Root FAR |
| PUT | `/api/local-files/{id}/far-analysis` | Assigned | Root FAR |
| GET | `/api/local-files/{id}/far/functions` | Assigned | Fungsi |
| POST | `/api/local-files/{id}/far/functions` | Assigned | Fungsi |
| PATCH | `/api/local-files/{id}/far/functions/{functionId}` | Assigned | Ubah fungsi yang dilakukan |
| DELETE | `/api/local-files/{id}/far/functions/{functionId}` | Assigned | Hapus fungsi yang dilakukan |
| GET | `/api/local-files/{id}/far/assets` | Assigned | Aset |
| POST | `/api/local-files/{id}/far/assets` | Assigned | Aset |
| PATCH | `/api/local-files/{id}/far/assets/{assetId}` | Assigned | Ubah struktur dan nilai aset |
| DELETE | `/api/local-files/{id}/far/assets/{assetId}` | Assigned | Hapus data aset |
| GET | `/api/local-files/{id}/far/risks` | Assigned | Risiko |
| POST | `/api/local-files/{id}/far/risks` | Assigned | Risiko |
| PATCH | `/api/local-files/{id}/far/risks/{riskId}` | Assigned | Ubah risiko yang ditanggung |
| DELETE | `/api/local-files/{id}/far/risks/{riskId}` | Assigned | Hapus risiko yang ditanggung |
| GET | `/api/local-files/{id}/tp-method-selection` | Assigned | Metode terpilih, tested party, PLI, alasan |
| PUT | `/api/local-files/{id}/tp-method-selection` | Assigned | Metode terpilih, tested party, PLI, alasan |
| GET | `/api/local-files/{id}/tp-method-evaluations` | Assigned | Evaluasi metode |
| POST | `/api/local-files/{id}/tp-method-evaluations` | Assigned | Evaluasi metode |
| PATCH | `/api/local-files/{id}/tp-method-evaluations/{evaluationId}` | Assigned | Ubah evaluasi metode TP |
| DELETE | `/api/local-files/{id}/tp-method-evaluations/{evaluationId}` | Assigned | Hapus evaluasi metode TP |

## 17. Benchmarking dan Pustaka Pembanding

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/local-files/{id}/benchmarking` | Assigned | Root benchmarking |
| PUT | `/api/local-files/{id}/benchmarking` | Assigned | Root benchmarking |
| POST | `/api/local-files/{id}/benchmarking/files` | Assigned | Upload/tautkan PDF |
| GET | `/api/local-files/{id}/benchmark-search-steps` | Assigned | Langkah pencarian |
| POST | `/api/local-files/{id}/benchmark-search-steps` | Assigned | Langkah pencarian |
| PATCH | `/api/local-files/{id}/benchmark-search-steps/{stepId}` | Assigned | Ubah langkah pencarian pembanding |
| DELETE | `/api/local-files/{id}/benchmark-search-steps/{stepId}` | Assigned | Hapus langkah pencarian pembanding |
| GET | `/api/local-files/{id}/benchmark-search-results` | Assigned | Hasil perusahaan |
| POST | `/api/local-files/{id}/benchmark-search-results` | Assigned | Hasil perusahaan |
| PATCH | `/api/local-files/{id}/benchmark-search-results/{resultId}` | Assigned | Ubah hasil perusahaan pembanding |
| DELETE | `/api/local-files/{id}/benchmark-search-results/{resultId}` | Assigned | Hapus hasil perusahaan pembanding |
| GET | `/api/local-files/{id}/pli-selection` | Assigned | Pilih satu PLI |
| PUT | `/api/local-files/{id}/pli-selection` | Assigned | Pilih satu PLI |
| GET | `/api/local-files/{id}/comparables` | Assigned | Pembanding final |
| POST | `/api/local-files/{id}/comparables` | Assigned | Pembanding final |
| PATCH | `/api/local-files/{id}/comparables/{comparableId}` | Assigned | Ubah pembanding final |
| DELETE | `/api/local-files/{id}/comparables/{comparableId}` | Assigned | Hapus pembanding final |
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
| GET | `/api/document-generations/{generationId}` | Tenant owner | Status generation |
| GET | `/api/local-files/{id}/document-versions` | Assigned | Histori version |
| GET | `/api/document-versions/{versionId}` | Resource permission | Metadata version |
| GET | `/api/document-versions/{versionId}/download?format=docx\|pdf` | Resource permission | Stream DOCX/PDF setelah validasi paket dan akses |
| POST | `/api/document-versions/{versionId}/clone` | Owner/Assigned | Clone sebagai dokumen baru |

## 20. Administrator — Dashboard dan Operasional

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/admin/dashboard` | Admin | Statistik agregat |
| GET | `/api/admin/dashboard/users` | Admin | Statistik pengguna |
| GET | `/api/admin/dashboard/organizations` | Admin | Statistik organisasi |
| GET | `/api/admin/dashboard/subscriptions` | Admin | Distribusi/status paket |
| GET | `/api/admin/dashboard/payments` | Admin | Status dan pendapatan agregat |
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

## 21. Administrator — Plans, Billing, dan Accounts

| Method | Endpoint | Auth | Fungsi |
| --- | --- | --- | --- |
| GET | `/api/admin/plans` | Admin | Daftar paket |
| POST | `/api/admin/plans` | Admin | Buat paket dan harga awal |
| GET | `/api/admin/plans/{planId}` | Admin | Detail paket |
| PATCH | `/api/admin/plans/{planId}` | Admin | Ubah paket melalui konfigurasi terversi |
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
| POST | `/api/admin/payment-disputes/{disputeId}/evidences/upload-url` | Admin + re-auth | Upload evidence privat |
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

## 22. Administrator — Enterprise/Custom

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

## 23. Administrator — Security, Files, Backup, dan Audit

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

## 24. Webhook Endpoints

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

## 25. Internal Worker dan AI Service

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

## 26. HTTP Status

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

## 27. Error Codes Utama

### 27.1 Identity dan Tenant

- `AUTH_INVALID_CREDENTIALS`
- `AUTH_MFA_REQUIRED`
- `AUTH_MFA_INVALID`
- `AUTH_SESSION_EXPIRED`
- `ACCOUNT_SUSPENDED`
- `TENANT_ACCESS_DENIED`
- `DOCUMENT_ASSIGNMENT_REQUIRED`

### 27.2 Billing

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

### 27.3 Local File dan Generation

- `LOCAL_FILE_NOT_READY`
- `DOCUMENT_QUOTA_EXHAUSTED`
- `DOCUMENT_GENERATION_FAILED`
- `DOCUMENT_VERSION_NOT_FOUND`
- `AI_TIMEOUT`
- `AI_RATE_LIMITED`
- `AI_OUTPUT_INVALID`
- `AI_MODEL_UNAVAILABLE`
- `AI_CAPACITY_FULL`


- `DOCUMENT_ASSIGNMENT_ACCESS_DENIED`
- `DOCUMENT_ASSIGNMENT_REVOKED`
- `DOCUMENT_SNAPSHOT_INVALID`
- `DOCUMENT_TEMPLATE_INVALID`
- `FILE_SCAN_PENDING`
- `FILE_QUARANTINED`

## 28. Rate Limiting

| Endpoint group | Kebijakan awal |
| --- | --- |
| Login/reset/MFA | Ketat per IP dan account hash |
| Upload intent | Per user/organization |
| AI generation | Per paket dan organisasi |
| Payment creation | Per request dan idempotency key |
| Document generation | Per Local File dan idempotency key |
| Report export | Per Administrator |
| Webhook | Provider-aware; jangan memblokir retry sah |

Nilai aktual disimpan sebagai konfigurasi dan ditentukan melalui pengujian.

## 29. Audit Requirements

Mutation berikut wajib mencatat audit:

- perubahan akun, MFA, sesi, role, dan status;
- organisasi, membership, invitation, dan document assignment;
- paket, kuota, payment, refund, dan withdrawal;
- Local File, upload, import, generate, download, dan delete;
- pengajuan Enterprise/Custom, quotation, pembayaran, dan aktivasi;
- perubahan status dokumen, warning, serta penerimaan hasil AI;
- tindakan Administrator.

Audit tidak menyimpan password, token, isi lengkap dokumen, NPWP lengkap, data finansial mentah, atau provider secret.

## 30. Contract Testing dan Acceptance

- Seluruh endpoint terlindungi menolak token invalid.
- Tenant A tidak dapat mengakses resource tenant B.
- Karyawan tanpa document assignment ditolak.
- Harga checkout tidak dapat dimanipulasi frontend.
- Webhook duplikat idempotent.
- AI/document generation retry tidak menggandakan job atau kuota.
- Penugasan dokumen yang dicabut langsung menutup akses Karyawan.
- Local File yang tidak lengkap gagal memasuki proses generate.
- Dokumen yang dihasilkan menggunakan snapshot dan template version yang sama.
- Admin tidak dapat menetapkan password pengguna.
- Response tidak mengandung secret atau stack trace.
- OpenAPI generated contract sesuai tabel endpoint ini.
- Integration test memverifikasi status code dan error code utama.

## 31. Referensi

- [System Architecture](./System-Architecture.md)
- [Technology Stack](./Technology-Stack.md)
- [AI Processing Architecture](./AI-Processing-Architecture.md)
- [Document Generation Architecture](./Document-Generation-Architecture.md)
- [ERD](../04-Database/ERD.dbml)
- [Functional Requirements](../01-Requirements/Functional-Requirements.md)

