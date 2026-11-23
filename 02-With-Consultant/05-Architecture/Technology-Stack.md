# Technology Stack dan Payment Gateway TP Docs
> Komponen diskon: modul NestJS `DiscountPricingService`, tabel PostgreSQL kampanye/target/pemakaian, cache hanya untuk tampilan yang boleh kedaluwarsa, dan transaksi atomik untuk checkout. Tidak ada SDK promo terpisah. Zona waktu event `Asia/Jakarta`; uang disimpan dalam minor unit. Harga reguler tetap tersedia ketika tidak ada promo.

## 1. Informasi Dokumen

| Atribut | Nilai |
| --- | --- |
| Produk | TP Docs — With Consultant |
| Jenis dokumen | Rancangan Technology Stack |
| Status | Draft — Pending Validation |
| Arsitektur awal | Modular Monolith dalam monorepo |
| Aplikasi web | Satu aplikasi Next.js untuk seluruh aktor |
| Backend | NestJS |
| Database yang direkomendasikan | PostgreSQL |
| Payment gateway yang direkomendasikan | Xendit |
| Strategi AI | Gemini API atau model lokal — belum diputuskan |

Dokumen ini menetapkan rancangan teknologi awal TP Docs. Keputusan yang masih berstatus **Pending Validation** wajib diuji melalui proof of concept, benchmark, pemeriksaan biaya, dan pemeriksaan ketentuan penyedia sebelum digunakan pada production.

---

## 2. Tujuan Arsitektur

Technology stack TP Docs dirancang untuk:

- Mendukung SaaS multi-tenant untuk User Induk, Karyawan, dan Administrator.
- Menjaga isolasi data antarorganisasi.
- Mendukung penyusunan Local File, analisis FAR, benchmarking, dan generate dokumen.
- Mendukung langganan, kuota dokumen, kursi pengguna, add-on, refund, dan pencairan dana perusahaan.
- Menghindari duplikasi aplikasi web selama produk masih berada pada tahap MVP.
- Memudahkan pemisahan layanan pada masa depan tanpa memulai dengan microservices yang terlalu kompleks.
- Menjaga kode mudah diuji melalui pemisahan domain, application service, dan infrastructure.
- Memungkinkan penggunaan AI melalui API eksternal ataupun model lokal dengan kontrak aplikasi yang sama.

### 2.1 Prinsip utama

1. **Backend adalah sumber otorisasi utama.** Penyembunyian menu atau pemeriksaan route pada frontend bukan pengamanan akhir.
2. **Satu akun untuk satu pengguna.** Password dan sesi tidak dibagikan.
3. **Tenant isolation diterapkan pada setiap akses data.** Semua query data perusahaan harus memiliki konteks organisasi yang sah.
4. **Modular monolith lebih dahulu.** Modul dapat dipisah menjadi service tersendiri apabila volume dan kebutuhan operasional telah membenarkannya.
5. **Integrasi eksternal menggunakan adapter.** Payment gateway, AI, email, dan malware scanner tidak boleh mengikat domain bisnis langsung ke satu vendor; file memakai `BinaryFileRepository` internal.
6. **Proses berat dijalankan asynchronous.** Generate dokumen, pemindaian file, email, dan pekerjaan AI tidak boleh menghambat request utama.
7. **Uang disimpan sebagai bilangan bulat.** Nominal Rupiah tidak menggunakan floating point.

---

## 3. Ringkasan Technology Stack

| Area | Teknologi yang direncanakan | Status/Alasan |
| --- | --- | --- |
| Monorepo | Nx + pnpm workspace | Direkomendasikan untuk mengelola aplikasi dan library bersama |
| Web | Next.js App Router + TypeScript | Satu aplikasi untuk publik, autentikasi, portal, dan Administrator |
| Styling | Tailwind CSS | Utility styling dan konsistensi responsive UI |
| UI primitives | shadcn/ui | Digunakan selektif dan tetap diklasifikasikan menurut Atomic Design |
| Component architecture | Atomic Design | `atoms`, `molecules`, `organisms`, dan `templates` di dalam `components/` |
| Backend API | NestJS + TypeScript | Modular monolith, dependency injection, guard, interceptor, dan validation |
| API style | REST JSON | Cocok untuk MVP dan integrasi frontend/backend yang eksplisit |
| ORM | Prisma | Schema, migration, query typed, dan transaction |
| Database | PostgreSQL | Rekomendasi utama untuk relasi, transaksi, JSONB, constraint, dan reporting |
| Cache/queue | Redis + BullMQ | Cache terbatas, rate-control, serta background processing |
| File storage | PostgreSQL `bytea` chunks | Metadata pada `files`; isi biner 1 MiB per chunk pada `file_blob_chunks`, maksimum 50 MB per file |
| Worker | NestJS/BullMQ worker | Generate dokumen, email Resend, file scan, AI job, dan webhook processing |
| Document generation | Template DOCX + PDF conversion service | Menghasilkan Word dan PDF secara asynchronous |
| AI API | Gemini API | Kandidat MVP; final provider Pending Evaluation |
| AI lokal | Python + FastAPI + model server | Kandidat self-hosted; model dan kebutuhan GPU Pending Benchmark |
| Payment gateway | Xendit | Kandidat utama pembayaran, refund, dan payout; fitur kontrak wajib diverifikasi |
| Email | Resend + React Email | Email transactional untuk verifikasi, undangan, reset password, billing, dan notifikasi |
| Container | Docker | Konsistensi development, staging, dan production |
| CI/CD | Pipeline lint, test, build, migration check, dan deploy | Vendor CI/CD belum ditetapkan |
| Observability | Structured log, metrics, tracing, dan error tracking | Vendor belum ditetapkan |

---

## 4. Struktur Monorepo

TP Docs menggunakan satu monorepo agar kontrak API, tipe data, konfigurasi, dan aturan kualitas dapat digunakan bersama tanpa menduplikasi implementasi.

```text
tp-docs/
├── apps/
│   ├── web/                         # Satu aplikasi Next.js
│   ├── api/                         # NestJS modular monolith
│   ├── worker/                      # BullMQ background worker
│   └── ai-service/                  # FastAPI hanya jika AI lokal dipilih
│
├── libs/
│   ├── contracts/                   # DTO/schema yang aman digunakan lintas aplikasi
│   ├── domain/                      # Value object dan domain primitive bersama
│   ├── config/                      # Validasi konfigurasi dan environment
│   ├── payment/                     # Interface payment provider dan tipe status
│   ├── binary-file/                 # BinaryFileRepository PostgreSQL
│   ├── notification/                # Interface email/notifikasi
│   ├── ai/                          # Kontrak request/output AI
│   ├── observability/               # Logging, metrics, dan correlation ID
│   └── testing/                     # Test fixture dan helper
│
├── tooling/                         # Konfigurasi lint, format, dan generator internal
├── docker/                          # Image dan konfigurasi container
├── nx.json
├── pnpm-workspace.yaml
└── package.json
```

### 4.1 Alasan `web` dan portal Administrator digabung

Portal pengguna dan portal Administrator berada pada satu aplikasi `apps/web` karena kondisi TP Docs saat ini masih berupa MVP dengan satu tim pengembang dan satu role Administrator. Pendekatan ini:

- Tidak menduplikasi konfigurasi Next.js.
- Tidak menduplikasi autentikasi, layout dasar, design token, dan komponen UI.
- Menyederhanakan build, pengujian, dan deployment.
- Memusatkan penerapan Atomic Design.
- Tetap memungkinkan layout dan route Administrator yang berbeda.
- Tetap dapat dipisah menjadi `admin-web` apabila kelak memiliki tim, domain, release cycle, atau kebutuhan deployment independen.

Penggabungan aplikasi web **bukan berarti aksesnya digabung**. Otorisasi tetap diperiksa pada frontend dan backend.

---

## 5. Frontend — Next.js

### 5.1 Struktur aplikasi web

```text
apps/web/src/
├── app/
│   ├── globals.css                  # Import Tailwind, reset, dan variabel tema shadcn/ui
│   ├── (public)/
│   │   ├── page.tsx
│   │   └── pricing/
│   │       └── page.tsx
│   │
│   ├── (auth)/
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── register/
│   │   │   └── page.tsx
│   │   └── forgot-password/
│   │       └── page.tsx
│   │
│   ├── (portal)/
│   │   ├── layout.tsx
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   ├── local-files/
│   │   │   └── page.tsx
│   │   ├── subscription/
│   │   │   └── page.tsx
│   │   ├── members/
│   │   │   └── page.tsx
│   │   └── settings/
│   │       └── page.tsx
│   │
│   └── admin/
│       ├── layout.tsx
│       ├── page.tsx
│       ├── users/
│       │   └── page.tsx
│       ├── organizations/
│       │   └── page.tsx
│       ├── subscriptions/
│       │   └── page.tsx
│       ├── payments/
│       │   └── page.tsx
│       ├── withdrawals/
│       │   └── page.tsx
│       ├── refunds/
│       │   └── page.tsx
│       ├── maintenance/
│       │   └── page.tsx
│       └── security/
│           └── page.tsx
│
├── components/
│   ├── atoms/
│   ├── molecules/
│   ├── organisms/
│   ├── templates/
│   └── index.ts
│
├── features/
│   ├── authentication/
│   ├── local-file/
│   ├── subscription/
│   ├── membership/
│   ├── payment/
│   └── administrator/
│
├── infrastructure/
│   ├── Container/
│   ├── Grid/
│   ├── Stack/
│   └── ErrorBoundary/
│
├── lib/
├── hooks/
├── providers/
└── middleware.ts
```

### 5.2 Pembagian tanggung jawab frontend

| Lokasi | Tanggung jawab |
| --- | --- |
| `app/` | Route, data loading, authorization boundary, metadata, dan integrasi halaman |
| `app/globals.css` | Import Tailwind, reset global minimum, dan variabel tema yang diperlukan shadcn/ui |
| `components/` | Komponen visual sesuai tingkat Atomic Design |
| `features/` | Logic UI per fitur, hook, mapper, action, dan state yang spesifik domain |
| `infrastructure/` | Primitive layout/teknis yang bukan elemen Atomic Design |
| `lib/` | Client API, formatter, helper umum, dan utilitas aman |
| `providers/` | Session, query client, theme, dan context tingkat aplikasi |

Business rule tidak ditempatkan di komponen visual. Validasi otoritatif tetap berada pada backend.

---

## 6. Penerapan Atomic Design

Hierarchy Atomic Design berada langsung di dalam `components/`:

```text
components/
├── atoms/
├── molecules/
├── organisms/
└── templates/
```

Design token merupakan fondasi **sub-atomic**, bukan komponen. Implementasinya menggunakan konfigurasi tema Tailwind dan variabel tema minimum di `app/globals.css`. Komponen tidak menggunakan CSS Modules atau file CSS manual per komponen.

### 6.1 Aturan dependency

Dependency hanya boleh bergerak satu arah:

```text
design tokens
    ↓
atoms
    ↓
molecules
    ↓
organisms
    ↓
templates
    ↓
pages
```

Komponen tingkat bawah tidak boleh mengimpor komponen dari tingkat yang lebih tinggi.

### 6.2 Atoms

Atoms merupakan elemen UI terkecil yang masih bermakna dan dapat digunakan ulang, misalnya:

- `Button`
- `Input`
- `Label`
- `Badge`
- `Icon`
- `Checkbox`
- `Radio`
- `Spinner`
- `Text`

Ketentuan atoms:

- Hanya bergantung pada token, utilitas styling, dan primitive yang benar-benar diperlukan.
- Bersifat presentational dan tidak mengambil data.
- Tidak mengetahui domain langganan, Local File, atau Administrator.
- Menyediakan state penting seperti default, hover, focus, disabled, loading, error, dan accessibility attributes.

### 6.3 Molecules

Molecules menggabungkan beberapa atoms untuk menyelesaikan satu micro-task, misalnya:

- `FormField`
- `PasswordField`
- `SearchInput`
- `FilePicker`
- `PaginationControl`
- `StatItem`
- `PriceDisplay`

Ketentuan molecules:

- Umumnya tersusun dari dua sampai empat atoms.
- Memiliki satu tujuan interaksi yang jelas.
- Tidak mengambil data secara langsung.
- Tidak memuat business rule atau global state.

### 6.4 Organisms

Organisms merupakan bagian halaman yang dapat dikenali, misalnya:

- `LoginForm`
- `LocalFileWizard`
- `SubscriptionPlanGrid`
- `MemberManagementTable`
- `AdminUserTable`
- `AdminPaymentPanel`
- `RefundRequestPanel`
- `WithdrawalPanel`

Ketentuan organisms:

- Menetapkan data contract melalui props.
- Menyusun atoms dan molecules menjadi section yang utuh.
- Tidak melakukan fetch data, routing, atau mutasi langsung.
- Tidak menjadi tempat business rule otoritatif.
- Organisms khusus Administrator diberi nama yang eksplisit agar tidak tertukar dengan portal pengguna.

### 6.5 Templates

Templates menentukan susunan layout dan slot halaman, misalnya:

- `AuthTemplate`
- `PortalDashboardTemplate`
- `LocalFileEditorTemplate`
- `AdminDashboardTemplate`
- `AdminDataManagementTemplate`

Ketentuan templates:

- Berfokus pada komposisi layout dan slot.
- Tidak menggunakan data riil atau melakukan fetch.
- Tidak memuat business rule.
- Sebaiknya menyusun organisms dan infrastructure components, bukan langsung menyusun atoms.

### 6.6 Pages

`page.tsx` merupakan titik integrasi untuk:

- Membaca route dan parameter.
- Memastikan session dan role.
- Mengambil atau mengirim data.
- Mengelola state halaman.
- Menghubungkan business flow dengan template/organism.
- Menampilkan loading, error, forbidden, dan empty state.

### 6.7 Penggunaan shadcn/ui

Komponen shadcn/ui tidak otomatis dikategorikan sebagai atom. Setiap komponen diklasifikasikan berdasarkan tanggung jawabnya. Contoh:

- Primitive `Button` dapat menjadi atom.
- Gabungan label, input, pesan error menjadi molecule.
- Dialog kompleks dengan form domain dapat menjadi organism.

Komponen hasil adopsi harus mengikuti token TP Docs, accessibility, naming, dan aturan dependency yang sama.

---

## 7. Keamanan Route Administrator

Route Administrator tetap menggunakan `/admin/**`. Nama route bukan kontrol keamanan dan tidak perlu disembunyikan dari pengguna.

Lapisan pengamanan yang wajib diterapkan:

1. `middleware.ts` melakukan pemeriksaan awal session dan role untuk pengalaman navigasi.
2. Layout Administrator memverifikasi session sebelum merender halaman.
3. Setiap endpoint NestJS menggunakan authentication guard dan permission guard.
4. Service backend memvalidasi kewenangan pada resource yang diminta.
5. Query organisasi selalu memvalidasi tenant context.
6. User Induk atau Karyawan yang membuka `/admin/**` memperoleh respons `403 Forbidden`.
7. Administrator wajib menggunakan MFA.
8. Tindakan berisiko tinggi meminta autentikasi ulang dan MFA challenge.
9. Aktivitas Administrator dicatat pada audit log yang tidak dapat diubah melalui portal.

Frontend tidak boleh menganggap route yang tersembunyi sebagai perlindungan. Request langsung ke API harus tetap ditolak apabila role atau permission tidak sah.

---

## 8. Backend — NestJS Modular Monolith

### 8.1 Struktur modul

```text
apps/api/src/
├── modules/
│   ├── identity/
│   ├── organization/
│   ├── membership/
│   ├── local-file/
│   ├── document-generation/
│   ├── benchmarking/
│   ├── ai-content/
│   ├── subscription/
│   ├── quota/
│   ├── billing/
│   ├── payment/
│   ├── refund/
│   ├── withdrawal/
│   ├── notification/
│   ├── file-management/
│   ├── audit/
│   ├── administrator/
│   └── health/
│
├── common/
│   ├── auth/
│   ├── guards/
│   ├── decorators/
│   ├── interceptors/
│   ├── filters/
│   ├── validation/
│   └── observability/
│
├── infrastructure/
│   ├── database/
│   ├── queue/
│   ├── binary-file/
│   ├── payment/
│   ├── email/
│   ├── malware-scanner/
│   └── ai/
│
└── main.ts
```

### 8.2 Struktur internal modul

Setiap modul domain menggunakan pemisahan tanggung jawab berikut:

```text
payment/
├── domain/
│   ├── entities/
│   ├── value-objects/
│   ├── events/
│   └── repositories/
├── application/
│   ├── commands/
│   ├── queries/
│   ├── services/
│   └── dto/
├── infrastructure/
│   ├── persistence/
│   └── providers/
└── presentation/
    ├── controllers/
    └── presenters/
```

Pemisahan ini dapat disederhanakan untuk modul kecil. Tujuannya adalah menjaga batas tanggung jawab, bukan menambah folder tanpa kebutuhan.

### 8.3 Penerapan OOP dan SOLID

OOP diterapkan untuk memperjelas domain dan dependency, bukan untuk membuat seluruh fungsi menjadi class.

- **Encapsulation:** perubahan status subscription, refund, atau withdrawal dilakukan melalui method yang menjaga invariant.
- **Single Responsibility:** controller menangani HTTP, application service mengatur use case, repository mengurus persistence, adapter mengurus vendor.
- **Open/Closed:** provider AI dan payment dapat diganti melalui interface tanpa mengubah domain.
- **Liskov Substitution:** adapter vendor harus mengikuti kontrak dan error model yang sama.
- **Interface Segregation:** pisahkan kontrak payment, refund, balance, dan payout apabila tanggung jawabnya berbeda.
- **Dependency Inversion:** application layer bergantung pada interface, bukan SDK vendor.

Contoh kontrak provider:

```ts
export interface PaymentProvider {
  createPayment(input: CreatePaymentInput): Promise<PaymentResult>;
  getPayment(providerPaymentId: string): Promise<PaymentResult>;
  createRefund(input: CreateRefundInput): Promise<RefundResult>;
  getRefund(providerRefundId: string): Promise<RefundResult>;
  getAvailableBalance(): Promise<Money>;
  createWithdrawal(input: CreateWithdrawalInput): Promise<WithdrawalResult>;
  getWithdrawal(providerWithdrawalId: string): Promise<WithdrawalResult>;
  verifyWebhook(input: VerifyWebhookInput): Promise<VerifiedWebhook>;
}
```

---

## 9. API dan Kontrak Data

### 9.1 REST API

MVP menggunakan REST JSON dengan aturan:

- Seluruh endpoint menggunakan prefix `/api`, misalnya `/api/auth/login`, `/api/local-files`, dan `/api/admin/users`.
- Versioning URL belum digunakan pada MVP. Perubahan kontrak yang merusak kompatibilitas memerlukan keputusan arsitektur tersendiri.
- Request dan response divalidasi dengan schema.
- OpenAPI digunakan sebagai dokumentasi kontrak.
- Error memiliki kode stabil, pesan aman, dan correlation ID.
- Pagination menggunakan aturan konsisten.
- Tanggal dikirim dalam ISO 8601.
- Nominal uang dikirim sebagai integer minor unit atau string integer sesuai kontrak.
- Endpoint mutasi finansial mendukung idempotency key.

### 9.2 Status uang dan transaksi

Status internal tidak boleh bergantung langsung pada istilah vendor. Adapter bertugas memetakan status vendor ke status TP Docs.

Contoh:

```text
Payment:    PENDING | PAID | FAILED | EXPIRED | CANCELLED
Refund:     SUBMITTED | UNDER_REVIEW | APPROVED | REJECTED |
            PROCESSING | REFUNDED | FAILED
Withdrawal: DRAFT | SUBMITTED | PROCESSING | SUCCEEDED |
            FAILED | CANCELLED
```

---

## 10. Database dan Persistence

### 10.1 Perbandingan PostgreSQL dan MySQL

| Aspek | PostgreSQL | MySQL |
| --- | --- | --- |
| Transaksi dan constraint | Sangat kuat | Memadai untuk kebutuhan umum |
| Query reporting | Kaya dan fleksibel | Baik untuk query umum |
| JSON | JSONB dan indexing kuat | JSON tersedia |
| Row-level security | Tersedia sebagai lapisan tambahan | Tidak setara secara native |
| Ekosistem Prisma | Didukung | Didukung |
| Kesesuaian TP Docs | Direkomendasikan | Alternatif jika tim lebih berpengalaman |

### 10.2 Rekomendasi

Gunakan **PostgreSQL** karena data TP Docs sangat relasional, memerlukan transaksi konsisten, constraint, audit, reporting, dan fleksibilitas JSONB untuk sebagian data dokumen yang terstruktur dinamis.

Keputusan final tetap mempertimbangkan kemampuan tim dan layanan database terkelola yang tersedia.

### 10.3 Aturan data utama

- Setiap record milik pelanggan memiliki `organization_id` apabila relevan.
- Repository wajib menerima tenant context; tidak menerima organisasi dari input bebas tanpa validasi membership.
- Foreign key dan unique constraint digunakan untuk menjaga integritas.
- Optimistic locking atau version field digunakan untuk data yang dapat diedit bersamaan.
- Nominal uang disimpan sebagai `BIGINT`, bukan `FLOAT` atau `DOUBLE`.
- Mata uang disimpan menggunakan kode ISO, misalnya `IDR`.
- Waktu disimpan dalam UTC dan ditampilkan sesuai zona pengguna.
- Audit log bersifat append-only dari sisi aplikasi.
- Data sensitif yang tidak diperlukan untuk pencarian dapat dienkripsi pada application layer.
- Migration dijalankan melalui pipeline terkontrol dan diuji pada staging.

---

## 11. Redis dan Background Worker

Redis digunakan untuk:

- BullMQ queue.
- Rate-control dan lock berjangka pendek.
- Cache data non-rahasia yang aman untuk di-cache.
- Idempotency record berjangka pendek, dengan catatan transaksi final tetap disimpan pada database.

Background worker menangani:

- Generate DOCX dan PDF.
- Pengiriman email transactional melalui Resend.
- Pemindaian malware.
- Proses impor Excel/TSV.
- AI content generation.
- Pemrosesan webhook lanjutan.
- Rekonsiliasi status pembayaran, refund, dan withdrawal.
- Penghapusan terjadwal sesuai retensi.

Setiap job wajib memiliki:

- Identitas organisasi apabila job berhubungan dengan tenant.
- Idempotency key.
- Batas retry.
- Exponential backoff.
- Dead-letter handling.
- Correlation ID.
- Log tanpa isi dokumen atau data finansial mentah.

---

## 12. File Storage dan Document Generation

### 12.1 PostgreSQL binary file storage

Semua file permanen—termasuk file review dan delivery Konsultan—disimpan di PostgreSQL melalui interface `BinaryFileRepository`. Tabel `files` menyimpan metadata, checksum, jumlah chunk, status scan, dan retensi; tabel `file_blob_chunks` menyimpan konten `bytea` berurutan. Browser tidak pernah mengakses database secara langsung.

Ketentuan teknis:

- Ukuran chunk adalah 1 MiB dan ukuran maksimum file 50 MB.
- Upload menggunakan `multipart/form-data`; download di-stream oleh NestJS dan mendukung HTTP Range.
- Metadata tidak di-query bersama isi biner secara default.
- Nama file asli disimpan sebagai metadata yang telah disanitasi.
- Upload divalidasi berdasarkan ukuran, MIME type, ekstensi, dan signature file.
- File dipindai malware sebelum diproses.
- File berbahaya dikarantina.
- Enkripsi PostgreSQL at rest dan TLS koneksi diaktifkan.
- Backup, PITR, restore, retensi, dan penghapusan mencakup seluruh chunk serta diverifikasi dengan checksum.
- Upload/download memiliki rate limit dan concurrency limit untuk melindungi connection pool dan memori.

Konfigurasi runtime:

```text
FILE_MAX_SIZE_BYTES=52428800
FILE_CHUNK_SIZE_BYTES=1048576
FILE_UPLOAD_CONCURRENCY_LIMIT=<hasil load test>
FILE_DOWNLOAD_CONCURRENCY_LIMIT=<hasil load test>
FILE_TEMP_DIRECTORY=<isolated-worker-directory>
```

Tidak dilakukan deduplikasi lintas organisasi. Temporary workspace hanya digunakan selama scan, ekstraksi, render, review, atau konversi dan selalu dihapus setelah job berhasil maupun gagal.

### 12.2 Generate dokumen

Document generation menggunakan pendekatan **template-first**. Satu master DOCX resmi TP Docs menjadi sumber kebenaran cover, daftar isi, header/footer, margin, font, heading, spacing, tabel, warna, dan section break. Sistem tidak menyusun desain Word dari nol.

Data Local File dibekukan sebagai snapshot, dipetakan ke `LocalFileDocumentViewModel`, lalu di-bind ke template. DOCX merupakan hasil utama; PDF opsional selalu dikonversi dari DOCX tervalidasi yang sama.

Komponen teknis document generation:

| Komponen | Tanggung jawab |
| --- | --- |
| `DocumentGenerationWorker` | Menjalankan pekerjaan generate di luar request utama |
| Template master DOCX | Menetapkan seluruh layout, styling, placeholder, loop, dan conditional section |
| Template manifest | Mendefinisikan placeholder wajib/opsional, repeating section, conditional section, schema, dan font |
| Local File snapshot | Membekukan data sumber untuk satu document version |
| View model mapper | Mengubah snapshot menjadi kontrak konten terformat tanpa mengekspos database schema ke template |
| Docxtemplater renderer | Mengisi placeholder, condition, dan row loop sambil mempertahankan styling template |
| DOCX validator | Memeriksa OpenXML, placeholder tersisa, macro, relationship, dan struktur hasil |
| LibreOffice headless converter | Mengubah DOCX tervalidasi menjadi PDF di process/container terisolasi |
| PDF validator | Memeriksa hasil, page count, checksum, font, dan keterkaitan dengan DOCX version |
| `BinaryFileRepository` | Membaca dan menulis template serta hasil dokumen sebagai chunk PostgreSQL |
| Job status | Menyimpan tahap render, validation, conversion, success, atau failure |
| Versioning | Menjaga versi hasil dokumen tanpa menimpa histori sebelumnya |
| Idempotency | Mencegah retry teknis membuat hasil atau penggunaan kuota ganda |

Template dikelola melalui rilis internal berversi dan tidak dapat diunggah oleh User Induk atau Administrator pada MVP. Worker menerapkan timeout, retry terbatas, exponential backoff, correlation ID, temporary workspace terisolasi, serta pencatatan error yang tidak memuat isi dokumen.

Detail kontrak dan visual tersedia pada [Document Generation Architecture](./Document-Generation-Architecture.md).

---

## 13. AI Content Generation

AI merupakan fitur opsional untuk membentuk draft fase skema proses bisnis dari uraian wajib dan daftar langkah opsional milik pengguna. AI bukan pemberi opini pajak, bukan pengganti komponen Local File lainnya, dan hasilnya wajib ditinjau pengguna jika fitur digunakan. Kegagalan AI tidak menghalangi penyimpanan uraian atau penyelesaian Local File.

### 13.1 Kontrak output bersama

Baik Gemini API maupun AI lokal menerima kontrak input dan menghasilkan struktur yang sama:

```ts
type ProcessStepInput = {
  id: string;
  sequence: number;
  activity: string;
  actorOrDepartment?: string;
  description?: string;
};

type BusinessProcessGenerationInput = {
  description: string;
  processSteps?: ProcessStepInput[];
};

type BusinessProcessPhase = {
  phaseNumber: number;
  phaseName: string;
  activityTitle: string;
  shortDescription: string;
  category: 'INPUT' | 'PROCESS' | 'CONTROL' | 'OUTPUT';
  nextPhaseNumber: number | null;
  sourceStepId: string | null;
};
```

Output divalidasi terhadap schema. Jika `processSteps` tersedia, setiap langkah harus menghasilkan tepat satu fase dengan `sourceStepId` yang sesuai dan urutan yang sama. Respons bebas yang tidak sesuai schema tidak disimpan ke Local File. Jika AI tidak digunakan, uraian bisnis tetap menjadi bagian Local File tanpa diagram fase.

### 13.2 Opsi A — Gemini API

Gemini API menjadi kandidat untuk MVP karena implementasi lebih cepat dan tidak membutuhkan pengelolaan GPU.

Komponen:

- `AiGenerationService` pada application layer.
- `GeminiAiProvider` pada infrastructure layer.
- Structured prompt dan structured output/schema validation.
- Secret API disimpan pada secret manager.
- Batas token, timeout, retry terbatas, dan usage monitoring.
- Log biaya hanya menyimpan metadata penggunaan, bukan uraian lengkap pelanggan.

Hal yang wajib divalidasi sebelum production:

- Region pemrosesan dan kebijakan penggunaan data.
- Retensi prompt dan respons oleh provider.
- Tarif token aktual.
- Rate limit dan quota.
- Kualitas Bahasa Indonesia dan konsistensi structured output.

### 13.3 Opsi B — AI lokal

Jika AI lokal dipilih, arsitektur menggunakan:

- Python.
- FastAPI sebagai internal inference API.
- Model server terpisah dari aplikasi utama.
- Queue dan concurrency limit.
- GPU server sesuai hasil benchmark.
- Health check dan timeout.
- Model artifact storage dan versioning.

Kandidat awal untuk benchmark:

| Kandidat | Posisi evaluasi |
| --- | --- |
| Qwen2.5-7B-Instruct | Kandidat utama awal untuk structured instruction dan multilingual |
| Gemma 3 4B IT | Kandidat lebih ringan untuk menguji kebutuhan resource |
| Qwen3-8B | Kandidat tambahan apabila lisensi, kualitas, dan resource sesuai |

Nama model final, lisensi, VRAM, quantization, concurrency, dan target response time berstatus **Pending Benchmark**.

### 13.4 Batasan keamanan AI

- Prompt tidak boleh memasukkan data yang tidak diperlukan.
- Data satu tenant tidak digunakan untuk konteks tenant lain.
- Data pelanggan tidak digunakan untuk melatih ulang model secara otomatis.
- Prompt dan respons mentah tidak dicatat pada application log.
- Kegagalan AI tidak menghapus uraian asli.
- Tidak ada fallback otomatis dari AI lokal ke API atau sebaliknya tanpa keputusan arsitektur yang eksplisit.
- Pengguna wajib menerima atau generate ulang hasil sebelum hasil digunakan oleh Local File.

---

## 14. Payment Gateway

### 14.1 Kebutuhan bisnis

Payment gateway harus mendukung:

- Pembayaran pelanggan Indonesia.
- Pembayaran kartu pelanggan luar negeri apabila akun merchant dan kontrak mendukung.
- Status pembayaran melalui webhook.
- Refund ke metode pembayaran asal apabila channel mendukung.
- Pemeriksaan saldo merchant.
- Payout atau pencairan dana ke rekening perusahaan yang telah diverifikasi.
- Rekonsiliasi transaksi.
- Idempotency dan signature verification.
- Lingkungan test/sandbox.

### 14.2 Perbandingan awal

| Provider | Kekuatan untuk TP Docs | Catatan |
| --- | --- | --- |
| Xendit | Coverage pembayaran Indonesia, API, webhook, refund, balance, dan payout | Direkomendasikan; ketersediaan setiap fitur mengikuti aktivasi akun dan kontrak |
| Midtrans | Kuat untuk pembayaran domestik dan integrasi umum | Kemampuan payout/refund lintas channel perlu divalidasi terhadap kebutuhan Admin portal |
| DOKU | Alternatif domestik dengan produk pembayaran dan disbursement | Proses approval dan cakupan internasional perlu dievaluasi |
| Stripe | Baik untuk pembayaran global dan developer experience | Ketersediaan akun merchant, payout, dan kebutuhan operasi Indonesia harus diverifikasi |

### 14.3 Rekomendasi implementasi

Gunakan **Xendit Payment Session** dengan `session_type: PAY`, `mode: PAYMENT_LINK`, dan `capture_method: AUTOMATIC` sebagai hosted checkout MVP. Backend membuat session dan frontend hanya membuka `payment_link_url`. Return URL tidak menjadi bukti pembayaran; aktivasi selalu menunggu verified webhook atau rekonsiliasi server-to-server.

Perpanjangan MVP menggunakan checkout baru. `allow_save_payment_method` dan auto-renew tidak diaktifkan sampai consent, recurring channel, retry, penghentian, serta kebijakan gagal bayar selesai divalidasi. `Organization` dapat dipetakan ke customer Xendit `BUSINESS`, tetapi bukan XenPlatform sub-account karena pelanggan TP Docs bukan merchant penerima dana.

Pembayaran pelanggan luar negeri hanya ditampilkan apabila country, channel, currency, settlement, dan refund capability telah aktif pada akun merchant. Database menyimpan currency ISO-4217 dan nominal minor unit; TP Docs tidak melakukan konversi otomatis saat checkout.

Seluruh kemampuan aktual, biaya, reserve, settlement, mata uang, channel refund, payout, dan proses approval harus dikonfirmasi langsung saat onboarding merchant. Dokumentasi ini tidak dianggap sebagai jaminan bahwa seluruh fitur otomatis aktif.

### 14.4 Adapter, webhook, dan idempotency

Integrasi payment gateway menggunakan `PaymentProvider` agar application layer tidak bergantung langsung pada SDK Xendit. Adapter bertanggung jawab memetakan request, response, status, serta error provider ke kontrak internal TP Docs.

Ketentuan teknis:

- Harga dan nominal pembayaran dihitung dari katalog internal backend, bukan dipercaya dari frontend.
- Webhook diverifikasi menggunakan signature atau verification token provider.
- Event ID, reference ID, nominal, mata uang, dan tipe event divalidasi sebelum status internal diperbarui.
- Event webhook disimpan secara idempotent agar pengiriman ulang tidak menghasilkan transaksi ganda.
- Payment Session, Payment Request, Payment, Refund, Dispute, dan Payout dipetakan ke resource internal yang berbeda.
- Event dispute yang berulang dideduplicasi menggunakan dispute ID, status, status reason, dan provider updated time.
- Redirect frontend tidak dianggap sebagai bukti pembayaran.
- Rekonsiliasi terjadwal tersedia untuk transaksi yang status webhook-nya tidak lengkap.
- Log hanya menyimpan metadata transaksi yang diperlukan dan tidak menyimpan credential atau data pembayaran mentah.

### 14.5 Konfigurasi refund dan withdrawal

Refund dan withdrawal tetap menjadi kemampuan teknis payment module, sementara urutan aktivitas bisnisnya didokumentasikan pada folder `03-Business-Flow`. Refund mengembalikan dana terhadap payment asal; payout adalah transfer keluar dan tidak digunakan sebagai pengganti refund.

Kontrol teknis untuk satu Administrator:

- MFA dan autentikasi ulang wajib untuk tindakan finansial berisiko tinggi.
- Alasan tindakan wajib dicatat.
- Refund dan withdrawal menggunakan idempotency key.
- Limit nominal, frekuensi, serta cooldown dapat dikonfigurasi.
- Rekening tujuan pencairan perusahaan disimpan sebagai konfigurasi terverifikasi dan bersifat read-only pada portal.
- Status provider dipetakan ke status internal yang konsisten.
- Notifikasi hasil dikirim melalui Resend ke email perusahaan.
- Audit log bersifat append-only.
- Fitur dinonaktifkan secara aman apabila konfigurasi wajib belum lengkap.
- Payout/withdrawal dinonaktifkan pada MVP sampai rekening perusahaan allowlist, permission API, limit, dan dukungan settlement tervalidasi.

Konfigurasi disimpan pada environment configuration atau secret manager, bukan ditulis permanen dalam source code:

```text
XENDIT_SECRET_API_KEY=
XENDIT_WEBHOOK_VERIFICATION_TOKEN=
XENDIT_SUCCESS_RETURN_URL=
XENDIT_CANCEL_RETURN_URL=
XENDIT_PAYOUT_ENABLED=false
WITHDRAWAL_MIN_IDR=
WITHDRAWAL_DAILY_LIMIT_IDR=
WITHDRAWAL_COOLDOWN_MINUTES=
WITHDRAWAL_DESTINATION_ID=
FINANCE_NOTIFICATION_EMAIL=
REFUND_ADMIN_LIMIT_IDR=
```

---

## 15. Autentikasi dan Otorisasi

### 15.1 Authentication

- Password di-hash menggunakan Argon2id atau algoritma adaptif setara.
- User Induk dan Administrator wajib MFA.
- Karyawan dapat menggunakan MFA opsional pada versi awal.
- Reset password menggunakan token sekali pakai, berumur pendek, dan disimpan dalam bentuk hash.
- Session idle berakhir setelah 30 menit sesuai NFR.
- Refresh token menggunakan rotation dan reuse detection.
- Tindakan sensitif memerlukan recent authentication.

### 15.2 Authorization

Gunakan gabungan RBAC dan resource-based authorization:

- Role menentukan kategori kewenangan umum.
- Membership menentukan organisasi yang dapat diakses.
- Document assignment menentukan dokumen Karyawan yang dapat diakses.
- Permission menentukan tindakan Administrator.
- Tenant validation memastikan resource berada pada organisasi yang benar.

Role awal:

```text
ORGANIZATION_OWNER
EMPLOYEE
ADMINISTRATOR
```

Administrator tidak otomatis dapat membaca isi Local File. Akses data harus mengikuti use case dan batas kewenangan yang telah ditetapkan.

---

## 16. Secret, Konfigurasi, dan Email Transactional

Secret berikut wajib disimpan pada secret manager:

- Database credential.
- Redis credential.
- JWT/session signing key.
- Xendit secret dan webhook verification token.
- Gemini API key.
- `RESEND_API_KEY`, alamat pengirim, dan alamat reply-to.
- Malware scanner credential.

Aturan:

- File `.env` production tidak disimpan pada repository.
- Konfigurasi divalidasi saat aplikasi mulai.
- Aplikasi gagal start secara aman apabila secret wajib tidak tersedia.
- Secret tidak ditampilkan pada log atau portal Administrator.
- Secret memiliki mekanisme rotasi.
- Konfigurasi development, staging, dan production dipisahkan.

### 16.1 Resend dan React Email

Resend digunakan khusus untuk email transactional, bukan sebagai mailbox untuk menerima email pelanggan. Domain layer bergantung pada interface `EmailProvider`, sedangkan SDK Resend hanya digunakan pada infrastructure adapter.

- React Email dapat digunakan untuk membuat template yang konsisten dan dapat diuji.
- Email mencakup verifikasi akun, undangan anggota, reset password, pembayaran, refund, serta notifikasi Administrator.
- Pengiriman dilakukan melalui BullMQ worker dengan retry terbatas, exponential backoff, dan idempotency key.
- Domain pengirim wajib dikonfigurasi dengan SPF, DKIM, dan DMARC.
- Sender dan reply-to dibaca dari konfigurasi lingkungan.
- Log tidak menyimpan token reset, isi rahasia, atau data pelanggan lengkap.
- Kegagalan permanen dan status delivery webhook dicatat untuk pemantauan tanpa menggandakan email.

---

## 17. Deployment

### 17.1 Lingkungan

```text
Local Development → Staging → Production
```

Setiap lingkungan menggunakan database, Redis, payment credential, Resend credential, dan AI configuration yang terpisah.

### 17.2 Unit deployment

| Unit | Fungsi |
| --- | --- |
| `web` | Next.js untuk publik, portal perusahaan, dan Administrator |
| `api` | NestJS API modular monolith |
| `worker` | Queue worker untuk pekerjaan asynchronous |
| `ai-service` | Hanya dideploy jika AI lokal dipilih |
| PostgreSQL | Database terkelola |
| Redis | Cache/queue terkelola |
| PostgreSQL binary tables | File privat, template, hasil dokumen, review, dan delivery Konsultan |

### 17.3 Container dan cloud

- Aplikasi dikemas menggunakan Docker.
- Production direkomendasikan menggunakan managed PostgreSQL dan managed Redis.
- Web, API, dan worker dapat di-scale secara terpisah.
- AI lokal harus berada pada node GPU terpisah agar tidak memengaruhi API utama.
- Migration database dijalankan sebagai langkah deployment terkontrol.
- Deployment menggunakan health check dan rollback aplikasi.
- Backup, retention, dan disaster recovery mengikuti Non-Functional Requirements.

### 17.4 Kapan portal Administrator perlu dipisahkan

`admin-web` baru dipertimbangkan apabila terdapat salah satu kondisi berikut:

- Tim Administrator frontend berbeda.
- Domain dan kebijakan jaringan harus benar-benar terpisah.
- Release cycle Administrator berbeda dari portal pelanggan.
- Deployment Administrator membutuhkan availability atau maintenance window berbeda.
- Ukuran bundle dan kebutuhan operasional telah terbukti menjadi masalah.

Pemisahan frontend tidak mengubah kewajiban backend untuk memvalidasi role dan permission.

---

## 18. Observability dan Audit

### 18.1 Application observability

- Structured JSON log.
- Correlation ID dari web hingga worker.
- Metrics untuk request, latency, error, queue, payment, AI, dan document generation.
- Distributed tracing apabila kebutuhan operasional membenarkan.
- Error tracking tanpa mengirim isi dokumen rahasia.
- Alert untuk kegagalan aplikasi, integrasi, dan backlog job.

### 18.2 Audit log

Audit log mencatat:

- Identitas pelaku.
- Role dan organisasi terkait.
- Waktu.
- Jenis tindakan.
- Resource identifier.
- Alasan untuk tindakan berisiko tinggi.
- Nilai sebelum/sesudah yang telah disanitasi bila diperlukan.
- Correlation ID.
- Hasil tindakan.

Audit log tidak menyimpan password, token, API key, isi lengkap Local File, NPWP lengkap, atau data finansial mentah.

---

## 19. Strategi Pengujian

| Jenis pengujian | Cakupan |
| --- | --- |
| Unit test | Domain rule, value object, mapper, dan application service |
| Component test | Atoms, molecules, organisms, state, dan accessibility |
| Contract test | Kontrak frontend–API dan adapter provider |
| Integration test | Database termasuk chunk file, Redis, BullMQ, Resend sandbox/test mode, AI, dan payment sandbox |
| End-to-end test | Registrasi, Local File, langganan, anggota, generate, refund, dan Admin |
| Tenant isolation test | Akses lintas organisasi melalui UI, API, file URL, dan background job |
| Security test | Authentication, authorization, rate-control, upload, dan secret leakage |
| Visual regression | Komponen penting dan halaman portal |
| Load test | Login, dashboard, import transaksi, generate dokumen, dan webhook |
| Disaster recovery test | Restore backup sesuai target RPO/RTO |

### 19.1 Pengujian payment dan withdrawal

- Webhook dengan signature salah ditolak.
- Event webhook yang sama hanya diproses sekali.
- Nominal dari frontend tidak dapat mengganti harga server.
- Pembayaran gagal tidak mengaktifkan paket.
- Refund retry tidak membuat pengembalian ganda.
- Refund berhasil menyesuaikan akses tepat satu kali.
- Pencairan gagal tidak dicatat sebagai sukses.
- Pencairan ditolak jika saldo atau limit tidak memenuhi.
- Rekening tujuan tidak dapat diganti melalui request browser.
- User Induk dan Karyawan tidak dapat mengakses endpoint withdrawal.
- Administrator harus menyelesaikan re-authentication dan MFA.

### 19.2 Pengujian Atomic Design

- Dependency lint mencegah atoms mengimpor molecules atau tingkat di atasnya.
- Organisms tidak mengambil data langsung.
- Templates tidak memuat business rule atau data riil.
- Komponen interaktif dapat digunakan dengan keyboard.
- State focus, error, loading, dan disabled tersedia.
- Storybook atau katalog komponen digunakan untuk komponen reusable utama.

---

## 20. Kriteria Penerimaan Arsitektur

- Hanya terdapat satu aplikasi Next.js pada `apps/web`.
- Portal Administrator tersedia pada `/admin/**` dengan layout terpisah.
- Akses `/admin/**` dan endpoint Admin ditolak untuk role non-Administrator.
- Atomic Design berada di dalam `components/` dan mengikuti dependency satu arah.
- Styling komponen menggunakan utility Tailwind; hanya `app/globals.css` yang digunakan untuk import Tailwind, reset global minimum, dan variabel tema shadcn/ui.
- NestJS berjalan sebagai modular monolith dengan batas modul yang jelas.
- Tenant context divalidasi pada setiap akses data organisasi.
- PostgreSQL menyimpan transaksi dengan constraint dan migration yang terkontrol.
- Pekerjaan berat berjalan melalui worker dan tidak memblokir request utama.
- Provider AI dapat diganti melalui adapter tanpa mengubah business flow.
- AI bersifat opsional; jika digunakan, output divalidasi, dapat ditelusuri ke langkah input, dan wajib ditinjau pengguna sebelum masuk ke Local File.
- Payment, refund, dan withdrawal menggunakan idempotency serta webhook verification.
- Tindakan finansial satu Administrator dilindungi re-authentication, MFA, limit, cooldown, notifikasi, dan audit log.
- Secret tidak berada di source code atau log.
- Email transactional menggunakan adapter Resend, dikirim melalui worker, dan domain pengirim memiliki SPF, DKIM, serta DMARC.
- Pengujian tenant isolation, authorization, dan transaksi finansial lulus sebelum production.

---

## 21. Keputusan Pending Validation

| Keputusan | Kandidat/Rekomendasi | Validasi yang diperlukan |
| --- | --- | --- |
| Database final | PostgreSQL | Kemampuan tim, biaya managed service, dan proof of concept |
| AI final | Gemini API atau AI lokal | Kualitas, privasi, latency, concurrency, dan total biaya |
| Model AI lokal | Qwen2.5-7B-Instruct sebagai benchmark awal | Lisensi, Bahasa Indonesia, structured output, VRAM, dan throughput |
| Payment gateway final | Xendit | Approval merchant, channel, biaya, refund, payout, settlement, dan mata uang |
| Mata uang pelanggan asing | IDR untuk MVP | Dukungan kartu internasional dan kebijakan provider |
| Kebijakan refund | Belum final | Batas waktu, kelayakan, nilai, channel, dan dampak akses |
| Limit pencairan | Belum final | Minimum, maksimum harian, cooldown, dan rekening tujuan |
| Cloud provider | Belum dipilih | Region, layanan terkelola, biaya, keamanan, dan dukungan |
| Conversion DOCX–PDF | Belum dipilih | Fidelity, concurrency, isolasi proses, dan lisensi |
| Observability vendor | Belum dipilih | Data residency, masking, integrasi, dan biaya |

Keputusan final harus dicatat melalui Architecture Decision Record agar alasan, alternatif, konsekuensi, dan tanggal keputusan dapat ditelusuri.

---

## 22. Technology Stack Layanan Konsultan

### 22.1 Komponen

| Area | Teknologi / Pendekatan |
| --- | --- |
| Portal Konsultan | Next.js App Router pada aplikasi web yang sama |
| UI | Tailwind CSS, shadcn/ui, React Hook Form, Zod |
| Viewer | HTML semantik read-only dan DOM Range API |
| State server | TanStack Query atau mekanisme setara |
| API domain | Modul NestJS Consultation, Assignment, Review, SLA, Workspace |
| Validation | class-validator/Zod boundary sesuai standar proyek |
| Persistence | PostgreSQL melalui ORM/query builder terpilih |
| Concurrency | Optimistic version/ETag dan transaction |
| Queue | BullMQ untuk preview, delivery, notification, cleanup |
| Cache/lock | Redis dengan key tenant/request scoped |
| File persistence | PostgreSQL `files` + `file_blob_chunks` |
| Email | Resend melalui `EmailProvider` |
| Payment | Xendit melalui `PaymentProvider` |
| Observability | Structured log, metrics, tracing, audit |
| Testing | Unit, integration, contract, E2E, tenant-isolation |

### 22.2 Struktur Frontend

```text
apps/web/src/
├── app/
│   ├── (consultant)/
│   │   ├── consultant/
│   │   │   ├── dashboard/
│   │   │   ├── assignments/
│   │   │   ├── reviews/
│   │   │   ├── workspaces/
│   │   │   └── settings/
│   └── (portal)/
│       └── consultations/
├── features/
│   ├── consultation-catalog/
│   ├── consultation-request/
│   ├── consultant-assignment/
│   ├── document-review/
│   ├── review-thread/
│   └── consultation-workspace/
└── components/
    ├── atoms/
    ├── molecules/
    ├── organisms/
    └── templates/
```

Portal Konsultan dapat memakai route yang tidak mudah ditebak, tetapi route bukan pengaman. Backend authorization tetap wajib.

### 22.3 Struktur Backend

```text
apps/api/src/modules/
├── consultant-management/
├── consultation-catalog/
├── consultation-request/
├── consultation-assignment/
├── consultation-sla/
├── document-review/
├── consultation-workspace/
├── payment/
├── notification/
└── audit/
```

Setiap modul mengikuti struktur domain/application/infrastructure/interfaces. SDK provider hanya berada pada infrastructure adapter.

### 22.4 Viewer dan Highlight

- Semantic preview dirender dari view model document version.
- DOM Range hanya digunakan menangkap selection di browser.
- Backend membentuk dan memvalidasi anchor final.
- HTML disanitasi dan menggunakan CSP.
- Nomor halaman tidak digunakan sebagai anchor.
- Viewer melakukan lazy load per section dan pagination thread.
- Comment update menggunakan optimistic concurrency.

### 22.5 Working Copy

Working copy menggunakan structured JSON/data model dengan schema version. Change set bersifat append-oriented dan menyimpan path, operation, before/after hash, actor, serta waktu. Promotion dilakukan melalui transaction backend, bukan merge di browser.

### 22.6 Background Jobs

| Queue | Job |
| --- | --- |
| `consultation-notification` | Assignment, ETA, delivery, resubmission |
| `consultation-preview` | Semantic preview dan draft preview |
| `consultation-delivery` | Build summary/checksum/delivery asset |
| `consultation-access-cleanup` | Revoke, refund, completion |
| `consultation-sla` | Reminder dan breach detection |
| `consultation-promotion` | Promotion working copy idempotent |

### 22.7 Database dan Index

Index minimum:

- request: organization/status/created;
- request: requester/created;
- assignment: consultant/status/due;
- assignment: request/status;
- preview: version/section/order;
- comment: request/round/status;
- replies: comment/created;
- SLA event: assignment/created;
- workspace change set: workspace/created.

Gunakan eager loading/read model terkontrol untuk mencegah N+1 pada dashboard, viewer, thread, dan timeline.

### 22.8 Security

- MFA wajib bagi Konsultan.
- Scoped assignment guard pada viewer dan workspace.
- Payment field tidak dimasukkan DTO Konsultan.
- Streaming file melalui NestJS dengan validasi tenant dan assignment.
- Rate limit komentar/reply/delivery.
- Output encoding dan sanitizer.
- Revoke session ketika suspend bila diperlukan.
- Audit untuk view sensitif dan mutation.
- Secret provider tetap di secret manager.

### 22.9 Pengujian Tambahan

- Unit test state transition.
- Integration test price snapshot dan payment.
- Contract test seluruh endpoint konsultasi.
- E2E empat aktor.
- Tenant isolation test.
- Assignment/reassignment access test.
- Anchor property test.
- Concurrency test comment/workspace.
- Idempotency test webhook/delivery/promotion.
- XSS/CSP test viewer.
- Load test dashboard dan section viewer.

### 22.10 Kriteria Penerimaan

- Satu aplikasi Next.js tetap digunakan.
- Portal Konsultan memiliki layout dan permission sendiri.
- API Konsultan tidak memuat harga.
- Payment dan assignment terpisah secara domain.
- Viewer tidak mengedit source document.
- Working copy tidak menimpa Local File.
- Queue failure tidak menghapus data review.
- Retry tidak membuat resource ganda.
- Observability tidak menyimpan isi rahasia.
- Kontrak endpoint sesuai [API Contract](./API-Contract.md).
