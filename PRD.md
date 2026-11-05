# PRD — Staffa (MVP)

Staffa adalah sistem HRMS untuk perusahaan skala kecil–menengah dengan banyak cabang.

| | |
|---|---|
| **Produk** | Staffa |
| **Versi** | 0.1 (MVP) |
| **Status** | Draft |
| **Stack** | Next.js (App Router) · TypeScript · Drizzle ORM · PostgreSQL · Better Auth · shadcn/ui |
| **Tanggal** | November 2026 |

---

## 1. Latar Belakang & Masalah

Perusahaan skala kecil–menengah dengan beberapa cabang masih mengelola data karyawan, absensi, dan cuti di Excel, grup WhatsApp, dan kertas. Akibatnya:

- Rekap absensi bulanan memakan waktu HR berhari-hari dan rawan salah.
- Saldo cuti tidak jelas, sering diperdebatkan, dan tidak bisa diaudit.
- Pengajuan cuti tidak punya jejak approval.
- Riwayat jabatan dan gaji karyawan tersebar di banyak file.

## 2. Tujuan MVP

1. Karyawan bisa absen (clock in/out) dari browser dengan validasi lokasi cabang.
2. Pengajuan dan approval cuti berjalan sepenuhnya di sistem, dengan saldo yang akurat dan bisa diaudit.
3. HR punya satu sumber data karyawan, termasuk riwayat jabatan.
4. HR bisa mengekspor rekap absensi bulanan ke Excel sebagai input penggajian (payroll masih manual di fase ini).

### Metrik Keberhasilan

| Metrik | Target |
|---|---|
| Karyawan aktif yang absen lewat sistem | ≥ 90% dalam 1 bulan setelah go-live |
| Waktu HR membuat rekap absensi bulanan | dari hitungan hari menjadi < 30 menit |
| Pengajuan cuti di luar sistem | 0 setelah bulan kedua |
| Selisih saldo cuti yang dikomplain | 0 kasus yang tidak bisa dijelaskan dari ledger |

## 3. Non-Goals (di luar MVP)

Payroll (PPh 21, BPJS, slip gaji), lembur, reimbursement, aplikasi mobile native, selfie saat absen, approval multi-level, multi-tenant/SaaS, notifikasi WhatsApp, rekrutmen, penilaian kinerja.

## 4. Pengguna & Role

| Role | Deskripsi | Cakupan data |
|---|---|---|
| `admin` (HR) | Mengelola master data, karyawan, kebijakan cuti, laporan | Semua cabang |
| `manager` | Atasan langsung; menyetujui cuti tim | Bawahan langsung (dan dirinya) |
| `employee` | Absen, ajukan cuti, lihat data sendiri | Dirinya sendiri |

Satu user bisa memiliki satu role. Manager tetap seorang karyawan, jadi juga bisa absen dan mengajukan cuti.

## 5. Kebutuhan Fungsional

### 5.1 Autentikasi & Akses
- **F-AUTH-1** Login dengan email + password; reset password lewat email.
- **F-AUTH-2** Akun dibuat oleh admin (tidak ada self-signup).
- **F-AUTH-3** Akun karyawan nonaktif tidak bisa login.
- **F-AUTH-4** Setiap akses data dicek berdasarkan role **dan** cakupan data (lihat §4) di service layer.

### 5.2 Master Organisasi
- **F-ORG-1** CRUD cabang: nama, alamat, koordinat, radius absen (meter), timezone (WIB/WITA/WIT).
- **F-ORG-2** CRUD departemen dan jabatan.
- **F-ORG-3** Shift default per cabang (jam masuk, jam pulang, toleransi telat dalam menit).
- **F-ORG-4** Kalender hari libur nasional/perusahaan.

### 5.3 Data Karyawan
- **F-EMP-1** Profil: NIK karyawan, nama, email, telepon, tanggal lahir, alamat, tanggal bergabung, status (aktif/nonaktif/resign).
- **F-EMP-2** Riwayat kepegawaian yang effective-dated: cabang, departemen, jabatan, atasan, jenis kontrak (PKWT/PKWTT/harian), gaji pokok, `effective_from`, `effective_to`.
- **F-EMP-3** Perubahan jabatan atau gaji membuat record baru dan menutup record lama, bukan mengubah record yang ada.
- **F-EMP-4** Field sensitif (NIK KTP, NPWP, nomor rekening) dienkripsi di level aplikasi dan hanya terlihat oleh admin.
- **F-EMP-5** Pencarian dan filter karyawan berdasarkan cabang, departemen, dan status.
- **F-EMP-6** Import karyawan dari Excel/CSV (sekali jalan di awal).

### 5.4 Absensi
- **F-ATT-1** Karyawan clock in/out dari browser; sistem mengambil lokasi GPS.
- **F-ATT-2** Clock in di luar radius cabang ditolak, kecuali admin menandai karyawan tersebut "bebas lokasi".
- **F-ATT-3** Status harian dihitung otomatis: hadir, telat, pulang cepat, tidak hadir, cuti, libur.
- **F-ATT-4** Waktu disimpan sebagai `timestamptz`, sedangkan status dihitung memakai timezone cabang.
- **F-ATT-5** Admin bisa mengoreksi absensi dengan alasan wajib; koreksi tercatat di audit log.
- **F-ATT-6** Karyawan melihat riwayat absensinya sendiri; manager melihat absensi timnya.
- **F-ATT-7** Export rekap bulanan ke Excel per cabang: jumlah hadir, telat, tidak hadir, cuti per karyawan.

### 5.5 Cuti
- **F-LV-1** Admin mengatur jenis cuti: tahunan, sakit, izin khusus, dll. Untuk tiap jenis: kuota, apakah memotong saldo, apakah butuh lampiran.
- **F-LV-2** Saldo cuti dicatat sebagai **ledger** (`leave_transactions`): accrual, pengambilan, penyesuaian, kedaluwarsa. Saldo = SUM transaksi.
- **F-LV-3** Accrual cuti tahunan berjalan otomatis lewat job terjadwal.
- **F-LV-4** Karyawan mengajukan cuti (tanggal, jenis, alasan, lampiran opsional). Sistem menghitung hari kerja dengan mengecualikan libur, dan menolak pengajuan bila saldo tidak cukup.
- **F-LV-5** Atasan langsung menyetujui atau menolak (1 level). Pengajuan yang disetujui membuat transaksi ledger pengurangan.
- **F-LV-6** Pembatalan pengajuan yang sudah disetujui membuat transaksi pembalik, bukan menghapus data.
- **F-LV-7** Kalender tim yang menampilkan siapa sedang cuti.

### 5.6 Approval (generik)
- **F-APV-1** Tabel `approval_requests` + `approval_steps` dengan `subject_type`. Di MVP hanya dipakai untuk cuti, tapi disiapkan untuk lembur dan reimbursement.
- **F-APV-2** Inbox approval untuk manager dan admin.

### 5.7 Dashboard
- **F-DASH-1** Admin: jumlah hadir, telat, cuti, dan belum absen hari ini per cabang; pengajuan yang menunggu.
- **F-DASH-2** Manager: status tim hari ini dan pengajuan yang menunggu approval.
- **F-DASH-3** Karyawan: tombol absen, status hari ini, sisa saldo cuti, status pengajuan.

### 5.8 Audit Log
- **F-AUD-1** Mencatat create/update/delete pada data karyawan, koreksi absensi, perubahan saldo, dan keputusan approval: siapa, kapan, sebelum/sesudah.

## 6. Kebutuhan Non-Fungsional

| Area | Kebutuhan |
|---|---|
| Performa | Halaman utama < 2 detik; clock in < 1 detik di jaringan 4G |
| Responsif | Seluruh alur karyawan (absen, cuti) nyaman dipakai di HP |
| Keamanan | Password di-hash (via Better Auth); field sensitif dienkripsi; HTTPS; rate limit login |
| Data | Backup PostgreSQL harian, retensi 30 hari; restore pernah diuji |
| Integritas | Uang disimpan sebagai integer rupiah (`bigint`); operasi multi-tabel memakai transaksi DB |
| Bahasa | UI Bahasa Indonesia; format tanggal `dd MMM yyyy`; mata uang Rp |
| Skala awal | ≤ 500 karyawan, ≤ 20 cabang |

## 7. Model Data Inti (ringkas)

```
branches, departments, positions, shifts, holidays
users (Better Auth) ── employees (1:1)
employees ── employment_records (1:N, effective-dated)
attendance_records (employee, date, clock_in, clock_out, lat/lng, status)
attendance_corrections
leave_types, leave_transactions (ledger), leave_requests
approval_requests, approval_steps
audit_logs
```

Prinsip:
- Gunakan riwayat effective-dated, jangan kolom yang di-overwrite.
- Saldo cuti dicatat sebagai ledger, jangan sebagai counter.
- Pakai soft delete/status, bukan hard delete, untuk data kepegawaian.

## 8. Arsitektur

- **Modular monolith:** `src/modules/<domain>/{schema,service,actions,validators}.ts`.
- Server Components untuk halaman; Server Actions untuk mutasi; Route Handlers hanya untuk integrasi atau mobile di masa depan.
- Logika bisnis hanya ada di `service.ts`, sehingga bisa dites tanpa Next.js.
- Deploy: VPS + Docker Compose (app + Postgres), backup terjadwal.

## 9. Rencana Rilis

| Minggu | Deliverable |
|---|---|
| 1 | Setup proyek, auth, RBAC, master organisasi |
| 2 | Data karyawan + riwayat kepegawaian + import |
| 3 | Absensi (geofence, status harian, koreksi) |
| 4 | Cuti + ledger + approval |
| 5 | Dashboard, export Excel, audit log |
| 6 | Testing, UAT dengan HR, deploy, migrasi data awal |

## 10. Fase Berikutnya

1. Payroll: komponen gaji, BPJS, PPh 21 (tarif disimpan sebagai tabel konfigurasi yang effective-dated), payroll run yang immutable, slip PDF, THR.
2. Lembur dan reimbursement memakai approval engine yang sama.
3. Aplikasi absen mobile (geofence + selfie), approval multi-level, notifikasi.

## 11. Pertanyaan Terbuka

- Untuk satu perusahaan (project klien) atau SaaS multi-tenant? Jawabannya menentukan apakah perlu `tenant_id` dan RLS.
- Aturan accrual cuti tahunan: 12 hari di depan setelah 1 tahun kerja, atau 1 hari/bulan?
- Apakah ada karyawan dengan lebih dari satu shift atau shift malam (lintas tanggal)?
- Siapa yang menyetujui cuti seorang manager: atasannya atau HR?
