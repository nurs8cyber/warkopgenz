# WARKOP GEN Z — Website Bisnis Kuliner & POS Kasir

Redesign penuh website statis menjadi **WARKOP GEN Z** (*Tempat Nongkrong, Ngopi, dan Cerita Tanpa Batas*), siap di-deploy ke **GitHub Pages** dan terintegrasi dengan **Supabase (Auth + PostgreSQL + Row Level Security)**.

---

## Struktur Folder Project

```text
warkop-gen-z/
├── index.html               # Halaman utama pengunjung (Hero, Video Promo, Quotes, Katalog Menu, Free WiFi, Login Modal)
├── admin.html               # Dashboard Admin (11 Menu Sidebar + CRUD Produk, Iklan, Quotes, WiFi, Transaksi, Laporan)
├── kasir.html               # Dashboard Kasir POS (Keranjang, Hitung Kembalian, Proses Transaksi, Cetak Struk)
├── 404.html                 # Halaman 404 kustom untuk GitHub Pages
├── .nojekyll                # Memastikan GitHub Pages menyajikan semua file tanpa Jekyll build
├── README.md                # Dokumentasi instalasi, Supabase RLS, dan deployment GitHub Pages
├── css/
│   └── style.css            # Tema warna Hitam, Cream, Cokelat Kopi, Orange + animasi + print struk
├── js/
│   ├── supabase-config.js   # Konfigurasi URL & Anon Key Supabase (tanpa service_role key)
│   ├── data-store.js        # Abstraksi Database & Autentikasi (Supabase Cloud + Mode Demo Lokal)
│   ├── landing.js           # Logika halaman utama, video embed, rotasi quotes, filter produk, modal login
│   ├── admin.js             # Logika seluruh CRUD Dashboard Admin & proteksi hak akses
│   └── kasir.js             # Logika Point of Sale (POS), kalkulasi kembalian, dan cetak struk
└── images/                  # Seluruh aset logo dan gambar menu makanan/minuman (relative path)
```

---

## Cara Menjalankan & Deploy ke GitHub Pages

1. Buat repository baru di GitHub (misalnya `warkop-gen-z`).
2. Upload seluruh isi folder `warkop-gen-z/` ke root repository (pastikan `index.html` berada di tingkat paling luar).
3. Buka tab **Settings → Pages** pada repository GitHub.
4. Pada bagian **Build and deployment**, pilih:
   - **Source**: `Deploy from a branch`
   - **Branch**: `main` / `/ (root)`
5. Klik **Save** dan tunggu ±1 menit hingga website aktif di `https://USERNAME.github.io/warkop-gen-z/`.

---

## Kredensial Login & Arsitektur Data

### 1. Mode Demo Lokal (Otomatis aktif jika Supabase belum diisi)
- **Admin**: username `admin` | sandi `123`
- **Kasir**: username `kasir` | sandi `123`
- Admin dapat mengubah sandi bawaan atau menambah akun kasir baru melalui menu **9. Manajemen Kasir**.
- *Catatan Transparansi*: Pada mode demo lokal, data disimpan di `localStorage` browser perangkat saat ini. Untuk sinkronisasi lintas perangkat dan autentikasi aman tingkat produksi di GitHub Pages, hubungkan ke **Supabase**.

### 2. Menghubungkan ke Supabase (Produksi Lintas Perangkat)
1. Buat project gratis di [supabase.com](https://supabase.com).
2. Buka **Settings → API**, salin **Project URL** dan **anon / public key** ke file `js/supabase-config.js`.
3. **JANGAN PERNAH** memasukkan `service_role` key ke kode frontend!
4. Jalankan skema SQL berikut di **SQL Editor** Supabase untuk membuat tabel dan mengaktifkan **Row Level Security (RLS)**:

```sql
-- Tabel Produk
create table if not exists warkop_products (
  id text primary key,
  name text not null,
  category text not null,
  price numeric not null default 0,
  image text,
  "desc" text
);

-- Tabel Kategori
create table if not exists warkop_categories (
  id text primary key,
  name text not null,
  icon text,
  description text
);

-- Tabel Video Iklan
create table if not exists warkop_ads (
  id text primary key,
  title text not null,
  url text not null,
  active boolean default false
);

-- Tabel Kata-Kata Mutiara
create table if not exists warkop_quotes (
  id text primary key,
  text text not null,
  author text
);

-- Tabel Pengaturan WiFi
create table if not exists warkop_wifi (
  id int primary key default 1,
  ssid text not null,
  password text not null,
  speed text,
  note text
);

-- Tabel Pengaturan Website
create table if not exists warkop_settings (
  id int primary key default 1,
  "siteTitle" text,
  tagline text,
  address text,
  whatsapp text,
  "waLink" text,
  creator text,
  "openHours" text
);

-- Tabel Transaksi
create table if not exists warkop_transactions (
  id text primary key,
  created_at timestamptz default now(),
  cashier text,
  items jsonb not null,
  total numeric not null,
  paid numeric not null,
  change numeric not null
);

-- Tabel Profil & Role Pengguna
create table if not exists warkop_profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text,
  role text check (role in ('admin', 'kasir')) not null default 'kasir'
);

-- Aktifkan RLS pada semua tabel
alter table warkop_products enable row level security;
alter table warkop_categories enable row level security;
alter table warkop_ads enable row level security;
alter table warkop_quotes enable row level security;
alter table warkop_wifi enable row level security;
alter table warkop_settings enable row level security;
alter table warkop_transactions enable row level security;
alter table warkop_profiles enable row level security;

-- Policy Publik Read-Only untuk Katalog, Iklan, Quotes, WiFi, Settings
create policy "Public read products" on warkop_products for select using (true);
create policy "Public read categories" on warkop_categories for select using (true);
create policy "Public read ads" on warkop_ads for select using (true);
create policy "Public read quotes" on warkop_quotes for select using (true);
create policy "Public read wifi" on warkop_wifi for select using (true);
create policy "Public read settings" on warkop_settings for select using (true);

-- Policy Write khusus Admin terautentikasi
create policy "Admin manage products" on warkop_products for all
  using (exists (select 1 from warkop_profiles where id = auth.uid() and role = 'admin'));
create policy "Admin manage categories" on warkop_categories for all
  using (exists (select 1 from warkop_profiles where id = auth.uid() and role = 'admin'));
create policy "Admin manage ads" on warkop_ads for all
  using (exists (select 1 from warkop_profiles where id = auth.uid() and role = 'admin'));
create policy "Admin manage quotes" on warkop_quotes for all
  using (exists (select 1 from warkop_profiles where id = auth.uid() and role = 'admin'));
create policy "Admin manage wifi" on warkop_wifi for all
  using (exists (select 1 from warkop_profiles where id = auth.uid() and role = 'admin'));
create policy "Admin manage settings" on warkop_settings for all
  using (exists (select 1 from warkop_profiles where id = auth.uid() and role = 'admin'));

-- Policy Transaksi: Kasir & Admin dapat membaca dan menambah transaksi
create policy "Staff read transactions" on warkop_transactions for select
  using (exists (select 1 from warkop_profiles where id = auth.uid() and role in ('admin', 'kasir')));
create policy "Staff insert transactions" on warkop_transactions for insert
  with check (exists (select 1 from warkop_profiles where id = auth.uid() and role in ('admin', 'kasir')));
```

---

**Created by : NUR SARIFUDIN**  
Alamat: Jl. Kemandoran No.119, RT.004/RW.023, Pekayon Jaya, Kec. Bekasi Selatan, Kota Bekasi, Jawa Barat 17148.
