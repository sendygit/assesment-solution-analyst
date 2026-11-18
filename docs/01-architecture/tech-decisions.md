# Tech Decisions

| Layer | Pilihan | Alasan |
|---|---|---|
| Mobile | Flutter / React Native | Satu codebase untuk Android & iOS, time-to-market lebih cepat untuk MVP fintech |
| API | REST (JSON over HTTPS) | Sederhana, mudah diintegrasikan, cukup untuk kebutuhan CRUD + scoring async |
| Backend | Monolith modular (Node.js/NestJS atau Java Spring Boot) | Development lebih cepat di tahap awal, modul tetap dipisah secara logis per domain |
| DB | PostgreSQL | Relational, mendukung transaksi ACID (penting untuk data finansial: saldo, tagihan) |
| Cache/Session | Redis | Session token, OTP dengan TTL, rate-limiting counter |
| Queue | RabbitMQ / Amazon SQS | Decouple proses scoring & notifikasi dari request utama agar tidak blocking |
| Object Storage | Amazon S3 / GCS | Penyimpanan file KTP & selfie, terenkripsi at-rest, lifecycle policy untuk retensi data |
| KYC/OCR | Privy / VIDA / Dukcapil API | Verifikasi KTP & liveness check, wajib untuk compliance KYC fintech |
| Notifikasi | SendGrid/SES (email), Twilio/Zenziva (SMS) | Provider umum, reliable delivery, mendukung template |
| Auth | JWT (access + refresh token), biometric via device secure storage | Stateless auth untuk skalabilitas, biometric key tidak pernah keluar device (sesuai best practice FIDO/local biometric) |

## Keamanan (ringkas)

- Data KTP & foto dienkripsi at-rest dan in-transit (TLS 1.2+).
- PII (NIK, foto) tidak di-log.
- Rate limiting & OTP untuk mencegah brute force di login/registrasi.
- Biometric login memverifikasi secara lokal di device (tidak mengirim data biometric ke server) — server hanya menerima token hasil verifikasi.
