# CLAUDE.md — Staffa

**Staffa** adalah aplikasi HRMS (MVP): data karyawan, absensi dengan geofence, cuti + approval. Spesifikasi lengkap ada di `PRD.md`. Baca PRD sebelum mengerjakan fitur baru dan rujuk ID kebutuhannya (misal `F-LV-2`) di commit/PR.

## Stack

- Next.js (App Router) + TypeScript strict
- Drizzle ORM + PostgreSQL (self-hosted, **bukan Supabase**)
- Better Auth (adapter Drizzle)
- shadcn/ui + Tailwind, TanStack Table, React Hook Form + Zod
- Vitest untuk unit test service
- Package manager: **npm**. Jangan pakai pnpm/yarn/bun; hanya `package-lock.json` yang boleh ada.

## Perintah

```bash
npm install               # install dependency
npm run dev               # dev server
npm run build             # build produksi
npm run lint              # eslint
npm run typecheck         # tsc --noEmit
npm test                  # vitest
npm run db:generate       # drizzle-kit generate (buat migrasi dari schema)
npm run db:migrate        # drizzle-kit migrate
npm run db:studio         # drizzle-kit studio
npm run db:seed           # seed data dev
```

Sebelum menyatakan tugas selesai: jalankan `npm run typecheck && npm run lint && npm test` dan pastikan lolos.

## Struktur

```
src/
  app/(auth)/                 login, reset password
  app/(dashboard)/            employees, attendance, leave, approvals, settings
  modules/<domain>/
    schema.ts                 tabel Drizzle domain ini
    service.ts                SEMUA logika bisnis
    actions.ts                Server Actions (tipis: auth → validasi → service)
    validators.ts             schema Zod (dipakai form & server)
    service.test.ts
  db/index.ts                 instance db
  db/schema.ts                re-export semua modules/*/schema.ts
  lib/auth.ts, lib/rbac.ts, lib/crypto.ts
drizzle/                      file migrasi (hasil generate, jangan diedit manual)
```

Domain: `org`, `employee`, `attendance`, `leave`, `approval`, `audit`.

## Aturan Arsitektur (wajib)

1. **Logika bisnis hanya di `service.ts`.** Server Action, Route Handler, dan komponen tidak boleh query DB langsung atau berisi aturan bisnis.
2. **Cek akses di service**, bukan hanya di middleware/UI. Setiap fungsi service menerima `actor` (user + role + employeeId) dan memvalidasi cakupan data: admin → semua, manager → bawahan langsung + diri sendiri, employee → diri sendiri.
3. **Riwayat kepegawaian effective-dated.** Jangan pernah `UPDATE` jabatan/gaji/cabang di `employment_records`. Tutup record lama (`effective_to`) dan insert record baru, dalam satu transaksi.
4. **Saldo cuti = ledger.** Tidak ada kolom `balance`. Saldo dihitung dari `SUM(leave_transactions.amount)`. Pembatalan dicatat sebagai transaksi pembalik.
5. **Waktu:** simpan sebagai `timestamptz`. Status harian absensi dihitung memakai timezone cabang (`Asia/Jakarta`, `Asia/Makassar`, `Asia/Jayapura`), jangan timezone server.
6. **Uang:** `bigint` rupiah (integer). Tidak boleh float/`real`/`double`.
7. **Operasi multi-tabel memakai `db.transaction`.**
8. **Tidak ada hard delete** untuk data kepegawaian, absensi, cuti; gunakan status/soft delete.
9. **Audit log** ditulis dari service untuk setiap perubahan pada data karyawan, koreksi absensi, transaksi cuti, dan keputusan approval.
10. **Field sensitif** (NIK KTP, NPWP, rekening) dienkripsi lewat `lib/crypto.ts` sebelum disimpan, dan hanya didekripsi untuk role admin.
11. Approval memakai tabel generik `approval_requests`/`approval_steps` dengan `subject_type`. Jangan membuat tabel approval khusus per fitur.

## Konvensi Database (Drizzle)

- Nama tabel dan kolom `snake_case` huruf kecil, nama tabel jamak.
- PK `uuid` (`defaultRandom()`); setiap tabel punya `created_at`, `updated_at`.
- Setiap foreign key diberi index.
- Gunakan constraint di DB (`notNull`, `unique`, `check`) selain validasi Zod.
- Perubahan schema: edit `schema.ts` → `pnpm db:generate` → review SQL migrasi → `pnpm db:migrate`. Jangan pakai `drizzle-kit push` di luar eksperimen lokal.
- Hindari N+1: gunakan relational query atau join, dan paginasi semua list.

## Konvensi UI

- Nama produk di UI, metadata, dan email: **Staffa** (huruf S kapital, sisanya kecil). Nama package/repo: `staffa`.
- Bahasa UI: Indonesia. Tanggal `dd MMM yyyy`, mata uang `Rp 1.250.000`.
- Alur karyawan (absen, cuti) harus mobile-first.
- Komponen dari shadcn/ui; tambah lewat CLI shadcn, jangan menulis ulang dari nol.
- Form: React Hook Form + resolver Zod dari `validators.ts`.
- Tabel data: TanStack Table dengan filter dan paginasi server-side.

## Testing

- Setiap fungsi service yang berisi aturan bisnis wajib punya unit test: perhitungan status absensi, hari kerja cuti, saldo ledger, cek akses RBAC.
- Uji kasus tepi: lintas tengah malam, hari libur, timezone berbeda, saldo pas nol, pengajuan tumpang tindih.

## Skills

Skills tersedia di `~/.agents/skills`. Gunakan sesuai situasi:

| Situasi | Skill |
|---|---|
| Sebelum membangun fitur/komponen baru atau mengubah perilaku | `brainstorming` |
| Menulis atau mengubah schema, query, relasi, atau migrasi Drizzle | `drizzle` |
| Desain tabel, index, constraint, query berat, locking, pagination | `supabase-postgres-best-practices` (berisi praktik Postgres umum; abaikan bagian khusus Supabase) |
| Menambah/menyusun komponen UI, form, theming | `shadcn` |
| Menulis kode apa pun | `antislop-code` |
| Membuat atau merapikan tampilan halaman | `antislop-ui` |
| Layout untuk tampilan HP (alur absen & cuti) | `antislop-layoutmobile` |
| Menulis teks UI, pesan error, empty state, email | `antislop-copywriting` |
| Cek kontras warna / aksesibilitas | `antislop-human` |
| Memverifikasi alur di browser setelah perubahan UI | `agent-browser` atau `browser-use` |
| Butuh kemampuan yang belum ada skill-nya | `find-skills` |

**Jangan gunakan** `supabase` (proyek ini tidak memakai Supabase). Gunakan `migrate-radix-to-base` hanya jika diminta migrasi komponen dari Radix ke Base UI.

## Yang Tidak Boleh Dikerjakan Tanpa Diminta

- Fitur di luar scope MVP (payroll, lembur, reimbursement, mobile app, multi-tenant). Lihat bagian Non-Goals di PRD.
- Menambah dependency baru tanpa menyebutkan alasannya.
- Mengedit file migrasi yang sudah ada di `drizzle/`.
- Menghapus atau melemahkan test supaya lolos.
