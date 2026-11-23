# System Architecture — TP Docs By System

## 1. Tujuan Dokumen

Dokumen ini memberikan gambaran visual arsitektur TP Docs sebagai SaaS multi-tenant untuk penyusunan Local File. Visual berfokus pada struktur sistem, batas komponen, penyimpanan data, integrasi eksternal, keamanan tenant, dan unit deployment.

Urutan aktivitas pengguna dan keputusan bisnis tidak dibahas di sini. Detail tersebut berada pada folder `03-Business-Flow`.

### 1.1 Keputusan arsitektur utama

`DiscountPricingService` berada di modul billing NestJS. Ia membaca harga katalog/quotation dan event aktif dari PostgreSQL, mengembalikan satu harga final untuk Next.js, payment, dan audit. Worker/webhook mencatat pemakaian sesudah pembayaran terverifikasi. Xendit hanya menerima total server yang telah dibekukan.


- User Induk, Karyawan, dan Administrator menggunakan satu aplikasi Next.js.
- Administrator menggunakan route `/admin/**`, tetapi NestJS tetap menjadi sumber otorisasi utama.
- Backend menggunakan NestJS modular monolith.
- PostgreSQL menyimpan data terstruktur dan metadata.
- Redis dan BullMQ menangani antrean serta pekerjaan asynchronous.
- Seluruh file permanen disimpan di PostgreSQL sebagai chunk `bytea`; metadata dan isi biner tetap dipisahkan secara logis.
- Pekerjaan berat dijalankan oleh background worker.
- Integrasi Resend, Xendit, AI, dan malware scanner menggunakan adapter; file memakai repository PostgreSQL internal.
- Gemini API dan AI lokal merupakan dua alternatif deployment; implementasi final hanya memilih satu.
- Seluruh endpoint backend menggunakan prefix `/api`.

---

## 2. Legenda Visual

| Notasi | Arti |
| --- | --- |
| Garis penuh `→` | Komunikasi synchronous atau dependency langsung |
| Garis putus-putus `⇢` | Pekerjaan asynchronous, signed access, atau alternatif deployment |
| Garis tebal `⇒` | Webhook dari layanan eksternal |
| Kotak biru | Aplikasi atau service TP Docs |
| Kotak hijau | Penyimpanan data |
| Kotak jingga | Layanan eksternal |
| Kotak abu-abu | Aktor, batas keamanan, atau catatan arsitektur |

---

## 3. System Context Diagram

Diagram berikut memperlihatkan aktor, sistem TP Docs, dan layanan eksternal pada tingkat tertinggi.

```mermaid
flowchart LR
    subgraph ACTORS["Aktor TP Docs"]
        OWNER["User Induk"]
        EMPLOYEE["Karyawan"]
        ADMIN["Administrator"]
    end

    subgraph TPDOCS["TP Docs SaaS"]
        WEB["Satu Aplikasi Web<br/>Next.js"]
        API["Backend API<br/>NestJS Modular Monolith"]
        WORKER["Background Worker<br/>BullMQ"]
    end

    subgraph DATA["Penyimpanan"]
        DB[("PostgreSQL")]
        REDIS[("Redis")]
        STORAGE[("PostgreSQL<br/>Binary File Tables")]
    end

    subgraph EXTERNAL["Layanan Eksternal"]
        RESEND["Resend<br/>Transactional Email"]
        XENDIT["Xendit<br/>Payment Gateway"]
        AI["Gemini API<br/>atau AI Lokal"]
        MALWARE["Malware Scanner"]
    end

    OWNER -->|"Portal perusahaan"| WEB
    EMPLOYEE -->|"Portal perusahaan"| WEB
    ADMIN -->|"Route /admin"| WEB
    WEB -->|"HTTPS /api"| API

    API --> DB
    API --> REDIS
    API --> STORAGE
    API -.->|"Job asynchronous"| WORKER
    WORKER --> REDIS
    WORKER --> STORAGE
    WORKER --> RESEND
    WORKER --> XENDIT
    WORKER --> AI
    WORKER --> MALWARE

    classDef actor fill:#f3f4f6,stroke:#4b5563,color:#111827
    classDef app fill:#dbeafe,stroke:#2563eb,color:#172554
    classDef data fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef external fill:#ffedd5,stroke:#ea580c,color:#7c2d12

    class OWNER,EMPLOYEE,ADMIN actor
    class WEB,API,WORKER app
    class DB,REDIS,STORAGE data
    class RESEND,XENDIT,AI,MALWARE external
```

### 3.1 Batas konteks

- Aplikasi web tidak menyimpan credential provider eksternal.
- Browser tidak mengakses PostgreSQL atau Redis secara langsung.
- Semua keputusan akses data dilakukan oleh backend.
- Konten file hanya dapat di-stream oleh backend setelah validasi tenant, assignment, status scan, paket, dan hak format.
- Administrator tidak memperoleh akses otomatis ke isi Local File hanya karena menggunakan route `/admin/**`.

---

## 4. Application Container Diagram

Diagram container memperlihatkan komponen runtime utama dan jalur komunikasi di dalam platform.

```mermaid
flowchart TB
    BROWSER["Browser<br/>User Induk, Karyawan, Administrator"]

    subgraph EDGE["Edge dan Transport Security"]
        WAF["TLS Termination<br/>WAF atau Reverse Proxy"]
    end

    subgraph APPLICATION["TP Docs Application"]
        WEB["apps/web<br/>Next.js<br/>Public + Portal + /admin"]
        API["apps/api<br/>NestJS Modular Monolith<br/>REST /api"]
        WORKER["apps/worker<br/>NestJS + BullMQ"]
        DOCGEN["Document Generation<br/>DOCX + PDF Conversion"]
    end

    subgraph PLATFORM_DATA["Platform Data Services"]
        POSTGRES[("Managed PostgreSQL<br/>Data dan Metadata")]
        REDIS[("Managed Redis<br/>Queue, Lock, Cache")]
        OBJECT[("PostgreSQL Binary Tables<br/>File Privat")]
        SECRET["Secret Manager"]
    end

    subgraph PROVIDERS["External Providers"]
        RESEND["Resend"]
        XENDIT["Xendit"]
        AIPROVIDER["AI Provider"]
        SCANNER["Malware Scanner"]
    end

    BROWSER -->|"HTTPS"| WAF
    WAF --> WEB
    WEB -->|"HTTPS /api"| API

    API -->|"Query dan transaction"| POSTGRES
    API -->|"Enqueue dan short-lived lock"| REDIS
    API -->|"Metadata dan byte streaming"| OBJECT
    API -->|"Read runtime secret"| SECRET

    REDIS -.->|"Consume BullMQ job"| WORKER
    WORKER --> POSTGRES
    WORKER --> OBJECT
    WORKER --> DOCGEN
    WORKER --> RESEND
    WORKER --> XENDIT
    WORKER --> AIPROVIDER
    WORKER --> SCANNER
    WORKER -->|"Read runtime secret"| SECRET

    BROWSER -.->|"Signed upload atau download terbatas"| OBJECT

    classDef client fill:#f3f4f6,stroke:#4b5563,color:#111827
    classDef app fill:#dbeafe,stroke:#2563eb,color:#172554
    classDef data fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef external fill:#ffedd5,stroke:#ea580c,color:#7c2d12

    class BROWSER,WAF client
    class WEB,API,WORKER,DOCGEN app
    class POSTGRES,REDIS,OBJECT,SECRET data
    class RESEND,XENDIT,AIPROVIDER,SCANNER external
```

### 4.1 Aturan container

- `apps/web` merupakan satu deployment untuk halaman publik, autentikasi, portal perusahaan, dan portal Administrator.
- `apps/api` merupakan sumber authentication, authorization, tenant validation, dan business rule.
- `apps/worker` menangani pekerjaan yang tidak harus selesai dalam request HTTP utama.
- Worker dapat di-scale terpisah dari API.
- AI lokal, bila dipilih, berjalan sebagai service tersendiri dan tidak berada dalam process API.
- Secret hanya dibaca server-side oleh API, worker, atau AI service yang berwenang.

---

## 5. Module Architecture — NestJS Modular Monolith

Modular monolith memisahkan domain secara logis dalam satu backend deployment. Pemisahan ini menjaga konsistensi transaksi tanpa menambah kompleksitas jaringan microservices pada tahap MVP.

```mermaid
flowchart TB
    subgraph PRESENTATION["Presentation Layer"]
        CONTROLLERS["REST Controllers<br/>/api"]
        GUARDS["Authentication, Permission,<br/>Tenant dan Document Guards"]
        WEBHOOKS["Verified Webhook Controllers"]
    end

    subgraph MODULES["Application dan Domain Modules"]
        IDENTITY["Identity dan Authorization"]
        ORG["Organization dan Membership"]
        LOCALFILE["Local File dan Benchmarking"]
        DOCUMENT["Document Generation<br/>dan File Management"]
        COMMERCIAL["Subscription, Quota, Billing,<br/>Payment, Refund, Withdrawal"]
        AICONTENT["AI Content"]
        NOTIFICATION["Notification"]
        GOVERNANCE["Audit dan Administrator"]
    end

    subgraph PORTS["Application Ports"]
        STORAGEPORT["BinaryFileRepository"]
        PAYMENTPORT["PaymentProvider"]
        EMAILPORT["EmailProvider"]
        AIPORT["AiProvider"]
        REPOSITORYPORT["Repository Interfaces"]
        QUEUEPORT["JobQueue"]
    end

    subgraph ADAPTERS["Infrastructure Adapters"]
        PRISMA["Prisma Repositories"]
        BINARYADAPTER["PostgreSQL Binary Repository"]
        XENDITADAPTER["Xendit Adapter"]
        RESENDADAPTER["Resend Adapter"]
        GEMINIADAPTER["Gemini Adapter"]
        LOCALAIADAPTER["Local AI Adapter"]
        BULLMQADAPTER["BullMQ Adapter"]
    end

    CONTROLLERS --> GUARDS
    WEBHOOKS --> GUARDS
    GUARDS --> IDENTITY
    GUARDS --> ORG
    GUARDS --> LOCALFILE
    GUARDS --> DOCUMENT
    GUARDS --> COMMERCIAL
    GUARDS --> AICONTENT
    GUARDS --> GOVERNANCE

    IDENTITY --> REPOSITORYPORT
    ORG --> REPOSITORYPORT
    LOCALFILE --> REPOSITORYPORT
    LOCALFILE --> STORAGEPORT
    DOCUMENT --> STORAGEPORT
    DOCUMENT --> QUEUEPORT
    COMMERCIAL --> PAYMENTPORT
    COMMERCIAL --> REPOSITORYPORT
    AICONTENT --> AIPORT
    AICONTENT --> QUEUEPORT
    NOTIFICATION --> EMAILPORT
    NOTIFICATION --> QUEUEPORT
    GOVERNANCE --> REPOSITORYPORT

    REPOSITORYPORT --> PRISMA
    STORAGEPORT --> BINARYADAPTER
    PAYMENTPORT --> XENDITADAPTER
    EMAILPORT --> RESENDADAPTER
    AIPORT -.->|"Alternatif A"| GEMINIADAPTER
    AIPORT -.->|"Alternatif B"| LOCALAIADAPTER
    QUEUEPORT --> BULLMQADAPTER

    classDef presentation fill:#f3f4f6,stroke:#4b5563,color:#111827
    classDef module fill:#dbeafe,stroke:#2563eb,color:#172554
    classDef port fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef adapter fill:#dcfce7,stroke:#16a34a,color:#14532d

    class CONTROLLERS,GUARDS,WEBHOOKS presentation
    class IDENTITY,ORG,LOCALFILE,DOCUMENT,COMMERCIAL,AICONTENT,NOTIFICATION,GOVERNANCE module
    class STORAGEPORT,PAYMENTPORT,EMAILPORT,AIPORT,REPOSITORYPORT,QUEUEPORT port
    class PRISMA,BINARYADAPTER,XENDITADAPTER,RESENDADAPTER,GEMINIADAPTER,LOCALAIADAPTER,BULLMQADAPTER adapter
```

### 5.1 Dependency rule

- Domain dan application module bergantung pada interface, bukan SDK vendor.
- Infrastructure adapter mengimplementasikan interface yang ditetapkan application layer.
- Controller tidak mengakses Prisma, Redis, SDK Xendit, SDK Resend, atau repository file secara langsung.
- Gemini adapter dan local AI adapter tidak digunakan bersamaan pada satu environment production.
- Perubahan vendor tidak mengubah kontrak domain selama interface tetap kompatibel.

---

## 6. Data dan Asynchronous Processing

Diagram ini menunjukkan pergerakan data teknis dan hubungan synchronous, asynchronous, serta webhook. Diagram ini bukan business flow pengguna.

```mermaid
flowchart LR
    WEB["Next.js Web"]
    API["NestJS API"]
    DB[("PostgreSQL<br/>Structured Data, Metadata,<br/>Job Status, Audit Log")]
    REDIS[("Redis<br/>BullMQ, Lock, Cache Terbatas")]
    WORKER["BullMQ Worker"]
    STORAGE[("PostgreSQL File Tables<br/>Upload, Template,<br/>DOCX, PDF, Benchmarking")]
    RESEND["Resend"]
    XENDIT["Xendit"]
    AI["AI Provider"]
    SCANNER["Malware Scanner"]
    IDEMPOTENCY["Webhook Verification<br/>dan Idempotency"]

    WEB -->|"Request /api"| API
    API -->|"Transaction dan metadata"| DB
    API -->|"Publish job"| REDIS
    REDIS -.->|"Consume job"| WORKER

    WORKER -->|"Status dan hasil"| DB
    WORKER -->|"Read atau write chunks"| STORAGE
    WORKER --> RESEND
    WORKER --> XENDIT
    WORKER --> AI
    WORKER --> SCANNER

    XENDIT ==>|"Payment, refund, payout webhook"| IDEMPOTENCY
    RESEND ==>|"Delivery status webhook"| IDEMPOTENCY
    IDEMPOTENCY --> API
    API -->|"Event ID dan status terverifikasi"| DB

    API -->|"Stream file terautentikasi"| WEB

    classDef app fill:#dbeafe,stroke:#2563eb,color:#172554
    classDef data fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef external fill:#ffedd5,stroke:#ea580c,color:#7c2d12
    classDef control fill:#f3f4f6,stroke:#4b5563,color:#111827

    class WEB,API,WORKER app
    class DB,REDIS,STORAGE data
    class RESEND,XENDIT,AI,SCANNER external
    class IDEMPOTENCY control
```

### 6.1 Data ownership

- PostgreSQL merupakan sumber kebenaran data bisnis, status, dan metadata.
- Redis bukan sumber kebenaran permanen.
- Isi file biner tidak disimpan dalam PostgreSQL.
- File ID, checksum, MIME type, ukuran, jumlah chunk, tenant, dan status scan disimpan sebagai metadata.
- Webhook disimpan dan diproses secara idempotent.
- Setiap background job membawa tenant context dan correlation ID yang tervalidasi.

---

## 7. AI Deployment Alternatives

Kedua alternatif menggunakan kontrak `AiProvider` yang sama. Hanya satu adapter yang diaktifkan pada environment production setelah evaluasi kualitas, privasi, latency, kapasitas, dan biaya.

```mermaid
flowchart LR
    MODULE["AI Content Module<br/>NestJS"]
    PORT["AiProvider Interface"]
    NOTE["Pilih satu setelah evaluasi<br/>Tidak ada fallback otomatis"]

    subgraph OPTION_API["Alternatif A — External API"]
        GEMINIADAPTER["GeminiAiProvider"]
        GEMINI["Gemini API"]
        GEMINIADAPTER -->|"Structured request dan response"| GEMINI
    end

    subgraph OPTION_LOCAL["Alternatif B — Self-hosted AI"]
        LOCALADAPTER["LocalAiProvider"]
        FASTAPI["FastAPI<br/>Internal Inference API"]
        MODEL["Local Model Server<br/>GPU"]
        LOCALADAPTER --> FASTAPI
        FASTAPI --> MODEL
    end

    MODULE --> PORT
    PORT -.->|"Kandidat MVP"| GEMINIADAPTER
    PORT -.->|"Kandidat setelah benchmark"| LOCALADAPTER
    NOTE -.- PORT

    classDef app fill:#dbeafe,stroke:#2563eb,color:#172554
    classDef port fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef external fill:#ffedd5,stroke:#ea580c,color:#7c2d12
    classDef note fill:#f3f4f6,stroke:#4b5563,color:#111827

    class MODULE,GEMINIADAPTER,LOCALADAPTER,FASTAPI app
    class PORT port
    class GEMINI,MODEL external
    class NOTE note
```

### 7.1 Ketentuan bersama

- Input disanitasi dan dibatasi pada data yang diperlukan.
- Output wajib mengikuti schema fase proses bisnis yang ditentukan aplikasi.
- Prompt dan respons mentah tidak ditulis pada application log.
- Data pelanggan tidak digunakan untuk training otomatis.
- Data dan hasil satu tenant tidak digunakan sebagai konteks tenant lain.
- Kegagalan provider tidak menghapus uraian asli pengguna.

---

## 8. Security dan Tenant Boundary

Keamanan diterapkan berlapis. Route frontend membantu navigasi, tetapi backend tetap melakukan validasi untuk setiap request.

```mermaid
flowchart TB
    REQUEST["HTTPS Request"]

    subgraph EDGE["Edge Security"]
        TLS["TLS, WAF atau Reverse Proxy"]
    end

    subgraph WEB_LAYER["Next.js Security Boundary"]
        SESSION["Session Check"]
        ROUTE["Route Protection<br/>Portal atau /admin"]
    end

    subgraph API_LAYER["NestJS Security Boundary"]
        AUTH["Authentication Guard"]
        PERMISSION["Role dan Permission Guard"]
        TENANT["Tenant Validation"]
        ASSIGNMENT["Document Assignment Validation<br/>bila resource berupa dokumen"]
        SERVICE["Application Service"]
    end

    subgraph DATA_BOUNDARY["Tenant-aware Data Boundary"]
        DB[("PostgreSQL<br/>organization_id dan constraint")]
        OBJECT[("File Metadata + bytea Chunks<br/>tenant validation")]
        CACHE[("Redis<br/>tenant-scoped key dan job context")]
    end

    AUDIT[("Append-only Audit Log")]
    DENIED["401 atau 403<br/>tanpa membuka data"]

    REQUEST --> TLS
    TLS --> SESSION
    SESSION --> ROUTE
    ROUTE --> AUTH
    AUTH --> PERMISSION
    PERMISSION --> TENANT
    TENANT --> ASSIGNMENT
    ASSIGNMENT --> SERVICE

    AUTH -.->|"Jika tidak valid"| DENIED
    PERMISSION -.->|"Jika tidak berwenang"| DENIED
    TENANT -.->|"Jika tenant berbeda"| DENIED
    ASSIGNMENT -.->|"Jika tidak ditugaskan"| DENIED

    SERVICE --> DB
    SERVICE --> OBJECT
    SERVICE --> CACHE

    AUTH -.-> AUDIT
    PERMISSION -.-> AUDIT
    TENANT -.-> AUDIT
    SERVICE -.-> AUDIT

    classDef control fill:#f3f4f6,stroke:#4b5563,color:#111827
    classDef app fill:#dbeafe,stroke:#2563eb,color:#172554
    classDef data fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef denied fill:#fee2e2,stroke:#dc2626,color:#7f1d1d

    class REQUEST,TLS,SESSION,ROUTE control
    class AUTH,PERMISSION,TENANT,ASSIGNMENT,SERVICE app
    class DB,OBJECT,CACHE,AUDIT data
    class DENIED denied
```

### 8.1 Aturan keamanan

- `/admin/**` hanya dapat digunakan role `ADMINISTRATOR`.
- Mengetahui URL `/admin` tidak memberikan hak akses.
- User Induk dan Karyawan yang mencoba mengakses resource Administrator menerima `403 Forbidden`.
- User Induk hanya dapat mengakses organisasinya.
- Karyawan hanya dapat mengakses dokumen yang ditugaskan.
- Administrator tidak otomatis dapat membaca isi dokumen pelanggan.
- Tenant context diterapkan pada query metadata file, pembacaan chunk, cache key, queue job, search, dan background process.
- Akses file menggunakan streaming NestJS dengan otorisasi resource dan HTTP Range bila diperlukan.
- Secret tidak pernah dikirim ke browser.

---

## 9. Deployment View

Diagram deployment memisahkan unit aplikasi, managed data service, secret, dan provider eksternal.

```mermaid
flowchart TB
    subgraph ENVIRONMENTS["Environment Terisolasi"]
        DEV["Development<br/>Credential dan resource development"]
        STAGING["Staging<br/>Credential dan resource staging"]
        PROD["Production<br/>Credential dan resource production"]
    end

    subgraph PROD_REGION["Production Cloud Region — Pending Validation"]
        EDGE["WAF atau Reverse Proxy"]

        subgraph COMPUTE["Container Compute"]
            WEB["Next.js Web Container"]
            API["NestJS API Container"]
            WORKER["BullMQ Worker Container"]
            LOCALAI["FastAPI + Local Model<br/>Hanya jika AI lokal dipilih"]
        end

        subgraph MANAGED_DATA["Managed Data Services"]
            POSTGRES[("PostgreSQL")]
            REDIS[("Redis")]
            STORAGE[("PostgreSQL Binary File Tables")]
            SECRETS["Secret Manager"]
        end
    end

    subgraph EXTERNAL["External Services"]
        RESEND["Resend"]
        XENDIT["Xendit"]
        GEMINI["Gemini API<br/>Hanya jika API dipilih"]
        SCANNER["Malware Scanner"]
    end

    DEV -.->|"Resource terpisah"| EDGE
    STAGING -.->|"Resource terpisah"| EDGE
    PROD --> EDGE

    EDGE --> WEB
    WEB --> API
    API --> POSTGRES
    API --> REDIS
    API --> STORAGE
    API --> SECRETS

    REDIS -.-> WORKER
    WORKER --> POSTGRES
    WORKER --> STORAGE
    WORKER --> SECRETS
    WORKER --> RESEND
    WORKER --> XENDIT
    WORKER --> SCANNER
    WORKER -.->|"Alternatif API"| GEMINI
    WORKER -.->|"Alternatif lokal"| LOCALAI

    classDef environment fill:#f3f4f6,stroke:#4b5563,color:#111827
    classDef app fill:#dbeafe,stroke:#2563eb,color:#172554
    classDef data fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef external fill:#ffedd5,stroke:#ea580c,color:#7c2d12

    class DEV,STAGING,PROD,EDGE environment
    class WEB,API,WORKER,LOCALAI app
    class POSTGRES,REDIS,STORAGE,SECRETS data
    class RESEND,XENDIT,GEMINI,SCANNER external
```

### 9.1 Deployment constraints

- Development, staging, dan production tidak berbagi database, Redis, API key, atau credential payment.
- Web, API, dan worker dapat di-scale secara terpisah.
- AI lokal menggunakan compute GPU terpisah dari API utama.
- Managed PostgreSQL dan Redis direkomendasikan untuk production.
- Region cloud, malware scanner, dan model AI final berstatus **Pending Validation**.
- Backup, retensi, dan disaster recovery mengikuti Non-Functional Requirements.

---

## 10. Tanggung Jawab Komponen

| Komponen | Tanggung jawab utama | Tidak boleh dilakukan |
| --- | --- | --- |
| Next.js Web | Rendering UI, route, session boundary awal, dan pemanggilan `/api` | Mengakses database atau menyimpan secret provider |
| NestJS API | Authentication, authorization, tenant validation, business rule, dan kontrak REST | Mempercayai role, harga, tenant, atau file ID dari frontend tanpa validasi |
| BullMQ Worker | Menjalankan pekerjaan asynchronous dan retry terkontrol | Menjalankan job tanpa tenant context atau idempotency |
| PostgreSQL | Data bisnis, metadata, status, dan audit log | Menyimpan file biner Local File |
| Redis | Queue, lock sementara, rate-control, dan cache terbatas | Menjadi sumber kebenaran data bisnis permanen |
| PostgreSQL Binary Repository | Upload, template, benchmarking, DOCX, dan PDF privat | Mengekspos BLOB langsung atau mengambilnya pada query daftar |
| Document Generator | Menghasilkan DOCX dan PDF dari snapshot data tervalidasi | Mengubah business data sumber |
| Resend Adapter | Mengirim email transactional dan memetakan delivery status | Menjadi mailbox pelanggan atau menyimpan password/reset token pada log |
| Xendit Adapter | Payment, refund, balance, payout, webhook, dan status mapping | Menetapkan harga paket atau mengubah business rule |
| AI Adapter | Mengubah request internal menjadi request provider dan memvalidasi output | Menyimpan prompt lintas tenant atau melatih model otomatis dari data pelanggan |
| Malware Scanner | Memindai upload dan memberi hasil scan | Membuka akses file yang belum dinyatakan aman |
| Secret Manager | Menyimpan dan merotasi credential runtime | Mengirim secret ke browser atau source code |

---

## 11. Jenis Data dan Lokasi Penyimpanan

| Jenis data | Lokasi utama | Perlindungan teknis |
| --- | --- | --- |
| Akun, organisasi, membership | PostgreSQL | Encryption at rest, constraint, tenant validation |
| Profil perusahaan dan NPWP | PostgreSQL | Least privilege, masking pada log, tenant scope |
| Data Local File dan FAR | PostgreSQL | Tenant scope, authorization, audit |
| Transaksi afiliasi dan benchmarking | PostgreSQL | Transaction, versioning, tenant scope |
| Metadata file | PostgreSQL `files` | File ID, checksum, jumlah chunk, status konten, status scan, dan retensi |
| Excel, PDF, DOCX, template | PostgreSQL `files` + `file_blob_chunks` | Encryption, tenant validation, checksum, API streaming |
| Queue dan job sementara | Redis | Tenant context, TTL, access restriction |
| Lock dan cache sementara | Redis | TTL, tenant-scoped key, tidak menyimpan secret |
| Credential provider | Secret manager | Access policy dan rotation |
| Audit log | PostgreSQL atau audit store terkontrol | Append-only dari aplikasi dan retensi minimal sesuai NFR |

---

## 12. Integrasi Eksternal dan Adapter

| Integrasi | Interface internal | Adapter | Arah komunikasi | Status keputusan |
| --- | --- | --- | --- | --- |
| Transactional email | `EmailProvider` | Resend adapter | Outbound API dan inbound delivery webhook | Dipilih |
| Payment gateway | `PaymentProvider` | Xendit adapter | Outbound API dan inbound verified webhook | Kandidat utama; kontrak perlu validasi |
| File storage | `BinaryFileRepository` | PostgreSQL binary repository | Server-side upload dan authenticated streaming | Aktif |
| AI eksternal | `AiProvider` | Gemini adapter | Outbound API | Kandidat MVP |
| AI lokal | `AiProvider` | Local AI adapter | Internal FastAPI | Pending Benchmark |
| Malware scanning | `MalwareScanner` | Provider adapter | Internal atau outbound API | Vendor Pending Validation |

---

## 13. Aturan Arsitektur yang Wajib Dijaga

- Frontend hanya berkomunikasi dengan backend melalui `/api`, kecuali signed file access yang diterbitkan backend.
- SDK vendor hanya berada pada infrastructure adapter.
- Domain module tidak mengimpor SDK Resend, Xendit, Gemini, atau implementasi repository file.
- Background job wajib idempotent, memiliki batas retry, dan membawa tenant context.
- Webhook wajib diverifikasi dan dideduplicasi sebelum mengubah status internal.
- Database—including metadata dan chunk file—Redis, search, dan queue harus mempertahankan isolasi organisasi.
- Route protection frontend tidak menggantikan backend authorization.
- Data rahasia tidak dimasukkan ke log, telemetry, atau error message.
- Gemini API dan AI lokal tidak diaktifkan bersamaan sebagai fallback otomatis.
- Perubahan arsitektur utama dicatat dalam Architecture Decision Record.

---

## 14. Arsitektur Integrasi Xendit

### 14.1 Payment Boundary

```mermaid
flowchart LR
    WEB[Next.js] -->|/api checkout dan status| API[NestJS Billing Module]
    API --> DOMAIN[Payment Domain]
    DOMAIN --> PORT[PaymentProvider Port]
    PORT --> ADAPTER[Xendit Adapter]
    ADAPTER -->|POST /sessions| XENDIT[Xendit API]
    XENDIT -->|payment_link_url| ADAPTER
    API --> DB[(PostgreSQL)]

    XENDIT ==>|Payment / Refund / Dispute Event| HOOK[Webhook Controller]
    HOOK --> VERIFY[Verification + Dedupe]
    VERIFY --> QUEUE[(BullMQ)]
    QUEUE --> WORKER[Payment Event Worker]
    WORKER --> DB
    WORKER --> ENTITLEMENT[Subscription dan Quota Service]
```

Frontend tidak memanggil Xendit secara langsung selain membuka hosted `payment_link_url`. Seluruh pembuatan session, retrieval status, refund, dan payout dilakukan server-side melalui adapter. API key, webhook credential, serta account identifier berada di secret manager.

### 14.2 Interface Internal

```ts
interface PaymentProvider {
  upsertBusinessCustomer(input: ProviderBusinessCustomerInput): Promise<ProviderCustomer>;
  createPaymentSession(input: CreatePaymentSessionInput): Promise<ProviderPaymentSession>;
  getPayment(input: GetProviderPaymentInput): Promise<ProviderPayment>;
  createRefund(input: CreateProviderRefundInput): Promise<ProviderRefund>;
  getRefund(input: GetProviderRefundInput): Promise<ProviderRefund>;
  verifyWebhook(input: VerifyPaymentWebhookInput): Promise<VerifiedProviderEvent>;
  getPayout?(input: GetProviderPayoutInput): Promise<ProviderPayout>;
}
```

Domain tidak mengenal payload mentah Xendit. Adapter memetakan `payment_session_id`, `payment_request_id`, `payment_id`, `refund_id`, status, actions, serta failure code ke model internal.

### 14.3 Data Ownership dan Status

| Data | Sumber kebenaran |
| --- | --- |
| Harga, purpose, paket, quotation | PostgreSQL TP Docs |
| Hosted checkout URL dan expiry | Xendit; cache/snapshot pada `payment_sessions` |
| Status domain payment | PostgreSQL setelah verified webhook/reconciliation |
| Provider status mentah | Metadata payment session/webhook event |
| Entitlement, quota, kursi | Subscription dan quota ledger TP Docs |
| Refund policy dan access adjustment | TP Docs |
| Status refund provider | Xendit, dipetakan ke refund internal |
| Dispute dan due date | Xendit event, disimpan untuk tindakan Admin |

Payment dan entitlement tidak diperbarui dalam callback browser. Worker memproses verified event dengan database transaction dan lock; perubahan `PAID` dan pemberian hak harus atomic atau dapat dilanjutkan secara idempotent.

### 14.4 Customer, Account, dan Payout

- `Organization` dapat dipetakan ke satu Xendit customer `BUSINESS` tetapi bukan Xendit Account/sub-account.
- XenPlatform tidak digunakan karena pelanggan TP Docs bukan merchant penerima pembayaran.
- Refund selalu mengacu pada payment asal.
- Payout hanya untuk transfer keluar yang telah disetujui dan bukan refund.
- Pencairan saldo dari portal Admin berstatus fitur terkendali/Pending Validation; rekening tujuan harus allowlist dan tindakan memerlukan dual approval.
- Settlement report dapat digunakan untuk rekonsiliasi tanpa memberi Administrator akses ke secret provider.

### 14.5 Webhook dan Failure Isolation

- verifikasi dilakukan sebelum parsing bisnis;
- event disimpan dengan dedupe key dan payload hash;
- handler segera mengembalikan respons aman dan pekerjaan bisnis diteruskan ke queue;
- event out-of-order tidak boleh menurunkan status terminal;
- amount, currency, reference ID, dan resource ID wajib cocok;
- retry Xendit atau BullMQ tidak menggandakan subscription, quota, refund, maupun payout;
- kegagalan Xendit tidak mengganggu Local File yang sudah tersimpan;
- rekonsiliasi berkala mengambil status provider untuk payment yang terlalu lama `PENDING/PROCESSING`.

Referensi resmi: [Payment Session](https://docs.xendit.co/docs/subscription-with-initial-payment), [Payment Request](https://docs.xendit.co/apidocs/get-payment-request), [Refund](https://docs.xendit.co/apidocs/get-a-refund), [Payout](https://docs.xendit.co/apidocs/get-payout-v3-by-id), dan [Dispute](https://docs.xendit.co/docs/manage-disputes-via-api).

---

## 15. Referensi

- [Technology Stack](./Technology-Stack.md)
- [API Contract](./API-Contract.md)
- [AI Processing Architecture](./AI-Processing-Architecture.md)
- [Document Generation Architecture](./Document-Generation-Architecture.md)
- [Database Design](../04-Database/README.md)
- [Database ERD](../04-Database/ERD.dbml)
- [Query Optimization dan Pencegahan N+1](../04-Database/Query-Optimization.md)
- [Business Flow Overview](../03-Business-Flow/00-Business-Flow-Overview.md)
- [Document Generation Flow](../03-Business-Flow/02-Document-Generation.md)
- [AI Content Generation Flow](../03-Business-Flow/04-AI-Content-Generation.md)
- [Subscription and Billing Flow](../03-Business-Flow/06-Subscription-Billing.md)
- [Administrator Flow](../03-Business-Flow/07-Administrator-Flow.md)
