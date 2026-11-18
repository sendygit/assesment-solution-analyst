# Entity Relationship Diagram

```mermaid
erDiagram
    USER ||--o{ DEVICE : uses
    USER ||--o{ AUTH_SESSION : has
    USER ||--o{ DOCUMENT : uploads
    USER ||--o{ CONSENT : gives
    USER ||--o{ LOAN : requests
    USER ||--o{ NOTIFICATION : receives
    DEVICE ||--o{ AUTH_SESSION : "logged in from"
    LOAN ||--o| LOAN_DECISION : "decided by"
    LOAN ||--o{ REPAYMENT_SCHEDULE : generates
    LOAN ||--o{ CONSENT : "agreement of"

    USER {
        uuid id PK
        string full_name
        string nik_encrypted "AES, field-level encryption"
        string nik_hash UK "untuk cek duplikasi tanpa dekripsi"
        date date_of_birth
        string email UK
        string phone_number UK
        string occupation
        decimal monthly_income "15,2"
        string password_hash "null sampai onboarding selesai"
        boolean email_verified
        boolean phone_verified
        enum kyc_status "NOT_SUBMITTED, PENDING, VERIFIED, REJECTED"
        enum status "ONBOARDING, ACTIVE, SUSPENDED"
        datetime created_at
        datetime updated_at
    }

    DEVICE {
        uuid id PK
        uuid user_id FK
        string device_id
        enum platform "ANDROID, IOS"
        string biometric_public_key "null jika biometric tidak aktif"
        datetime last_login_at
    }

    AUTH_SESSION {
        uuid id PK
        uuid user_id FK
        uuid device_id FK
        string refresh_token_hash
        enum login_method "PASSWORD, BIOMETRIC"
        datetime expires_at
        datetime revoked_at
        datetime created_at
    }

    DOCUMENT {
        uuid id PK
        uuid user_id FK
        enum type "SELFIE, KTP"
        string file_key "object key, diakses via signed URL"
        int attempt_no
        enum verification_status "PENDING, VERIFIED, REJECTED"
        string provider_ref
        string reason_code
        datetime uploaded_at
    }

    CONSENT {
        uuid id PK
        uuid user_id FK
        uuid loan_id FK "null untuk consent registrasi"
        enum consent_type "TNC_PRIVACY, LOAN_AGREEMENT"
        string document_version
        string ip_address
        string device_id
        datetime agreed_at
    }

    LOAN {
        uuid id PK
        uuid user_id FK
        decimal principal_amount "15,2, max 12.000.000"
        int tenor_months "1-12"
        decimal interest_rate_monthly "5,4, flat"
        decimal monthly_installment "15,2"
        decimal total_repayment "15,2"
        enum status "SUBMITTED, PROCESSING, APPROVED, REJECTED, ACTIVE, PAID_OFF"
        string idempotency_key UK
        datetime submitted_at
        datetime disbursed_at
        datetime paid_off_at
    }

    LOAN_DECISION {
        uuid id PK
        uuid loan_id FK, UK
        enum result "APPROVED, REJECTED"
        decimal credit_score
        string reason_code
        string scoring_model_version
        datetime decided_at
    }

    REPAYMENT_SCHEDULE {
        uuid id PK
        uuid loan_id FK
        int installment_number
        decimal principal_part "15,2"
        decimal interest_part "15,2"
        decimal amount_due "15,2"
        date due_date
        enum status "UPCOMING, DUE, PAID, OVERDUE"
        datetime paid_at
    }

    NOTIFICATION {
        uuid id PK
        uuid user_id FK
        uuid loan_id FK
        enum channel "EMAIL, SMS"
        enum type "LOAN_APPROVED, LOAN_REJECTED, PAYMENT_REMINDER"
        enum delivery_status "SENT, FAILED"
        int retry_count
        datetime sent_at
    }
```

## Aturan Bisnis di Level Data

**1. Satu pinjaman berjalan per user.** Pinjaman dianggap masih berjalan jika berstatus `SUBMITTED`, `PROCESSING`, `APPROVED`, atau `ACTIVE`. Sesuai brief: user tidak bisa meminjam selama ada pengajuan yang diproses **atau** pinjaman yang belum lunas. Aturan ini dijaga di dua lapis:

```sql
-- cek di aplikasi sebelum insert
SELECT 1 FROM loan
WHERE user_id = :user_id
  AND status IN ('SUBMITTED', 'PROCESSING', 'APPROVED', 'ACTIVE');

-- pengaman terakhir di database: dua request bersamaan tidak bisa sama-sama lolos
CREATE UNIQUE INDEX uq_loan_one_open_per_user
  ON loan (user_id)
  WHERE status IN ('SUBMITTED', 'PROCESSING', 'APPROVED', 'ACTIVE');
```

**2. Syarat mengajukan pinjaman.** `USER.status = ACTIVE` dan `USER.kyc_status = VERIFIED`.

**3. Jadwal cicilan.** `REPAYMENT_SCHEDULE` dibuat saat `LOAN` menjadi `APPROVED`, sebanyak `tenor_months` baris. Dengan bunga flat: `interest_part = principal_amount × interest_rate_monthly`, `principal_part = principal_amount / tenor_months`.

**4. Lunas.** Saat semua cicilan berstatus `PAID`, `LOAN.status` menjadi `PAID_OFF` dan user bisa mengajukan lagi.

## Catatan Desain

- **`LOAN_DECISION` dipisah dari `LOAN`** (relasi 1 ke 0..1) supaya data hasil scoring, yaitu skor, reason code, dan versi model, tersimpan utuh untuk audit dan sengketa, tanpa membebani tabel transaksi utama. Waktu keputusan hanya disimpan di `LOAN_DECISION.decided_at`.
- **`DEVICE` menyimpan public key biometric per perangkat**, bukan per user, karena satu user bisa punya lebih dari satu HP dan biometric terikat ke hardware.
- **`CONSENT` mencatat persetujuan** syarat dan ketentuan saat registrasi dan perjanjian pinjaman saat pengajuan, lengkap dengan versi dokumen, waktu, IP, dan device. Ini bukti audit yang dibutuhkan untuk kepatuhan perlindungan data pribadi dan regulasi fintech lending.
- **NIK dienkripsi di level field.** `nik_hash` dipakai untuk cek duplikasi tanpa perlu mendekripsi.
- **Foto KTP dan selfie** disimpan di object storage. Database hanya menyimpan `file_key`, dan akses file memakai signed URL berumur pendek.
- **Pembayaran cicilan** berada di luar scope (lihat README). Diasumsikan sistem pembayaran eksternal mengubah `REPAYMENT_SCHEDULE.status` menjadi `PAID` dan mengisi `paid_at`.
