# Loan State Diagram & Aturan Pengajuan

## 1. State Diagram LOAN

```mermaid
stateDiagram-v2
    [*] --> SUBMITTED: POST /loans valid
    SUBMITTED --> PROCESSING: worker mengambil event dari queue
    PROCESSING --> APPROVED: skor lolos
    PROCESSING --> REJECTED: skor tidak lolos
    PROCESSING --> SUBMITTED: scoring engine error, retry
    APPROVED --> ACTIVE: dana dicairkan (disbursed_at terisi)
    ACTIVE --> PAID_OFF: semua cicilan PAID
    REJECTED --> [*]
    PAID_OFF --> [*]

    note right of SUBMITTED
        SUBMITTED, PROCESSING, APPROVED, ACTIVE
        = pinjaman berjalan.
        User tidak bisa mengajukan pinjaman baru.
    end note
```

| Transisi | Pemicu | Efek samping |
|---|---|---|
| `[*] → SUBMITTED` | `POST /loans` lolos validasi | Simpan `LOAN` + `CONSENT` (perjanjian pinjaman), publish event `LoanSubmitted` |
| `SUBMITTED → PROCESSING` | Worker mengambil event | Panggil credit scoring engine |
| `PROCESSING → APPROVED` | Hasil scoring lolos | Simpan `LOAN_DECISION`, buat `REPAYMENT_SCHEDULE`, kirim notifikasi disetujui (email, SMS) |
| `PROCESSING → REJECTED` | Hasil scoring tidak lolos | Simpan `LOAN_DECISION`, kirim notifikasi ditolak |
| `PROCESSING → SUBMITTED` | Scoring engine timeout/error | Retry dengan backoff, maksimal 3 kali, lalu masuk dead-letter queue untuk ditangani tim operasional |
| `APPROVED → ACTIVE` | Pencairan dana berhasil | Isi `disbursed_at`, tanggal jatuh tempo cicilan dihitung dari tanggal ini |
| `ACTIVE → PAID_OFF` | Cicilan terakhir `PAID` | Isi `paid_off_at`, user bisa mengajukan lagi |

## 2. State Diagram Cicilan (REPAYMENT_SCHEDULE)

```mermaid
stateDiagram-v2
    [*] --> UPCOMING: jadwal dibuat
    UPCOMING --> DUE: scheduler, H-0 jatuh tempo
    UPCOMING --> PAID: dibayar lebih awal
    DUE --> PAID: pembayaran diterima
    DUE --> OVERDUE: scheduler, lewat jatuh tempo
    OVERDUE --> PAID: pembayaran diterima
    PAID --> [*]
```

Perubahan status `DUE` dan `OVERDUE` dilakukan oleh scheduler yang berjalan harian. Scheduler yang sama mengirim reminder H-3 dan H-1. Perubahan ke `PAID` datang dari sistem pembayaran eksternal (di luar scope).

## 3. Flowchart Validasi `POST /loans`

```mermaid
flowchart TD
    A[Request POST /loans] --> B{Access token valid?}
    B -->|Tidak| E401[401 Unauthorized]
    B -->|Ya| C{Idempotency-Key<br/>pernah dipakai?}
    C -->|Ya| R[Kembalikan respons<br/>pengajuan sebelumnya]
    C -->|Tidak| D{USER ACTIVE dan<br/>KYC VERIFIED?}
    D -->|Tidak| E403[403 KYC_NOT_VERIFIED]
    D -->|Ya| F{Nominal 500 rb - 12 jt<br/>dan tenor 1-12 bulan?}
    F -->|Tidak| E422[422 VALIDATION_ERROR]
    F -->|Ya| G{Ada pinjaman SUBMITTED/<br/>PROCESSING/APPROVED/ACTIVE?}
    G -->|Ya| E409[409 LOAN_ALREADY_OPEN]
    G -->|Tidak| H[Hitung bunga dan<br/>cicilan di server]
    H --> I[INSERT LOAN + CONSENT<br/>dalam satu transaksi]
    I --> J{Unique index<br/>dilanggar?}
    J -->|Ya, request paralel| E409
    J -->|Tidak| K[Publish LoanSubmitted ke queue]
    K --> L[202 Accepted<br/>status SUBMITTED]
```

Cek pinjaman berjalan dilakukan dua kali dengan sengaja: query di aplikasi memberi pesan error yang jelas, sedangkan partial unique index menjamin aturan tetap benar ketika dua request datang bersamaan, misalnya dari dua device.

## 4. Rumus Cicilan (Asumsi)

Bunga **flat** per bulan, nilainya dikonfigurasi oleh bisnis dan wajib mengikuti batas manfaat ekonomi dari OJK.

| Komponen | Rumus | Contoh: Rp 6.000.000, 6 bulan, bunga 1,5% |
|---|---|---|
| Bunga per bulan | pokok × bunga | Rp 90.000 |
| Pokok per bulan | pokok ÷ tenor | Rp 1.000.000 |
| Cicilan per bulan | pokok per bulan + bunga per bulan | Rp 1.090.000 |
| Total pengembalian | cicilan × tenor | Rp 6.540.000 |

Semua angka dihitung di server (`POST /loans/simulate`) supaya app dan backend tidak pernah berbeda hasil.
