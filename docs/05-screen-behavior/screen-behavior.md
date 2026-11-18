# Screen Behavior Detail

Mengacu pada [screen-flow.md](../02-screen-flow/screen-flow.md). Setiap screen dijabarkan: state, input dan validasi, behavior, error handling, serta endpoint yang dipakai.

## Traceability Screen ke API

| Screen | Endpoint |
|---|---|
| Landing | `POST /auth/refresh` (cek sesi saat app dibuka) |
| Register | `POST /auth/register` |
| Verifikasi OTP | `POST /auth/verify-otp`, `POST /auth/register` (kirim ulang) |
| Upload Selfie + KTP | `POST /users/me/documents` |
| Verifikasi KYC | `GET /users/me/kyc-status` |
| Set Password | `POST /auth/set-password` |
| Registrasi Berhasil | `PUT /devices/{deviceId}/biometric` |
| Login | `POST /auth/login`, `POST /auth/biometric/challenge`, `POST /auth/login/biometric` |
| Lupa Password | `POST /auth/password/forgot`, `POST /auth/password/reset` |
| Home | `GET /loans/current` |
| Form Pengajuan | `POST /loans/simulate` |
| Review & Persetujuan | `POST /loans` |
| Status Pengajuan | `GET /loans/{loanId}` |
| Detail Pinjaman & Histori Cicilan | `GET /loans/{loanId}`, `GET /loans/{loanId}/installments`, `GET /loans` |
| Profile/Settings | `GET /users/me`, `PUT`/`DELETE /devices/{deviceId}/biometric`, `POST /auth/logout` |

## Behavior Global

| Kondisi | Behavior |
|---|---|
| Access token kedaluwarsa (401) | App memanggil `POST /auth/refresh` sekali lalu mengulang request. Jika refresh gagal, sesi dihapus dan user diarahkan ke Login dengan pesan "Sesi berakhir, silakan masuk lagi" |
| Tidak ada koneksi | Banner "Tidak ada koneksi". Tombol aksi nonaktif, data terakhir tetap ditampilkan jika ada |
| Error server (5xx) | Pesan generik "Terjadi gangguan, coba lagi" dengan tombol Coba Lagi. Detail teknis tidak ditampilkan |
| Error validasi (422) | Pesan dari `field_errors` ditampilkan inline di bawah field terkait |
| Tombol nonaktif | Selalu disertai teks bantu inline yang menjelaskan alasannya (bukan tooltip, karena tidak ada hover di mobile) |
| Screenshot di screen sensitif | Diblokir di screen yang menampilkan KTP, OTP, dan password |

## 1. Landing

| Aspek | Detail |
|---|---|
| State | `checking_session` → `show_options` |
| Behavior | Saat app dibuka, jika ada refresh token tersimpan app mencoba `POST /auth/refresh`. Berhasil langsung ke Home. Gagal atau tidak ada token, tampilkan tombol "Masuk" dan "Daftar" |
| Loading state | Splash dengan logo selama pengecekan sesi |

## 2. Register

| Aspek | Detail |
|---|---|
| State | `idle` → `submitting` → `success` / `error` |
| Input | Nama lengkap sesuai KTP, NIK, tanggal lahir, email, nomor telepon, pekerjaan, penghasilan per bulan, checkbox persetujuan Syarat & Ketentuan dan Kebijakan Privasi |
| Validasi | NIK 16 digit angka. Umur minimal sesuai kebijakan risiko (asumsi 21 tahun). Email format valid. Nomor telepon format Indonesia (+62/08). Penghasilan lebih dari 0. Checkbox wajib dicentang agar tombol "Lanjut" aktif |
| Behavior | Password belum diminta di sini. Password dibuat setelah identitas terverifikasi |
| Error handling | `409 PHONE_ALREADY_REGISTERED` atau email sudah dipakai: error inline dan tautan "Masuk". Jika nomor terdaftar tapi registrasi belum selesai, server mengirim ulang OTP dan app langsung ke screen OTP (resume) |
| Loading state | Tombol "Lanjut" nonaktif + spinner saat `submitting` |

## 3. Verifikasi OTP

| Aspek | Detail |
|---|---|
| State | `idle` → `submitting` → `success` / `error` / `locked` |
| Input | 6 digit angka, auto-submit saat lengkap, autofill OTP SMS di Android |
| Error handling | OTP salah: field bergetar + teks error, sisa percobaan ditampilkan. Setelah 3 kali salah, state `locked` selama 15 menit. OTP kedaluwarsa (5 menit): tombol "Kirim Ulang" aktif setelah hitung mundur 60 detik |
| Sukses | Onboarding token disimpan di secure storage, app lanjut ke langkah sesuai `next_step` dari server |

## 4. Upload Selfie + KTP

| Aspek | Detail |
|---|---|
| State | `empty` → `captured` → `uploading` → selesai (lanjut ke Verifikasi KYC) / `error` |
| Behavior | Kamera dengan bingkai panduan untuk KTP dan wajah. Foto selfie diambil dengan instruksi liveness (misalnya kedip atau menoleh) |
| Validasi | Format jpg/png, maksimal 5MB setelah kompresi. Deteksi blur dan pencahayaan di sisi app sebelum upload |
| Izin kamera ditolak | Penjelasan kenapa kamera dibutuhkan + tombol "Buka Pengaturan". Upload dari galeri tidak diizinkan untuk selfie (mencegah foto orang lain) |
| Error handling | Upload gagal: tombol Coba Lagi tanpa harus foto ulang. Onboarding token kedaluwarsa (401): user diminta verifikasi OTP ulang |
| Loading state | Progress bar upload |

## 5. Verifikasi KYC

| Aspek | Detail |
|---|---|
| State | `pending` → `verified` / `rejected` |
| Behavior | Animasi menunggu + teks "Memverifikasi identitasmu, biasanya kurang dari 2 menit". Polling `GET /users/me/kyc-status` tiap 5 detik. Setelah 3 menit polling berhenti dan muncul teks "Verifikasi masih berjalan, kamu bisa kembali lagi nanti" |
| `rejected` | Tampilkan alasan sesuai `reason_code` (contoh: "Foto KTP buram") dan tombol "Upload Ulang" |
| `verified` | Lanjut otomatis ke Set Password |
| Resume | Jika user menutup app lalu kembali, registrasi dilanjutkan lewat Register dengan nomor yang sama, OTP, lalu app membuka langkah terakhir berdasarkan `next_step` |

## 6. Set Password

| Aspek | Detail |
|---|---|
| State | `idle` → `submitting` → `success` / `error` |
| Input | Password dan konfirmasi password |
| Validasi | Minimal 8 karakter, kombinasi huruf dan angka, tidak sama dengan NIK atau tanggal lahir. Indikator kekuatan password real-time. Konfirmasi harus sama |
| Sukses | Server mengembalikan access dan refresh token, user langsung masuk tanpa login ulang |

## 7. Registrasi Berhasil

| Aspek | Detail |
|---|---|
| State | `shown` |
| Behavior | Ucapan selamat + ringkasan limit pinjaman (maks Rp 12.000.000, tenor maks 12 bulan) |
| Tawaran biometric | Jika device punya sidik jari/Face ID yang sudah terdaftar di OS, tampilkan "Aktifkan login biometric?" dengan pilihan Aktifkan / Nanti. Jika belum ada biometric di device, tawaran tidak ditampilkan |
| Aksi | Tombol "Mulai" ke Home |

## 8. Login

| Aspek | Detail |
|---|---|
| State | `idle` → `authenticating` → `success` / `error` / `locked` |
| Validasi | Field tidak boleh kosong |
| Behavior | Jika biometric aktif untuk device ini, prompt biometric muncul otomatis saat screen dibuka, dengan opsi "Pakai password" |
| Biometric berubah | Jika sidik jari/wajah di device ditambah atau dihapus, OS membatalkan key. App menonaktifkan biometric dan meminta login password, lalu menawarkan aktivasi ulang |
| Error handling | Kredensial salah: pesan generik "Email/nomor atau password salah" (mencegah user enumeration). 5 kali gagal: `locked` 15 menit. Biometric gagal 3 kali: otomatis beralih ke password |
| Tautan | "Lupa password?" ke screen Lupa Password |
| Loading state | Tombol nonaktif + spinner |

## 9. Lupa Password

| Aspek | Detail |
|---|---|
| State | `input_phone` → `input_otp_and_password` → `success` |
| Behavior | Setelah nomor dikirim, pesan selalu "Jika nomor terdaftar, OTP sudah dikirim" (tidak membocorkan apakah nomor terdaftar). Setelah reset, semua sesi lama dicabut dan user kembali ke Login |

## 10. Home

| Aspek | Detail |
|---|---|
| State | `loading` → `no_loan` / `processing` / `active` → `error` |
| `no_loan` | CTA "Ajukan Pinjaman" aktif |
| `processing` | Kartu status pengajuan (nominal, tenor, status "Sedang diproses" / "Disetujui, dana sedang dicairkan"). Tombol Ajukan nonaktif dengan teks bantu "Pengajuanmu sedang diproses". Tap kartu membuka Status Pengajuan |
| `active` | Kartu sisa hutang, tagihan bulan ini, dan tanggal jatuh tempo. Badge merah jika ada cicilan `OVERDUE`. Tombol Ajukan nonaktif dengan teks bantu "Lunasi pinjaman berjalan untuk mengajukan lagi". Tap kartu membuka Detail Pinjaman |
| Error handling | Gagal memuat: skeleton diganti pesan + tombol Coba Lagi. Tombol Ajukan tetap nonaktif selama status belum diketahui |
| Refresh | Pull to refresh, dan otomatis memuat ulang setiap kali app kembali ke foreground |

## 11. Form Pengajuan Pinjaman

| Aspek | Detail |
|---|---|
| State | `idle` → `simulating` → `ready` → `error` |
| Input | Nominal (slider + input, kelipatan Rp 100.000, Rp 500.000 sampai Rp 12.000.000) dan tenor (pilihan 1 sampai 12 bulan) |
| Behavior | Setiap perubahan input memanggil `POST /loans/simulate` (debounce 500 ms) untuk menampilkan cicilan per bulan, bunga, dan total pengembalian. Angka selalu dari server |
| Validasi | Nominal dan tenor dalam batas. Tombol "Lanjut" nonaktif selama simulasi belum selesai |
| Guard | Screen ini hanya bisa dibuka dari Home varian `no_loan`. Jika dibuka lewat deep link saat `can_apply = false`, app mengarahkan kembali ke Home |

## 12. Review & Persetujuan

| Aspek | Detail |
|---|---|
| State | `idle` → `submitting` → `submitted` / `error` |
| Behavior | Ringkasan hasil simulasi + tautan dokumen perjanjian pinjaman. Checkbox persetujuan wajib dicentang agar tombol "Ajukan" aktif. Sebelum submit, user konfirmasi dengan biometric atau password |
| Pengajuan ganda | App membuat `Idempotency-Key` sekali per screen ini, sehingga tap ganda atau retry karena jaringan tidak membuat dua pengajuan |
| Error handling | `409 LOAN_ALREADY_OPEN` (misalnya sudah mengajukan dari device lain): pesan + arahkan ke Home. `403 KYC_NOT_VERIFIED`: arahkan ke status KYC |

## 13. Status Pengajuan

| Aspek | Detail |
|---|---|
| State | `submitted` / `processing` → `approved` / `rejected` |
| Behavior | Timeline status: Diajukan, Diproses, Hasil. Teks "Kamu akan mendapat notifikasi saat hasil keluar". User boleh keluar dari screen ini |
| `approved` | Ringkasan pinjaman dan jadwal cicilan pertama, info bahwa dana sedang dicairkan |
| `rejected` | Pesan penolakan yang sopan tanpa membuka detail skor, dan tombol kembali ke Home |
| Notifikasi | Hasil juga dikirim via email dan SMS, jadi user tetap tahu meski sudah menutup app |

## 14. Detail Pinjaman & Histori Cicilan

| Aspek | Detail |
|---|---|
| State | `loading` → `loaded` → `error` |
| Behavior | Info pinjaman (pokok, bunga, tenor, total) dan sisa hutang. Daftar cicilan dengan status Akan Datang / Jatuh Tempo / Lunas / Terlambat. Bagian riwayat pinjaman sebelumnya (`GET /loans`, dengan pagination) |
| Empty state | Belum ada riwayat pinjaman: ilustrasi + teks "Belum ada riwayat pinjaman" |

## 15. Profile / Settings

| Aspek | Detail |
|---|---|
| State | `loaded` |
| Behavior | Lihat data diri. Nama, NIK, dan tanggal lahir terkunci setelah KYC terverifikasi. Toggle login biometric (aktivasi meminta konfirmasi password). Logout |
| Toggle biometric | Jika device tidak punya biometric terdaftar, toggle nonaktif dengan teks "Atur sidik jari/Face ID di pengaturan HP terlebih dahulu" |
| Logout | Konfirmasi, panggil `POST /auth/logout`, hapus token lokal, kembali ke Login |
