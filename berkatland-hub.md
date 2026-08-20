# BerkatLand Hub — Multi-Feature Implementation Plan

## Scope (6 Fitur Utama + 2 Enhancement)

---

### 1. Fix Dashboard SuperAdmin Project Filter Bug
**Problem:** Query `paidBills` di `loadData()` mengambil semua baris tanpa filter project di level DB, lalu filter client-side. Jika `project_name` null atau mismatch, data tidak terhitung.

**Fix:**
- Tambahkan `.eq("project_name", projectFilter)` langsung pada query DB untuk bills, paidBills
- Buat query `complaints` dan `feedbacks` list juga di-filter per project
- Fix `paidToday` calculation agar consistent

---

### 2. Unit Type Sync (Admin Settings → Konsumen)
**Problem:** `MyUnit.tsx` menggunakan hardcoded `UNIT_TYPES_BY_PROJECT` dict, tidak membaca dari admin settings.

**Fix:**
- Ganti `UNIT_TYPES_BY_PROJECT[form.project_name]` dengan `settings.project_unit_types[form.project_name]` dari `useSettingsStore`
- Add fallback ke hardcoded list jika settings kosong
- Tambah `fetchSettings()` jika belum dipanggil

---

### 3. Claim Auto-Reject Jika Unit Tidak Ada
**Problem:** Konsumen bisa klaim unit yang tidak ada di DB admin, dan klaim tetap "pending" tanpa penolakan otomatis.

**Fix:**
- Tambah DB Migration: Trigger PostgreSQL pada `unit_claims` INSERT
- Trigger cek: apakah ada unit dengan `block = NEW.block AND number = NEW.unit_number AND project_name = NEW.project_name`
- Jika tidak ada → SET status = 'rejected', tambah `rejection_reason = 'Unit tidak ditemukan dalam database'`
- Update UI consumer untuk tampilkan alasan penolakan

---

### 4. WhatsApp 1-Click per Tagihan
**Problem:** Belum ada tombol WA per baris tagihan yang langsung kirim dengan nominal + payment link.

**Fix (di Bills.tsx):**
- Tambah `whatsapp` field ke query profiles
- Tambah tombol WA (MessageCircle icon) di setiap baris tagihan
- Generate pesan: nama, unit, nominal tagihan, periode, jatuh tempo, payment link
- Payment link: `{origin}/app/bills` (deep link ke halaman tagihan di app)
- Buka `wa.me/{phone}?text={pesan}` langsung tanpa modal tambahan

---

### 5. Add Item (Billing Template + Auto-Recurring)
**User response:** Auto-recurring — admin atur sekali, tagihan terbit otomatis tiap bulan.

**DB Schema:**
```sql
CREATE TABLE billing_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_name TEXT NOT NULL,
  house_type TEXT NOT NULL,
  ipl_amount NUMERIC DEFAULT 0,
  water_price_per_unit NUMERIC DEFAULT 0,
  bill_due_days INTEGER DEFAULT 30,  -- hari dari issue date ke due date
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(project_name, house_type)
);
```

**Edge Function `auto-generate-bills`:**
- Baca semua `billing_templates` yang `active = true`
- Untuk setiap template, cari semua profil user dengan `project_name` match dan `unit_type` match
- Cek apakah bill IPL bulan ini sudah ada (cegah duplikat)
- Buat bill IPL jika belum ada
- Buat bill air jika `water_price_per_unit > 0` dan belum ada

**pg_cron:**
```sql
SELECT cron.schedule('auto-bills', '0 0 1 * *', $$
  SELECT net.http_post(
    url := '{EDGE_FUNCTION_URL}/auto-generate-bills',
    headers := '{"Authorization": "Bearer {ANON_KEY}"}'
  );
$$);
```

**New Page `/admin/add-item`:**
- List semua billing templates per project
- Form: Project, House Type, IPL Amount, Water Price/m³, Due Days
- Toggle active/inactive
- Tombol "Generate Sekarang" → manual trigger edge function
- Stats: berapa tagihan sudah dibuat bulan ini

---

### 6. Halaman Transaksi Table
**New Page `/admin/transactions`:**
- Tabel dengan kolom: DateTime, ID Pembayaran, Metode, Channel, Status, Amount, Konsumen
- Filter: tanggal, status, metode pembayaran
- Badge status: Pending (kuning), Lunas (hijau), Expired (abu), Gagal (merah)
- Channel mapping: virtual_account → "Virtual Account", ewallet → "E-Wallet", qris → "QRIS", dll
- Hanya tampilkan metode yang aktif di payment settings
- Export CSV

---

### 7. VA & E-Wallet (Manual Mode — Tanpa Midtrans)
**Karena tidak ada Midtrans API key:**
- VA: Konsumen lihat nomor VA yang admin atur, transfer via banking app, upload bukti
- E-Wallet: Konsumen lihat nomor HP/QR yang admin atur, bayar via GoPay/OVO/Shopeepay, upload bukti
- Admin konfirmasi manual di PaymentsDashboard
- Tidak ada konfirmasi otomatis (tidak bisa tanpa Midtrans/Xendit)
- **Enhancement:** Tampilkan instruksi pembayaran yang lebih jelas di sisi konsumen (langkah-langkah transfer)

---

## Files to Create/Modify

### New Files:
- `src/pages/admin/AddItem.tsx` — billing template management page
- `src/pages/admin/AdminTransactions.tsx` — transaction table page
- `supabase/functions/auto-generate-bills/index.ts` — auto billing edge function
- New migration: billing_templates table + auto-reject trigger + pg_cron

### Modified Files:
- `src/pages/admin/Dashboard.tsx` — fix project filter
- `src/pages/app/MyUnit.tsx` — unit type sync from settings
- `src/pages/admin/Bills.tsx` — WA 1-click per bill
- `src/pages/admin/AdminLayout.tsx` — add nav items
- `src/router.tsx` — add routes

---

## Implementation Order
1. DB Migration (billing_templates + trigger)
2. Fix Dashboard filter
3. Unit type sync in MyUnit.tsx
4. WA per-bill in Bills.tsx
5. AddItem page + Edge Function
6. AdminTransactions page
7. Add routes + nav items
