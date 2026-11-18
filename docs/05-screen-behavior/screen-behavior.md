# Screen Behavior Detail

Mengacu pada [screen-flow.md](../02-screen-flow/screen-flow.md). Setiap screen dijabarkan: state, validasi, dan error handling.

## 1. Register

| Aspek | Detail |
|---|---|
| State | `idle` → `submitting` → `success` / `error` |
| Input | Nama lengkap, email, nomor telepon, password + confirm password |
| Validasi | Email format valid, nomor telepon format ID (+62), password min. 8 karakter + kombinasi huruf-angka, confirm password match |
| Error handling | Email/nomor sudah terdaftar → tampilkan inline error di field terkait. Network error → toast "Gagal terhubung, coba lagi" dengan tombol retry |
| Loading state | Tombol "Daftar" disabled + spinner saat `submitting` |

## 2. Upload Selfie + KTP

| Aspek | Detail |
|---|---|
| State | `empty` → `captured` → `uploading` → `verifying` → `verified` / `rejected` |
| Validasi | Ukuran file max 5MB, format jpg/png, deteksi blur sebelum upload (client-side jika memungkinkan) |
| Error handling | KYC provider menolak (foto tidak jelas / KTP tidak valid) → tampilkan alasan penolakan + CTA "Upload Ulang" |
| Loading state | Progress bar upload, lalu spinner "Memverifikasi dokumen..." (bisa memakan waktu, beri estimasi/disclaimer) |
| Empty/edge case | Jika user keluar app saat status `verifying`, saat kembali app menampilkan status terakhir dari server (bukan ulang dari awal) |

## 3. OTP Verification

| Aspek | Detail |
|---|---|
| State | `idle` → `submitting` → `success` / `error` |
| Validasi | 6 digit numerik, auto-submit saat lengkap |
| Error handling | OTP salah → shake animation + error text, counter percobaan (max 3x lalu lock sementara). OTP expired → tombol "Kirim Ulang" aktif setelah countdown 60 detik |

## 4. Login

| Aspek | Detail |
|---|---|
| State | `idle` → `authenticating` → `success` / `error` |
| Validasi | Field tidak boleh kosong |
| Behavior | Jika device mendukung biometric & user pernah enable → tampilkan opsi "Login dengan Fingerprint/Face ID" di bawah form password |
| Error handling | Kredensial salah → error generic "Email/password salah" (tidak spesifik untuk mencegah user enumeration). Biometric gagal 2x → fallback paksa ke password |
| Loading state | Tombol login disabled + spinner |

## 5. Home

| Aspek | Detail |
|---|---|
| State | `loading` → `loaded` (dengan 2 varian: `has_active_loan` / `no_active_loan`) → `error` |
| Behavior - `has_active_loan` | Tampilkan card: sisa hutang, tagihan bulan ini, tanggal jatuh tempo. Tombol "Ajukan Pinjaman" **disabled** dengan tooltip/info "Lunasi pinjaman berjalan terlebih dahulu" |
| Behavior - `no_active_loan` | Tampilkan CTA besar "Ajukan Pinjaman Sekarang", tombol aktif |
| Error handling | Gagal load data → skeleton loader diganti empty state dengan tombol "Coba Lagi" |
| Pull to refresh | Mengulang fetch status pinjaman & tagihan terbaru |

## 6. Form Pengajuan Pinjaman

| Aspek | Detail |
|---|---|
| State | `idle` → `validating` → `review` → `submitting` → `success` / `error` |
| Input | Slider/input nominal (max Rp 12.000.000), dropdown tenor (1–12 bulan) |
| Validasi | Nominal tidak boleh 0 atau melebihi limit, tenor wajib dipilih. Kalkulasi estimasi cicilan bulanan ditampilkan real-time saat input berubah |
| Error handling | Submit gagal karena race condition (ternyata ada pinjaman aktif baru masuk dari device lain) → tampilkan error dari server dan redirect ke Home |
| Guard | Screen ini seharusnya tidak reachable jika `has_active_loan = true` (double protection selain disable tombol di Home) |

## 7. Review & Konfirmasi

| Aspek | Detail |
|---|---|
| State | `idle` → `confirming` → `submitted` |
| Behavior | Ringkasan: nominal pinjaman, tenor, estimasi cicilan per bulan, total bayar. Checkbox persetujuan syarat & ketentuan wajib dicentang sebelum tombol submit aktif |

## 8. Status Diproses

| Aspek | Detail |
|---|---|
| State | `processing` → `approved` / `rejected` (polling atau push update) |
| Behavior | Tampilkan animasi loading + copy "Pengajuan sedang diproses, kamu akan mendapat notifikasi". User boleh navigasi keluar screen ini, status tetap diproses di backend |
| Notifikasi | Saat hasil keluar, kirim push notification (jika app di-background) + email + SMS sesuai requirement |

## 9. Detail Pinjaman & Histori Bayar

| Aspek | Detail |
|---|---|
| State | `loading` → `loaded` → `error` |
| Behavior | List histori cicilan (status: akan datang/jatuh tempo/lunas/telat), sisa hutang total |
| Empty state | Jika belum ada histori pinjaman sama sekali → ilustrasi + copy "Belum ada riwayat pinjaman" |

## 10. Profile / Settings

| Aspek | Detail |
|---|---|
| State | `loaded` |
| Behavior | Lihat/update data diri (field tertentu locked setelah KYC verified, misal nama & NIK), toggle enable/disable biometric login, logout |
| Error handling | Update data gagal → toast error, data tidak berubah di UI sampai konfirmasi server sukses (optimistic update dengan rollback) |
