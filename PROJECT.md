# PROJECT: Depot Air Galon (MVP)

## Aturan untuk semua AI
- Ikuti dokumen ini persis. Jangan ubah nama tabel, kolom, file, atau fungsi.
- Kalau butuh perubahan, tulis usulan dulu, jangan langsung ubah.
- Semua panggilan ke Supabase hanya lewat js/api.js.
- Teks tampilan Bahasa Indonesia. Nama kode memakai Inggris snake_case persis seperti di bawah.
- Stack: HTML + JavaScript biasa + Supabase JS (CDN). Tanpa framework.
- Hanya anon key di frontend. service_role key dan token Telegram TIDAK boleh ada di repo.

## Struktur file
index.html -> halaman pelanggan (form pesan)
admin.html -> dasbor toko (pesanan, stok, langganan, reminder)
kurir.html -> daftar antar per area + tombol Selesai
js/config.js -> SUPABASE_URL dan SUPABASE_ANON_KEY
js/api.js -> satu-satunya tempat query Supabase
supabase/schema.sql -> skema database

## Tabel
brands(id, name, price_refill, price_new_gallon)
customers(id, name, whatsapp, address, area, lat, lng, created_at)
inventory(brand_id PK->brands, full_qty, empty_qty)
orders(id, customer_id, brand_id, service_type, qty, status, payment_method, courier_name, note, created_at, completed_at)
  service_type: 'tukar' | 'beli_baru' | 'pinjam'
  status: 'baru' | 'diantar' | 'selesai' | 'batal'
  payment_method: 'cash' | 'qris' | 'kuota'
subscriptions(id, customer_id, quota_total, quota_left, price_paid, created_at)

## Fungsi di js/api.js (nama tidak boleh diubah)
buatPesanan(data) -> insert customers (jika baru) + orders
ambilPesananBaru() -> orders status 'baru', urut per area
ambilStok() -> inventory + nama brand
tandaiDiantar(order_id)
selesaikanPesanan(order_id) -> memanggil RPC selesaikan_pesanan
tambahKuota(customer_id, jumlah, harga)
ambilPelangganPerluReminder() -> pelanggan yang rata-rata hari habis galonnya sudah tercapai
linkMaps(lat, lng) -> https://www.google.com/maps?q=LAT,LNG
linkRute(daftarKoordinat) -> https://www.google.com/maps/dir/?api=1&destination=...&waypoints=A|B

## Aturan bisnis (dijalankan di database, bukan di frontend)
Fungsi SQL selesaikan_pesanan(order_id), atomik, hanya jalan jika status belum 'selesai':
- tukar: full_qty - qty, empty_qty + qty
- beli_baru: full_qty - qty
- pinjam: full_qty - qty
- payment_method 'kuota': subscriptions.quota_left - 1 (tolak jika kuota 0)
- set status 'selesai' dan completed_at = now()
Stok tidak boleh negatif.

## Keamanan (RLS aktif di semua tabel)
- Publik (anon): hanya boleh INSERT ke orders dan customers, dan baca brands.
- Admin/kurir: login Supabase Auth, baru boleh baca/ubah semua tabel.

## Urutan pengerjaan (satu fitur per sesi)
1. schema.sql + RLS + fungsi selesaikan_pesanan
2. index.html (form pesan)
3. admin.html (pesanan + stok) dan kurir.html (tombol Selesai)
4. Notifikasi Telegram ke kurir (Supabase Edge Function)
5. Langganan/kuota
6. Pengelompokan area + tombol Mulai Rute
7. Halaman reminder dengan tombol wa.me
