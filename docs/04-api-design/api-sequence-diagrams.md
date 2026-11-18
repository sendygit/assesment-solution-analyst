# API Sequence Diagrams

Spesifikasi lengkap tiap endpoint ada di [api-spec.yaml](api-spec.yaml). State dan aturan pengajuan pinjaman ada di [loan-state-and-rules.md](loan-state-and-rules.md).

## 1. Registrasi: Data Diri → OTP → Upload KTP + KYC → Set Password

```mermaid
sequenceDiagram
    actor U as User (Mobile App)
    participant GW as API Gateway
    participant Auth as Auth Module
    participant Doc as Document Module
    participant Blob as Object Storage
    participant KYC as KYC Provider
    participant DB as Database
    participant Cache as Redis
    participant SMS as SMS Provider

    U->>GW: POST /auth/register {data diri, consent_version}
    GW->>Auth: forward
    Auth->>DB: create USER (status=ONBOARDING, kyc=NOT_SUBMITTED) + CONSENT
    Auth->>Cache: simpan OTP (TTL 5 menit)
    Auth->>SMS: kirim OTP
    Auth-->>U: 201 {user_id}

    U->>GW: POST /auth/verify-otp {user_id, otp}
    GW->>Auth: forward
    Auth->>Cache: cocokkan OTP (maks 3 percobaan)
    Auth->>DB: phone_verified = true
    Auth-->>U: 200 {onboarding_token, next_step=UPLOAD_DOCUMENTS}

    U->>GW: POST /users/me/documents (selfie, ktp) + onboarding token
    GW->>Doc: forward (user_id diambil dari token)
    Doc->>Blob: simpan file (terenkripsi)
    Doc->>DB: DOCUMENT (PENDING), kyc_status = PENDING
    Doc->>KYC: submit OCR + liveness + pencocokan data Dukcapil
    Doc-->>U: 202 {kyc_status=PENDING}

    loop polling tiap 5 detik
        U->>GW: GET /users/me/kyc-status
        GW-->>U: {kyc_status}
    end

    KYC->>GW: POST /webhooks/kyc {provider_ref, result} + X-Signature
    GW->>Doc: forward (signature diverifikasi)
    Doc->>DB: update DOCUMENT & USER.kyc_status

    alt VERIFIED
        U->>GW: POST /auth/set-password {password} + onboarding token
        GW->>Auth: forward
        Auth->>DB: password_hash, status = ACTIVE, buat AUTH_SESSION
        Auth-->>U: 200 {access_token, refresh_token}
    else REJECTED
        Note over U: kembali ke screen upload dengan alasan penolakan
    end
```

## 2. Login Password, Refresh, dan Logout

```mermaid
sequenceDiagram
    actor U as User
    participant GW as API Gateway
    participant Auth as Auth Module
    participant Cache as Redis
    participant DB as Database

    U->>GW: POST /auth/login {identifier, password, device_id}
    GW->>Auth: forward
    Auth->>Cache: cek rate limit (maks 5 gagal / 15 menit)
    Auth->>DB: verifikasi password_hash
    Auth->>DB: buat AUTH_SESSION (refresh_token_hash, device)
    Auth-->>U: 200 {access_token 15 menit, refresh_token}

    Note over U,Auth: access token kedaluwarsa, API membalas 401
    U->>GW: POST /auth/refresh {refresh_token}
    GW->>Auth: forward
    Auth->>DB: validasi hash, rotasi refresh token
    Auth-->>U: 200 {access_token, refresh_token baru}

    U->>GW: POST /auth/logout
    GW->>Auth: forward
    Auth->>DB: AUTH_SESSION.revoked_at = now
    Auth-->>U: 204
```

Access token berupa JWT stateless dan diverifikasi oleh API Gateway. Refresh token disimpan ter-hash di `AUTH_SESSION` supaya sesi bisa dicabut per device. Redis hanya dipakai untuk OTP, challenge biometric, dan rate limit.

## 3. Aktivasi dan Login Biometric

```mermaid
sequenceDiagram
    actor U as User
    participant App as Mobile App (Secure Enclave/Keystore)
    participant GW as API Gateway
    participant Auth as Auth Module
    participant Cache as Redis
    participant DB as Database

    Note over U,DB: Aktivasi (sudah login, dari Registrasi Berhasil atau Profile)
    U->>App: aktifkan biometric, konfirmasi password
    App->>App: buat keypair, private key tidak pernah keluar device
    App->>GW: PUT /devices/{deviceId}/biometric {public_key, platform, password}
    GW->>Auth: forward
    Auth->>DB: simpan DEVICE + biometric_public_key
    Auth-->>App: 204

    Note over U,DB: Login berikutnya
    App->>GW: POST /auth/biometric/challenge {device_id}
    GW->>Auth: forward
    Auth->>Cache: simpan nonce (TTL 60 detik, sekali pakai)
    Auth-->>App: {challenge_id, challenge}
    App->>U: prompt sidik jari / Face ID
    U-->>App: biometric cocok (diverifikasi OS)
    App->>App: tanda tangani challenge dengan private key
    App->>GW: POST /auth/login/biometric {device_id, challenge_id, signature}
    GW->>Auth: forward
    Auth->>Cache: ambil dan hapus nonce (cegah replay)
    Auth->>DB: verifikasi signature dengan public key
    Auth->>DB: buat AUTH_SESSION (login_method=BIOMETRIC)
    Auth-->>App: 200 {access_token, refresh_token}
```

## 4. Pengajuan Pinjaman

```mermaid
sequenceDiagram
    actor U as User
    participant GW as API Gateway
    participant Loan as Loan Module
    participant DB as Database
    participant Q as Message Queue
    participant W as Worker
    participant Score as Credit Scoring Engine
    participant Notif as Notification Module

    U->>GW: POST /loans/simulate {amount, tenor}
    GW->>Loan: forward
    Loan-->>U: 200 {cicilan per bulan, bunga, total}

    U->>GW: POST /loans {amount, tenor, agreement_version} + Idempotency-Key
    GW->>Loan: forward
    Loan->>DB: cek idempotency key, status user, KYC
    Loan->>DB: cek pinjaman SUBMITTED/PROCESSING/APPROVED/ACTIVE
    alt Masih ada pinjaman berjalan
        Loan-->>U: 409 LOAN_ALREADY_OPEN
    else Tidak ada
        Loan->>DB: INSERT LOAN (SUBMITTED) + CONSENT (unique index menjaga race)
        Loan->>Q: publish LoanSubmitted
        Loan-->>U: 202 {loan_id, status=SUBMITTED}

        Q->>W: consume event
        W->>DB: LOAN.status = PROCESSING
        W->>Score: minta skor (data profil + riwayat)
        Score-->>W: {result, score, reason_code}
        W->>DB: simpan LOAN_DECISION, update LOAN.status
        alt APPROVED
            W->>DB: buat REPAYMENT_SCHEDULE
        end
        W->>Notif: trigger LOAN_APPROVED / LOAN_REJECTED
        Notif->>U: email + SMS
    end
```

## 5. Home: Status Pinjaman, Sisa Hutang, dan Tagihan

```mermaid
sequenceDiagram
    actor U as User
    participant GW as API Gateway
    participant Loan as Loan Module
    participant DB as Database

    U->>GW: GET /loans/current
    GW->>Loan: forward
    Loan->>DB: pinjaman berstatus SUBMITTED/PROCESSING/APPROVED/ACTIVE
    Loan->>DB: jika ACTIVE, hitung sisa hutang dan cicilan berikutnya
    Loan-->>U: 200 {can_apply, blocked_reason, loan, remaining_balance, next_installment}

    U->>GW: GET /loans/{loanId}/installments
    GW->>Loan: forward
    Loan-->>U: 200 [daftar cicilan + status]
```

## 6. Reminder dan Status Jatuh Tempo

```mermaid
sequenceDiagram
    participant S as Reminder Scheduler (cron harian)
    participant DB as Database
    participant Notif as Notification Module
    participant Ext as Email / SMS Provider
    actor U as User

    S->>DB: UPCOMING yang jatuh tempo hari ini menjadi DUE
    S->>DB: DUE yang lewat jatuh tempo menjadi OVERDUE
    S->>DB: cari cicilan belum PAID dengan due_date H-3 / H-1
    DB-->>S: daftar cicilan + data user
    loop setiap cicilan
        S->>Notif: trigger PAYMENT_REMINDER
        Notif->>Ext: kirim
        Ext-->>U: pengingat tagihan
        Notif->>DB: simpan NOTIFICATION (SENT / FAILED)
    end
```
