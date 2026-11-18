# High-Level Architecture

## Diagram

```mermaid
flowchart TB
    subgraph Client["Mobile Client"]
        A1[Mobile App Flutter<br/>Android & iOS]
    end

    subgraph Edge["Edge Layer"]
        LB[Load Balancer]
        GW[API Gateway<br/>verifikasi JWT, rate limit, routing]
    end

    subgraph Backend["Backend NestJS - Monolith Modular"]
        AuthM[Auth Module<br/>register, OTP, login, sesi, biometric]
        UserM[User Module<br/>profil, device, status KYC]
        DocM[Document Module<br/>upload KTP/selfie, webhook KYC]
        LoanM[Loan Module<br/>simulasi, pengajuan, aturan blokir]
        RepayM[Repayment Module<br/>jadwal cicilan, sisa hutang]
        NotifM[Notification Module<br/>email, SMS]
    end

    subgraph Async["Async Processing"]
        Queue[(Amazon SQS<br/>+ dead-letter queue)]
        Worker[Worker Service<br/>credit scoring job,<br/>pengiriman notifikasi]
    end

    subgraph Data["Data Layer"]
        DB[(PostgreSQL)]
        Cache[(Redis<br/>OTP, challenge, rate limit)]
        Blob[(Amazon S3<br/>KTP/selfie terenkripsi)]
    end

    subgraph ThirdParty["Layanan Pihak Ketiga"]
        KYC[KYC Provider VIDA<br/>OCR, liveness, Dukcapil]
        CreditScore[Credit Scoring Engine<br/>rule internal + biro kredit]
        EmailSvc[Amazon SES]
        SmsSvc[SMS Gateway Lokal]
    end

    A1 -->|HTTPS/REST| LB --> GW
    GW --> AuthM
    GW --> UserM
    GW --> DocM
    GW --> LoanM
    GW --> RepayM

    AuthM --> Cache
    AuthM --> DB
    AuthM -->|OTP| SmsSvc
    UserM --> DB
    DocM --> Blob
    DocM -->|submit verifikasi| KYC
    KYC -.->|webhook hasil| GW
    DocM --> DB
    LoanM --> DB
    LoanM -->|LoanSubmitted| Queue
    RepayM --> DB

    Queue --> Worker
    Worker --> CreditScore
    Worker --> DB
    Worker --> NotifM
    NotifM --> EmailSvc
    NotifM --> SmsSvc
    NotifM --> DB
```

## Komponen Utama

| Komponen | Tanggung Jawab |
|---|---|
| **Mobile App (Flutter)** | UI, kamera untuk KTP/selfie, biometric prompt lokal, menyimpan token di secure storage |
| **Load Balancer** | Membagi traffic ke beberapa instance backend dan mengalihkan traffic jika satu instance mati |
| **API Gateway** | Pintu masuk tunggal: verifikasi JWT, rate limiting, routing ke modul backend |
| **Auth Module** | Registrasi, OTP, onboarding token, login password, refresh/logout, challenge dan verifikasi biometric, lupa password |
| **User Module** | Profil user, device untuk biometric, status KYC |
| **Document Module** | Upload KTP/selfie ke S3, mengirim ke provider KYC, menerima webhook hasil verifikasi |
| **Loan Module** | Simulasi cicilan, pengajuan, aturan "satu pinjaman berjalan", publish event ke queue |
| **Repayment Module** | Jadwal cicilan, sisa hutang, tagihan bulan berjalan |
| **Notification Module** | Mengirim notifikasi hasil pengajuan (disetujui/ditolak) via email dan SMS, serta mencatat status pengiriman |
| **Message Queue + Worker** | Menjalankan credit scoring dan pengiriman notifikasi di luar request user, dengan retry dan dead-letter queue |
| **Credit Scoring Engine** | Menentukan disetujui/ditolak dari data profil (umur, pekerjaan, penghasilan) dan data biro kredit |
| **Object Storage** | Menyimpan file KTP dan selfie, terenkripsi, diakses lewat signed URL |

## Alasan Desain

- **Scoring dan KYC asinkron.** Keduanya bergantung pada pihak ketiga yang bisa lambat atau down. `POST /loans` langsung membalas `202` dengan status `SUBMITTED`, dan KYC membalas `202` lalu hasilnya datang lewat webhook. User tidak menunggu dengan koneksi terbuka, dan kegagalan provider di-retry dari queue.
- **API Gateway** sebagai satu pintu masuk memusatkan autentikasi dan rate limiting, penting untuk aplikasi fintech yang rawan brute force dan fraud.
- **Object storage terpisah dari database** untuk file KTP/selfie: lebih murah, mudah di-scale, dan akses ke data sensitif bisa dikontrol terpisah.
- **Monolith modular** dipilih karena skala awal belum butuh kompleksitas microservices. Modul dipisah secara logis sehingga bisa diangkat menjadi service sendiri saat skala bertambah. Worker sudah terpisah sejak awal karena pola bebannya berbeda.
