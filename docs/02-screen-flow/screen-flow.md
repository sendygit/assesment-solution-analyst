# Screen Flow

```mermaid
flowchart TD
    Start([App Start]) --> HasSession{Ada session<br/>aktif?}
    HasSession -->|Ya| Home
    HasSession -->|Tidak| Landing[Landing / Splash]

    Landing --> Login[Screen: Login]
    Landing --> Register[Screen: Register]

    Register --> RegForm[Isi Data Diri<br/>nama, email, no. telp]
    RegForm --> OTP[Verifikasi OTP<br/>email/SMS]
    OTP --> UploadDoc[Upload Selfie + Foto KTP]
    UploadDoc --> KYCCheck{KYC/OCR<br/>Verifikasi}
    KYCCheck -->|Gagal, upload ulang| UploadDoc
    KYCCheck -->|Sukses| SetPassword[Set Password]
    SetPassword --> RegSuccess[Registrasi Berhasil]
    RegSuccess --> Login

    Login --> LoginMethod{Metode Login}
    LoginMethod -->|Password| LoginPwd[Input Email/No HP + Password]
    LoginMethod -->|Biometric| LoginBio[Scan Fingerprint/Face]
    LoginPwd --> AuthCheck{Valid?}
    LoginBio --> AuthCheck
    AuthCheck -->|Tidak| Login
    AuthCheck -->|Ya| Home[Screen: Home]

    Home --> LoanStatusCheck{Ada pinjaman<br/>aktif?}
    LoanStatusCheck -->|Ya| HomeActive[Tampilkan:<br/>Sisa Hutang & Tagihan Bulan Ini]
    LoanStatusCheck -->|Tidak| HomeEmpty[Tampilkan:<br/>CTA Ajukan Pinjaman]

    HomeActive --> Detail[Screen: Detail Pinjaman & Histori Bayar]

    HomeEmpty --> LoanForm[Screen: Form Pengajuan Pinjaman<br/>nominal max 12jt, tenor max 12 bulan]
    LoanForm --> LoanReview[Screen: Review & Konfirmasi]
    LoanReview --> Submit[Submit Pengajuan]
    Submit --> Processing[Screen: Status Diproses]
    Processing --> Decision{Hasil Scoring}
    Decision -->|Diterima| Approved[Notifikasi: Pinjaman Disetujui<br/>Email + SMS]
    Decision -->|Ditolak| Rejected[Notifikasi: Pinjaman Ditolak<br/>Email + SMS]
    Approved --> Home
    Rejected --> Home

    Home --> Profile[Screen: Profile/Settings]
    Profile --> Logout[Logout]
    Logout --> Login
```

## Catatan Flow

- **Blocked state**: tombol "Ajukan Pinjaman" di screen `HomeActive` otomatis disabled (dengan tooltip "Lunasi dulu pinjaman berjalan") selama `LoanStatusCheck` bernilai aktif (ada pinjaman belum lunas), sesuai requirement "User tidak dapat melakukan peminjaman jika sedang ada proses peminjaman dan belum dilunaskan". Ini state di dalam screen Home, bukan screen terpisah.
- **Processing screen**: karena hasil scoring bisa async (lihat [architecture](../01-architecture/high-level-architecture.md)), screen ini bisa menunggu dengan polling/push, atau user boleh keluar dan menerima notifikasi push/email/SMS saat hasil sudah ada.
- **Urutan OTP sebelum upload KTP**: OTP memverifikasi kepemilikan email/nomor telepon terlebih dulu, baru lanjut upload dokumen identitas. Dengan urutan ini, jika KYC/OCR gagal (foto blur, data tidak cocok, dll), user cukup diarahkan ulang ke `UploadDoc` → `KYCCheck` tanpa perlu mengulang verifikasi OTP yang sudah berhasil.
