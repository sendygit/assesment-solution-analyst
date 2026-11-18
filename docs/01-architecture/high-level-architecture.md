# High-Level Architecture

## Diagram

```mermaid
flowchart TB
    subgraph Client["Mobile Client"]
        A1[Mobile App - Android/iOS]
    end

    subgraph Edge["Edge Layer"]
        LB[Load Balancer]
        GW[API Gateway<br/>Auth, Rate Limit, Routing]
    end

    subgraph Backend["Backend - Monolith Modular"]
        AuthM[Auth Module<br/>Login, Register, Biometric Token]
        UserM[User Module<br/>Profile, KYC Status]
        LoanM[Loan Module<br/>Apply, Approval, Status]
        RepayM[Repayment Module<br/>Billing, Due, History]
        NotifM[Notification Module<br/>Email/SMS Trigger]
        DocM[Document Module<br/>KTP/Selfie Upload]
    end

    subgraph Async["Async Processing"]
        Queue[(Message Queue<br/>e.g. RabbitMQ/SQS)]
        Worker[Worker Service<br/>Credit Scoring Job,<br/>Notification Sender]
    end

    subgraph Data["Data Layer"]
        DB[(Primary DB<br/>PostgreSQL)]
        Cache[(Cache<br/>Redis - session, OTP)]
        Blob[(Object Storage<br/>KTP/Selfie Images)]
    end

    subgraph ThirdParty["3rd Party Services"]
        KYC[KYC/OCR Provider<br/>e.g. Privy/VIDA]
        CreditScore[Credit Scoring Engine<br/>internal rule-engine or external bureau]
        EmailSvc[Email Provider]
        SmsSvc[SMS Provider]
    end

    A1 -->|HTTPS/REST| LB --> GW
    GW --> AuthM
    GW --> UserM
    GW --> LoanM
    GW --> RepayM
    GW --> DocM

    AuthM --> Cache
    AuthM --> DB
    UserM --> DB
    DocM --> Blob
    DocM --> KYC
    LoanM --> DB
    LoanM --> Queue
    RepayM --> DB

    Queue --> Worker
    Worker --> CreditScore
    Worker --> NotifM
    NotifM --> EmailSvc
    NotifM --> SmsSvc
    NotifM --> DB
```

## Komponen Utama

| Komponen | Tanggung Jawab |
|---|---|
| **Mobile App** | Native/cross-platform app (Flutter/React Native), menyimpan token sesi, biometric prompt lokal |
| **API Gateway** | Single entry point: autentikasi token (JWT), rate limiting, routing ke modul backend |
| **Auth Module** | Register, login (password + biometric token exchange), OTP verifikasi email/telepon |
| **Document Module** | Upload foto diri + KTP, kirim ke KYC provider untuk OCR & liveness check |
| **User Module** | Data profil user, status KYC |
| **Loan Module** | Pengajuan pinjaman, cek eligibility (ada pinjaman aktif/tidak), trigger proses scoring |
| **Repayment Module** | Kalkulasi tagihan bulanan, sisa hutang, histori pembayaran |
| **Notification Module** | Mengirim notifikasi hasil pengajuan via email & SMS |
| **Message Queue + Worker** | Memproses credit scoring secara async (tidak blocking request), mengirim notifikasi async |
| **Credit Scoring Engine** | Menentukan diterima/ditolak berdasarkan rule/skor kredit |
| **Object Storage** | Penyimpanan file foto KTP & selfie (terenkripsi, akses terbatas) |

## Alasan Desain

- **Async untuk proses scoring**: pengajuan pinjaman tidak selalu instan (tergantung 3rd party), sehingga request `POST /loans` langsung return `PENDING` dan status final diupdate lewat worker + push notifikasi.
- **API Gateway** sebagai satu pintu masuk memudahkan pengelolaan autentikasi & rate limiting, penting untuk aplikasi fintech yang rawan abuse (fraud, brute force).
- **Object Storage terpisah dari DB utama** untuk file KTP/selfie — lebih murah, scalable, dan memudahkan kontrol akses/enkripsi data sensitif secara terpisah.
- **Monolith modular** dipilih karena skala awal startup fintech belum butuh kompleksitas microservices; modul tetap dipisah secara logis agar mudah di-extract jadi service independen saat scale bertambah.
