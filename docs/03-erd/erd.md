# Entity Relationship Diagram

```mermaid
erDiagram
    USER ||--o{ DOCUMENT : uploads
    USER ||--o{ LOAN : requests
    USER ||--o{ AUTH_SESSION : has
    USER ||--o{ NOTIFICATION : receives
    LOAN ||--o{ REPAYMENT_SCHEDULE : generates
    LOAN ||--o| LOAN_DECISION : "has result"
    REPAYMENT_SCHEDULE ||--o{ PAYMENT : "paid via"

    USER {
        uuid id PK
        string full_name
        string email UK
        string phone_number UK
        string nik UK
        string password_hash
        string biometric_public_key
        enum kyc_status "PENDING, VERIFIED, REJECTED"
        datetime created_at
        datetime updated_at
    }

    DOCUMENT {
        uuid id PK
        uuid user_id FK
        enum type "SELFIE, KTP"
        string file_url
        enum verification_status "PENDING, VERIFIED, REJECTED"
        string verification_provider_ref
        datetime uploaded_at
    }

    AUTH_SESSION {
        uuid id PK
        uuid user_id FK
        string refresh_token_hash
        enum login_method "PASSWORD, BIOMETRIC"
        string device_id
        datetime expires_at
        datetime created_at
    }

    LOAN {
        uuid id PK
        uuid user_id FK
        decimal principal_amount "max 12000000"
        int tenor_months "max 12"
        enum status "SUBMITTED, PROCESSING, APPROVED, REJECTED, ACTIVE, PAID_OFF"
        datetime submitted_at
        datetime decided_at
    }

    LOAN_DECISION {
        uuid id PK
        uuid loan_id FK
        enum result "APPROVED, REJECTED"
        string reason_code
        decimal credit_score
        datetime decided_at
    }

    REPAYMENT_SCHEDULE {
        uuid id PK
        uuid loan_id FK
        int installment_number
        decimal amount_due
        date due_date
        enum status "UPCOMING, DUE, PAID, OVERDUE"
    }

    PAYMENT {
        uuid id PK
        uuid repayment_schedule_id FK
        decimal amount_paid
        string payment_method
        datetime paid_at
    }

    NOTIFICATION {
        uuid id PK
        uuid user_id FK
        uuid loan_id FK
        enum channel "EMAIL, SMS"
        enum type "LOAN_APPROVED, LOAN_REJECTED, PAYMENT_REMINDER"
        enum delivery_status "SENT, FAILED"
        datetime sent_at
    }
```

## Catatan

- `USER.kyc_status` harus `VERIFIED` sebelum user bisa mengajukan pinjaman (`LOAN`).
- Constraint bisnis "tidak boleh pinjam jika ada pinjaman aktif" divalidasi di level aplikasi: cek apakah ada `LOAN` milik user dengan status `APPROVED`/`ACTIVE` sebelum mengizinkan `INSERT` baru.
- `LOAN_DECISION` dipisah dari `LOAN` agar histori keputusan scoring (termasuk reason code & score) tidak bercampur dengan data pengajuan, dan memudahkan audit trail.
- `REPAYMENT_SCHEDULE` di-generate otomatis saat `LOAN.status = APPROVED`, berdasarkan `principal_amount` dan `tenor_months`.
