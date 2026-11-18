# Pinjaman Online App — Solution Design (PT. XYZ)

Dokumen desain solusi untuk aplikasi mobile pinjaman online PT. XYZ. Disusun berdasarkan user story yang diberikan di assessment, mencakup arsitektur, screen flow, ERD, API design, dan screen behavior.

## Ringkasan Kebutuhan

- Registrasi dengan data diri, email, nomor telepon, upload foto + KTP
- Login dengan password atau biometric (jika tersedia di perangkat)
- Lihat sisa hutang & tagihan bulanan (jika ada)
- Pengajuan pinjaman maksimal Rp 12.000.000, tenor maksimal 12 bulan
- Proses pengajuan menghasilkan status diterima/ditolak
- Notifikasi via email & SMS ke kontak terdaftar saat pinjaman diterima
- User tidak bisa mengajukan pinjaman baru selama ada pengajuan yang sedang diproses atau pinjaman yang belum lunas

## Daftar Dokumen

| Tugas | Dokumen | Isi |
|---|---|---|
| 1 | [high-level-architecture.md](docs/01-architecture/high-level-architecture.md) | Diagram arsitektur, komponen, dan alasan desain |
| 1 | [tech-decisions.md](docs/01-architecture/tech-decisions.md) | Pilihan teknologi, trade-off, dan keamanan |
| 2 | [screen-flow.md](docs/02-screen-flow/screen-flow.md) | Screen flow aplikasi mobile |
| 2 | [erd.md](docs/03-erd/erd.md) | Entity Relationship Diagram dan aturan bisnis di level data |
| 3 | [api-sequence-diagrams.md](docs/04-api-design/api-sequence-diagrams.md) | Sequence diagram untuk registrasi, login, biometric, pengajuan, dan Home |
| 3 | [loan-state-and-rules.md](docs/04-api-design/loan-state-and-rules.md) | State diagram pinjaman dan cicilan, flowchart validasi pengajuan, rumus cicilan |
| 3 | [api-spec.yaml](docs/04-api-design/api-spec.yaml) | Spesifikasi OpenAPI 3.0 seluruh endpoint |
| 4 | [screen-behavior.md](docs/05-screen-behavior/screen-behavior.md) | Behavior per screen dan pemetaan screen ke endpoint |

## Cara Membaca

Semua diagram ditulis dalam format [Mermaid](https://mermaid.js.org/) sehingga otomatis ter-render saat dibuka di GitHub. `api-spec.yaml` bisa dibuka di [Swagger Editor](https://editor.swagger.io) atau di-import ke Postman.

## Asumsi Desain

- **Monolith modular** (NestJS) dengan worker terpisah untuk proses asinkron. Cukup untuk skala awal, dan modul bisa dipecah menjadi service sendiri saat dibutuhkan.
- **Persetujuan pinjaman otomatis** lewat credit scoring engine, sesuai user story "diproses dengan hasil diterima atau ditolak" yang tidak menyebut review manual.
- **Data diri** yang dikumpulkan saat registrasi: nama, NIK, tanggal lahir, pekerjaan, dan penghasilan. Data ini dibutuhkan untuk KYC dan credit scoring.
- **KYC lewat provider pihak ketiga** (OCR KTP, liveness, pencocokan data Dukcapil). Jika ditolak, user upload ulang dokumen.
- **Bunga flat per bulan.** Nilainya dikonfigurasi bisnis dan mengikuti ketentuan OJK. Rumus ada di [loan-state-and-rules.md](docs/04-api-design/loan-state-and-rules.md).
- **"Pinjaman berjalan"** mencakup status `SUBMITTED`, `PROCESSING`, `APPROVED`, dan `ACTIVE`, sesuai brief "sedang ada proses peminjaman dan belum dilunaskan".
- **Notifikasi email dan SMS** dikirim juga untuk pengajuan yang ditolak, karena user perlu tahu hasilnya meski brief hanya menyebut pengajuan yang diterima.

## Di Luar Scope

Hal berikut dibutuhkan agar produk bisa berjalan penuh, tetapi tidak diminta di brief. Desain ini menyediakan titik sambungnya saja:

- **Pencairan dana.** Status `APPROVED` berubah menjadi `ACTIVE` saat dana cair (`LOAN.disbursed_at`). Data rekening tujuan dan integrasi payment gateway untuk pencairan belum didesain.
- **Pembayaran cicilan.** Diasumsikan user membayar lewat virtual account, dan sistem pembayaran mengubah status cicilan menjadi `PAID`.
