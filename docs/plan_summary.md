# Plan Summary — Web App Laporan Keuangan SPPG (MBG)  
**Acuan utama:** SK No. 401.1 Tahun 2025 – *Juknis Tata Kelola MBG TA 2026*  
**Tujuan dokumen ini:** menjadikan semua output sebelumnya (database + workflow + UI + OpenAPI) sebagai **satu kesatuan rencana implementasi** yang bisa Anda pakai sebagai acuan pengembangan di VS Code/Codex.

---

## 0) Ringkas Produk yang Dibangun
Web App ini adalah sistem terpadu untuk:
1. **Pencatatan transaksi keuangan** (VA/transfer & kas kecil/petty cash) + lampiran bukti.
2. **Pencatatan penyaluran MBG** (tanda terima + dokumentasi + daftar hadir).
3. **Pencatatan pajak** (WAPU/WAPOT + bukti setor).
4. **Generate laporan**: harian, 2 mingguan (LPJ), bulanan.
5. **Generate dokumen LPJ**: BAST, rekap, dan **bundle ZIP** berisi semua lampiran.
6. **Approval workflow & audit trail** untuk memenuhi prinsip otorisasi dan akuntabilitas.

---

## 1) “Compliance Rules” yang Harus Dikunci di Sistem (Juknis)
Buat rule ini sebagai **validasi backend** + **guard UI** (biar pengguna tidak bisa “jalan pintas”):
1. **LPJ tiap 2 minggu** dan wajib lampiran:  
   - laporan harian (untuk auto top up) + laporan 2 mingguan  
   - BAST  
   - tanda terima setiap penyaluran  
   - dokumentasi (foto/video) + daftar hadir penerima manfaat  
2. **Semua pengeluaran dana** harus ada otorisasi **Kepala SPPG** dan **perwakilan Yayasan**.
3. **Kas kecil (petty cash):**  
   - maks. **Rp500.000/transaksi**  
   - total **≤ Rp5.000.000/minggu**  
   - harus ada bukti fisik (struk/faktur)  
   - dicatat Pengawas Keuangan, dipertanggungjawabkan bulanan, sisa disetor kembali ke VA
4. **Pajak:** wajib pungut/potong dan bukti setor disimpan.
5. **VA MBG:** rekening VA atas nama Yayasan punya **2 user** (maker=perwakilan yayasan, approver=Kepala SPPG).

> Implementasi: rule #1–#5 wajib dibuat sebagai *server-side validations* + *status gate* (tidak boleh hanya di frontend).

---

## 2) Peran (Roles) & Hak Akses (RBAC)
Minimal 4 role:
- `maker_yayasan` — input transaksi + otorisasi yayasan
- `approver_kepala_sppg` — otorisasi kepala
- `pengawas_keuangan` — review bukti, rekonsiliasi, pajak, laporan
- `auditor` — read-only + download bundle + audit log

### Prinsip akses
- Semua data **selalu scope ke `sppg_id`**.
- Endpoint workflow (submit/review/approve/lock) harus cek:
  - role yang benar
  - urutan status benar
  - bukti & lampiran lengkap

---

## 3) Data Model (Database) — Ringkasan Struktur
Gunakan Postgres. Tabel utama (minimal):
- Master: `yayasan`, `sppg`, `school`, `vendor`
- User & RBAC: `app_user`, `user_role`
- Bukti: `attachment` (doc_type, file_url, uploaded_by)
- Keuangan:
  - `expense` + `expense_item` + `expense_attachment`
  - `petty_cash_tx` + `petty_cash_attachment`
  - `va_account`, `va_mutation` (rekonsiliasi, saldo)
  - `tax_record` (relasi ke expense + bukti setor)
- Penyaluran:
  - `distribution` + `distribution_attachment` (receipt/photo/video/attendance)
- Pelaporan:
  - `report` (daily/biweekly/monthly)
  - `bast`
  - `document_bundle` (zip)
- Governance:
  - `approval`
  - `audit_log`

### Constraint & Trigger Wajib
1. Petty cash:
   - check `amount <= 500000`
   - trigger weekly sum `<= 5000000`
2. Status-lock:
   - jika status `locked` → blok edit (patch)
3. Approval gate:
   - expense/petty cash tidak boleh approve bila:
     - tidak ada attachment minimal
     - pajak required tapi belum ada record/bukti (sesuai kebijakan internal)
4. Report gate:
   - report biweekly tidak boleh `lock` sebelum `bundle_validate` OK.

---

## 4) Workflow Status Machine
Status standar: `draft → submitted → reviewed → approved → locked` (atau `rejected`)

### Workflow Expense
- Maker membuat draft + upload bukti
- Submit
- Pengawas keuangan review (cek bukti + pajak)
- Approve yayasan
- Approve kepala
- Lock

### Workflow LPJ 2 Mingguan
- Generate report biweekly (draft)
- Generate BAST
- Validasi kelengkapan lampiran (gate checklist)
- Generate ZIP bundle
- Submit → Review → Approve yayasan → Approve kepala → Lock

---

## 5) API Implementation (berdasarkan OpenAPI YAML)
### 5.1 Prinsip implementasi API
- Semua endpoint berada di `/api/v1/*`
- Semua request lewat middleware:
  - auth (JWT)
  - `sppg_scope_guard`
  - validation (schema)
  - audit logging

### 5.2 Cara menjadikan OpenAPI sebagai “single source of truth”
**Pilihan workflow (rekomendasi):**
1. Simpan file `openapi.yaml` di repo: `docs/openapi.yaml`
2. Generate:
   - validator server (request/response)
   - types client (TypeScript types)
   - API client (opsional)

Tool yang umum:
- `openapi-generator`
- `orval` (TS client)
- `zod` generator (untuk validasi runtime)

### 5.3 Mapping Endpoint → Service Layer
Buat service per domain:
- `attachmentsService`
- `expensesService`
- `pettyCashService`
- `distributionsService`
- `taxService`
- `reportsService`
- `approvalsService`
- `auditService`

Setiap service memiliki:
- `create/update/get/list`
- `submit/review/approve/lock/reject`
- validator + business rules

---

## 6) UI/UX Implementation (halaman wajib)
Struktur halaman (minimal MVP):
1. Login
2. Dashboard (KPI + quick actions)
3. Transaksi Cepat (mobile-first)
4. Daftar & Detail Expense
5. Kas Kecil (petty cash)
6. Penyaluran MBG (per sekolah/hari)
7. Pajak (rekap + bukti setor)
8. Laporan Harian (generate + PDF)
9. Wizard LPJ 2 Mingguan (generate + checklist + bundle ZIP)
10. Laporan Bulanan
11. Approval Inbox
12. Audit Log Viewer (khusus auditor/keu)

### UI Guard (wajib)
- Tombol “Approve/Lock” hanya muncul jika:
  - role sesuai
  - status sesuai
  - attachment lengkap

---

## 7) PDF & ZIP Bundle Generation
### 7.1 Dokumen yang digenerate
- Laporan Harian (PDF)
- LPJ 2 Mingguan (PDF)
- BAST (PDF)
- Rekap Pajak (PDF)
- Kas Kecil Bulanan (PDF)
- Tanda terima insentif (opsional)

### 7.2 Cara generate PDF (rekomendasi paling fleksibel)
- Render HTML template → PDF dengan:
  - Playwright / Puppeteer
  - atau wkhtmltopdf
- Simpan hasil PDF sebagai `attachment(doc_type=...)`.

### 7.3 ZIP Bundle LPJ
Isi ZIP:
- LPJ 2 mingguan PDF
- BAST PDF
- Semua laporan harian dalam periode
- Lampiran penyaluran (receipt/photo/video/attendance)
- Bukti pajak
- (opsional) Kas kecil bulanan jika periode overlap

Naming convention:
- `LPJ_{SPPG}_{YYYYMMDD}-{YYYYMMDD}.zip`

---

## 8) Roadmap Implementasi (MVP → v1)
### Phase 0 — Setup Repo & Tooling (1–2 hari)
- Inisialisasi repo + CI
- Setup Postgres + migration tool
- Setup auth (JWT)
- Setup storage attachment (local/minio/s3)

**DoD:** login + create attachment metadata + read me

### Phase 1 — Core Transactions (Expense + Petty Cash) (3–7 hari)
- CRUD expense + items
- upload/link attachment
- workflow submit/review/approve/lock
- petty cash dengan limit trx + weekly cap

**DoD:** transaksi bisa locked + audit log berjalan

### Phase 2 — Penyaluran & Pajak (3–7 hari)
- CRUD distribution + attachments
- tax record + bukti setor
- halaman UI penyaluran + pajak

**DoD:** semua bukti penyaluran bisa direkap per periode

### Phase 3 — Reporting & LPJ Bundle (5–10 hari)
- generate report daily/biweekly/monthly
- generate BAST
- validate bundle checklist
- generate ZIP bundle

**DoD:** LPJ 2 mingguan bisa 1 klik download ZIP

### Phase 4 — Hardening (v1)
- role & permission matrix lengkap
- rekonsiliasi VA + import CSV
- monitoring + backup
- UAT & training

---

## 9) Struktur Repo (contoh monorepo)
```
sppg-finance/
  apps/
    api/                  # backend
    web/                  # frontend
  packages/
    shared/               # types, constants, utils
  docs/
    openapi.yaml
    plan_summary.md
  infra/
    docker/
    migrations/
  scripts/
```

---

## 10) Checklist “Benar dan Siap Dipakai” (Acceptance Criteria)
### Transaksi
- [ ] Expense VA/transfer wajib punya bukti (invoice + bukti transfer jika relevan)
- [ ] Petty cash: blok jika > 500.000 per trx
- [ ] Petty cash: warning + blok jika total minggu > 5.000.000
- [ ] Semua transaksi punya audit log create/update/status change

### Penyaluran
- [ ] Untuk setiap penyaluran: receipt wajib, dokumentasi & attendance bisa dikumpulkan
- [ ] Sistem bisa cek kelengkapan per sekolah/hari

### LPJ 2 Mingguan
- [ ] Wizard bisa generate report + BAST
- [ ] Gate validate memastikan semua lampiran wajib terpenuhi
- [ ] ZIP bundle bisa diunduh dan tersimpan sebagai attachment

### Pajak
- [ ] Tax record bisa dibuat per expense
- [ ] Bukti setor pajak bisa diupload dan tertaut

### Security
- [ ] RBAC benar (role-based endpoint)
- [ ] Scope `sppg_id` tidak bisa ditembus
- [ ] Data locked tidak bisa diubah

---

## 11) Langkah Praktis Memulai di VS Code (tanpa pilih framework dulu)
1. Buat repo baru `sppg-finance`
2. Tambahkan folder `docs/` dan simpan:
   - `docs/openapi.yaml` (OpenAPI)
   - `docs/plan_summary.md` (dokumen ini)
3. Setup DB + migration:
   - buat `infra/migrations/*.sql`
4. Implement backend mengikuti urutan:
   - attachments → expenses → petty cash → distributions → taxes → reports → bundle
5. Implement frontend berdasarkan halaman MVP:
   - transaksi cepat & inbox approval dulu (paling sering dipakai)
6. Uji dengan dataset contoh:
   - 1 SPPG, 3 sekolah, 5 vendor, 14 hari transaksi, 14 hari penyaluran

---

## 12) Catatan Implementasi (biar tidak “nyasar”)
- **Business rule harus di backend**, bukan hanya UI.
- **Workflow endpoint harus idempotent** (klik dua kali tidak merusak data).
- **Semua file lampiran** disimpan terpusat (`attachment`), lalu dikaitkan ke entity.
- **Generate PDF/ZIP** sebaiknya jalan di background job (queue) bila volume besar, tapi MVP boleh sync jika file kecil.
- Pastikan ada “export & download” untuk audit.

---

## 13) Deliverables (yang harus ada di repo saat mulai coding)
- `docs/openapi.yaml`
- `docs/plan_summary.md`
- `infra/migrations/` (DDL schema)
- `apps/api/` (backend)
- `apps/web/` (frontend)
- `README.md` (cara run lokal)

---

## 14) Next Step (yang paling efektif)
Kalau Anda ingin, saya bisa lanjutkan dengan:
1) Template **migration SQL** (sesuai schema)  
2) Template **backend skeleton** (controller/service/validators) berdasarkan OpenAPI  
3) Template **UI skeleton** (routing + layout + halaman inti)  

Cukup bilang: “lanjutkan ke skeleton backend + migration”.
