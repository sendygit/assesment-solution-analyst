# Screen Flow

```mermaid
flowchart TD
    Start([App Start]) --> HasSession{Refresh token<br/>valid?}
    HasSession -->|Ya| Home
    HasSession -->|Tidak| Landing[Screen: Landing]

    Landing --> Login[Screen: Login]
    Landing --> Register[Screen: Register<br/>data diri + persetujuan T&C]

    Register --> OTP[Screen: Verifikasi OTP]
    OTP --> UploadDoc[Screen: Upload Selfie + KTP]
    UploadDoc --> Verifying[Screen: Verifikasi KYC<br/>menunggu hasil]
    Verifying --> KYCCheck{Hasil KYC}
    KYCCheck -->|Ditolak, upload ulang| UploadDoc
    KYCCheck -->|Terverifikasi| SetPassword[Screen: Set Password]
    SetPassword --> RegSuccess[Screen: Registrasi Berhasil<br/>tawaran aktifkan biometric]
    RegSuccess --> Home

    Login --> LoginMethod{Metode Login}
    LoginMethod -->|Password| LoginPwd[Input Email/No HP + Password]
    LoginMethod -->|Biometric| LoginBio[Prompt Sidik Jari / Face ID]
    LoginPwd --> AuthCheck{Valid?}
    LoginBio --> AuthCheck
    AuthCheck -->|Tidak| Login
    AuthCheck -->|Ya| Home[Screen: Home]
    Login --> Forgot[Screen: Lupa Password<br/>OTP + password baru]
    Forgot --> Login

    Home --> LoanStatus{GET /loans/current}
    LoanStatus -->|Tidak ada pinjaman| HomeEmpty[Home: Belum ada pinjaman<br/>tombol Ajukan aktif]
    LoanStatus -->|SUBMITTED / PROCESSING / APPROVED| HomeProcessing[Home: Pengajuan diproses<br/>tombol Ajukan nonaktif]
    LoanStatus -->|ACTIVE| HomeActive[Home: Sisa hutang & tagihan bulan ini<br/>tombol Ajukan nonaktif]

    HomeActive --> Detail[Screen: Detail Pinjaman & Histori Cicilan]
    HomeProcessing --> Processing

    HomeEmpty --> LoanForm[Screen: Form Pengajuan<br/>maks Rp 12 jt, tenor maks 12 bulan]
    LoanForm --> LoanReview[Screen: Review & Persetujuan]
    LoanReview --> Processing[Screen: Status Pengajuan]
    Processing --> Decision{Hasil Scoring}
    Decision -->|Disetujui| Approved[Hasil: Disetujui<br/>notifikasi email & SMS]
    Decision -->|Ditolak| Rejected[Hasil: Ditolak<br/>notifikasi email & SMS]
    Approved --> Home
    Rejected --> Home

    Home --> Profile[Screen: Profile/Settings]
    Profile --> Logout[Logout]
    Logout --> Login
```

## Catatan Flow

- **Aturan blokir pinjaman.** Brief menyebut user tidak bisa meminjam jika "sedang ada proses peminjaman dan belum dilunaskan". Karena itu Home punya tiga varian, dan tombol Ajukan nonaktif di dua di antaranya:

  | Varian Home | Status LOAN | Tombol Ajukan | Teks bantu di bawah tombol |
  |---|---|---|---|
  | Belum ada pinjaman | tidak ada, `REJECTED`, atau `PAID_OFF` | Aktif | - |
  | Pengajuan diproses | `SUBMITTED`, `PROCESSING`, `APPROVED` | Nonaktif | "Pengajuanmu sedang diproses" |
  | Pinjaman aktif | `ACTIVE` | Nonaktif | "Lunasi pinjaman berjalan untuk mengajukan lagi" |

  Status ini dibaca dari `GET /loans/current` (field `can_apply` dan `blocked_reason`). Server tetap menolak dengan `409` jika app mengirim pengajuan saat tidak boleh, misalnya dari device lain.
- **Urutan registrasi.** Data diri, lalu OTP, upload dokumen, KYC, dan set password. OTP diverifikasi lebih dulu untuk memastikan nomor milik user sebelum menerima data KTP. Setelah OTP valid, server memberi onboarding token, sehingga jika KYC gagal user cukup upload ulang tanpa mengulang OTP.
- **KYC asinkron.** Verifikasi bisa butuh waktu. Screen Verifikasi KYC menunggu dengan polling. Jika user menutup app, registrasi bisa dilanjutkan dari langkah terakhir (lihat [screen behavior](../05-screen-behavior/screen-behavior.md)).
- **Status Pengajuan.** Scoring berjalan asinkron (lihat [arsitektur](../01-architecture/high-level-architecture.md)). User boleh keluar dari screen ini dan akan menerima notifikasi email dan SMS saat hasil keluar.
