# Pinjaman Online App — Solution Design (PT. XYZ)

Dokumen desain solusi untuk aplikasi mobile pinjaman online PT. XYZ. Disusun berdasarkan user story yang diberikan di assessment, mencakup arsitektur, screen flow, ERD, API design, dan screen behavior.

## Ringkasan Kebutuhan

- Registrasi dengan data diri, email, nomor telepon, upload foto + KTP
- Login dengan password atau biometric
- Lihat sisa hutang & tagihan bulanan (jika ada)
- Pengajuan pinjaman maksimal Rp 12.000.000, tenor maksimal 12 bulan
- Proses pengajuan menghasilkan status diterima/ditolak
- Notifikasi via email & SMS saat pinjaman disetujui
- User tidak bisa mengajukan pinjaman baru jika masih ada pinjaman aktif yang belum lunas

## Daftar Dokumen

| # | Dokumen | Isi |
|---|---------|-----|
| 1 | [docs/01-architecture/high-level-architecture.md](docs/01-architecture/high-level-architecture.md) | High-level architecture diagram & komponen sistem |
| 1b | [docs/01-architecture/tech-decisions.md](docs/01-architecture/tech-decisions.md) | Pilihan teknologi & justifikasi |
| 2a | [docs/02-screen-flow/screen-flow.md](docs/02-screen-flow/screen-flow.md) | Screen flow aplikasi mobile |
| 2b | [docs/03-erd/erd.md](docs/03-erd/erd.md) | Entity Relationship Diagram |
| 3a | [docs/04-api-design/api-sequence-diagrams.md](docs/04-api-design/api-sequence-diagrams.md) | Sequence diagram per use case utama |
| 3b | [docs/04-api-design/api-spec.yaml](docs/04-api-design/api-spec.yaml) | OpenAPI 3.0 spec seluruh endpoint |
| 4 | [docs/05-screen-behavior/screen-behavior.md](docs/05-screen-behavior/screen-behavior.md) | Detail behavior per screen (state, validasi, error handling) |

## Cara Membaca

Semua diagram ditulis dalam format [Mermaid](https://mermaid.js.org/) sehingga otomatis ter-render saat dibuka di GitHub — tidak perlu tool tambahan. `api-spec.yaml` adalah spesifikasi OpenAPI 3.0 yang bisa di-import langsung ke Swagger Editor / Postman untuk eksplorasi interaktif.

## Asumsi Desain

- Arsitektur backend: **monolith modular** (bukan microservices) — cukup untuk skala fintech tahap awal, modul terpisah secara logis (Auth, User, Loan, Notification, Document) agar mudah dipecah jadi service terpisah di kemudian hari.
- Proses persetujuan pinjaman diasumsikan melalui **credit scoring engine** otomatis (rule-based/3rd party), bukan manual review — mengacu pada user story "akan diproses dengan hasil diterima/ditolak" tanpa menyebut approval manual.
- Verifikasi KTP menggunakan **OCR + liveness check** pihak ketiga (misal: Privy/VIDA/Dukcapil API) untuk validasi identitas sebelum akun aktif.
