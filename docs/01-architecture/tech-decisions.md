# Tech Decisions

Satu pilihan per komponen, beserta alasan dan konsekuensinya.

| Layer | Pilihan | Alasan | Trade-off |
|---|---|---|---|
| Mobile | **Flutter** | Satu codebase untuk Android dan iOS, performa UI mendekati native, plugin matang untuk kamera, biometric (`local_auth`), dan secure storage | Ukuran app lebih besar. Fitur OS yang sangat baru kadang perlu kode native (platform channel) |
| Backend | **NestJS (TypeScript)**, monolith modular | Konsep Module bawaan cocok untuk memisahkan Auth, User, Document, Loan, Repayment, Notification. Validasi DTO dan dependency injection built-in. SDK provider KYC dan pembayaran lokal umumnya tersedia untuk Node.js | Performa per instance di bawah Go/Java untuk beban CPU berat. Diterima karena beban utama I/O (DB, provider pihak ketiga), bukan komputasi |
| Database | **PostgreSQL** | Transaksi ACID untuk data finansial, partial unique index untuk aturan "satu pinjaman berjalan", tipe `numeric` untuk uang | Skala tulis horizontal lebih sulit dibanding NoSQL. Belum relevan di skala awal |
| Cache | **Redis** | OTP dan challenge biometric dengan TTL, counter rate limit | Data di Redis tidak permanen, jadi hanya untuk data sementara |
| Queue | **Amazon SQS** + dead-letter queue | Managed, tidak perlu mengoperasikan broker sendiri, retry dan DLQ bawaan | Tidak ada routing kompleks seperti RabbitMQ. Cukup untuk dua jenis job (scoring, notifikasi) |
| Object storage | **Amazon S3** | Enkripsi at-rest (SSE-KMS), signed URL berumur pendek, lifecycle untuk retensi data | - |
| KYC | **VIDA** | OCR KTP, liveness check, face match, dan pencocokan data ke Dukcapil dalam satu integrasi | Ketergantungan pada satu vendor. Dimitigasi dengan interface `KycProvider` agar vendor bisa diganti |
| Email | **Amazon SES** | Satu ekosistem dengan AWS, murah, mendukung template | Perlu setup reputasi domain (SPF, DKIM) |
| SMS | **SMS gateway lokal** dengan kanal OTP khusus | Delivery rate ke operator Indonesia lebih baik dibanding provider global | Vendor dipilih saat procurement berdasarkan delivery rate dan harga |
| Auth | **JWT access token (15 menit) + refresh token di DB** | Access token stateless, diverifikasi di gateway tanpa query DB. Refresh token di tabel `AUTH_SESSION` agar sesi bisa dicabut per device | Access token yang sudah terbit tetap berlaku sampai kedaluwarsa (maks 15 menit) |
| Biometric | **Keypair di Secure Enclave / Android Keystore** | Data biometric tidak pernah keluar device. Server hanya memverifikasi signature atas challenge sekali pakai | Login biometric hanya berlaku di device yang didaftarkan |

**Kenapa bukan Go untuk backend?** Go unggul untuk concurrency dan throughput. Jika nanti beban worker (scoring, notifikasi) menjadi besar, worker bisa dipisah dan ditulis ulang dengan Go tanpa mengubah API, karena worker berkomunikasi lewat queue.

## Keamanan

- TLS untuk semua koneksi. Data KTP, foto, dan NIK dienkripsi saat disimpan, dan tidak ditulis ke log.
- Rate limiting dan lockout untuk OTP dan login.
- Endpoint `/users/me/...` mengambil user dari token, bukan dari parameter, agar user tidak bisa mengakses data orang lain.
- Biometric diverifikasi lokal oleh OS. Server hanya menerima signature atas challenge sekali pakai, sehingga tidak bisa di-replay.
- Webhook dari provider KYC diverifikasi dengan signature.
