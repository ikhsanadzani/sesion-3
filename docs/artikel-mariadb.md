# Pemanfaatan AI untuk Normalisasi MariaDB (1NF → 2NF → 3NF) — Panduan Praktis Step-by-Step

Mantap — berikut artikel lengkap, praktis, dan bisa langsung kamu pakai untuk **menggunakan AI** (mis. ChatGPT / LLM lain) membantu normalisasi database MariaDB dari bentuk denormal (gabungan) ke **1NF → 2NF → 3NF**. Semua langkah disusun supaya bisa diikuti: backup → analisis → rekomendasi AI → skrip migrasi → validasi → deploy.

---

## Ringkasan singkat
- **Tujuan:** Hilangkan redudansi dan anomaly, susun tabel sesuai 1NF/2NF/3NF.  
- **Peran AI:** Membantu analisis schema & sample data, merekomendasi dekomposisi, men-generate DDL & migration SQL, menulis skrip ETL/validasi.  
- **Prinsip:** Selalu *backup*, coba di staging, verifikasi data, lalu pindah ke produksi.

---

## Persiapan (prasyarat)
1. Akses ke DB (user dengan cukup hak).  
2. MariaDB client terpasang (`mysql`, `mysqldump`).  
3. Lingkungan staging / backup.  
4. Akses ke LLM (ChatGPT atau model lain).  
5. Tool ops untuk menjalankan skrip SQL (CLI atau Python).

Contoh backup cepat:
```bash
# dump seluruh database
mysqldump -u root -p --databases nama_db > backup_nama_db.sql

# atau dump satu tabel
mysqldump -u root -p nama_db nama_tabel > backup_tabel.sql
```

---

## Alur kerja singkat (high level)
1. Snapshot / backup.  
2. Inventarisasi schema & sample data.  
3. Minta AI menganalisis (beri DDL + contoh rows).  
4. AI hasilkan rekomendasi tabel ter-normalisasi + DDL + migration SQL.  
5. Jalankan di staging, validasi integritas & counts.  
6. Terapkan di produksi (dengan rollback plan).  
7. Optimasi indeks dan queries.

---

## Langkah-langkah detail (praktis)

### 1) Inventarisasi: ambil schema & contoh data
Ambil DDL tiap tabel:
```sql
SHOW TABLES;
SHOW CREATE TABLE nama_tabel\G
```
Ambil beberapa row contoh:
```sql
SELECT * FROM nama_tabel LIMIT 20;
```
Dump schema tanpa data (untuk dikirim ke AI):
```bash
mysqldump -u user -p --no-data nama_db > schema.sql
```

### 2) Prompt untuk AI: minta analisis & rekomendasi
**Contoh prompt (bahasa Indonesia)** — copy paste ke ChatGPT/LLM:
```
Aku punya tabel MariaDB bernama `orders_raw`. Ini DDL-nya:
<paste hasil SHOW CREATE TABLE orders_raw>

Contoh beberapa baris:
<paste 5-10 row hasil SELECT ... LIMIT ...>

Tolong:
1. Analisis masalah normalisasi (apakah melanggar 1NF/2NF/3NF, jelaskan).
2. Rekomendasikan dekomposisi ke tabel terpisah (nama tabel + kolom).
3. Buatkan DDL (CREATE TABLE) untuk setiap tabel target dengan tipe data, PK, FK.
4. Berikan migration SQL (INSERT INTO new_table SELECT ... FROM orders_raw) beserta urutan aman.
5. Buatkan skrip validasi untuk memastikan total/jumlah/checksum cocok.
6. Jika perlu, saran indeks untuk performa.

Format jawaban: step-by-step + skrip SQL siap pakai.
```

### 3) Contoh kasus lengkap — dari denormal ke 3NF  
Misal tabel `orders_raw` (denormal):
```sql
CREATE TABLE orders_raw (
  id INT AUTO_INCREMENT PRIMARY KEY,
  order_date DATETIME,
  customer_name VARCHAR(200),
  customer_email VARCHAR(200),
  product_id INT,
  product_name VARCHAR(255),
  product_price DECIMAL(10,2),
  qty INT,
  shipping_address TEXT
);
```
**Masalah:** data customer & product berulang di tiap baris → redundansi, update anomaly.

**Rekomendasi dekomposisi (contoh)**:
- `customers` (customer_id, name, email)
- `products` (product_id, name, price)
- `orders` (order_id, order_date, customer_id, shipping_address_id)
- `order_items` (order_item_id, order_id, product_id, qty, unit_price)
- `addresses` (address_id, raw_address, city, etc) — optional split

**DDL contoh:**
```sql
CREATE TABLE customers (
  customer_id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  email VARCHAR(200),
  UNIQUE(email)
) ENGINE=InnoDB CHARSET=utf8mb4;

CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  price DECIMAL(10,2) NOT NULL
) ENGINE=InnoDB CHARSET=utf8mb4;

CREATE TABLE orders (
  order_id INT AUTO_INCREMENT PRIMARY KEY,
  order_date DATETIME NOT NULL,
  customer_id INT NOT NULL,
  shipping_address TEXT,
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
) ENGINE=InnoDB CHARSET=utf8mb4;

CREATE TABLE order_items (
  order_item_id INT AUTO_INCREMENT PRIMARY KEY,
  order_id INT NOT NULL,
  product_id INT NOT NULL,
  qty INT NOT NULL,
  unit_price DECIMAL(10,2),
  FOREIGN KEY (order_id) REFERENCES orders(order_id),
  FOREIGN KEY (product_id) REFERENCES products(product_id)
) ENGINE=InnoDB CHARSET=utf8mb4;
```

**Migration SQL (langkah aman):**
1. Buat tabel target di staging DB (DDL di atas).
2. Isi `customers` (unique by email / name):
```sql
INSERT INTO customers (name, email)
SELECT DISTINCT customer_name, customer_email FROM orders_raw;
```
3. Isi `products` (asumsi product_id sudah ada):
```sql
INSERT INTO products (product_id, name, price)
SELECT DISTINCT product_id, product_name, product_price FROM orders_raw;
```
4. Isi `orders` (buat 1 order per order id — jika id dari orders_raw unik per order)
```sql
INSERT INTO orders (order_id, order_date, customer_id, shipping_address)
SELECT id AS order_id, order_date,
       (SELECT customer_id FROM customers c WHERE c.email = orders_raw.customer_email) AS customer_id,
       shipping_address
FROM orders_raw
GROUP BY id, order_date, customer_email, shipping_address;
```
5. Isi `order_items`:
```sql
INSERT INTO order_items (order_id, product_id, qty, unit_price)
SELECT id AS order_id, product_id, qty, product_price FROM orders_raw;
```

### 4) Validasi setelah migrasi
Beberapa cek yang sangat penting:

- **Row count & checksum**:
```sql
-- total row count original
SELECT COUNT(*) FROM orders_raw;

-- total items after migration
SELECT SUM(qty) FROM order_items;
```
- **Jumlah orders**:
```sql
SELECT COUNT(DISTINCT id) FROM orders_raw;
SELECT COUNT(*) FROM orders;
```
- **Integrity check (missing FK)**:
```sql
-- cari order_items tanpa order
SELECT oi.* FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_id IS NULL;
```
- **Spot-check sample rows**: ambil beberapa order id random dan bandingkan semua kolom antara `orders_raw` dan join tabel baru.

- **Checksum per order** (bandingkan total price):
```sql
-- original per order
SELECT id, SUM(product_price * qty) AS total_orig
FROM orders_raw GROUP BY id;

-- new per order
SELECT o.order_id, SUM(oi.unit_price * oi.qty) AS total_new
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY o.order_id;
```

### 5) Rename & cutover (strategi aman)
1. Jangan drop original table langsung. Simpan sebagai backup:
```sql
RENAME TABLE orders_raw TO orders_raw_backup;
RENAME TABLE orders TO orders;
-- atau: RENAME TABLE orders_new TO orders; (jika kamu buat orders_new)
```
2. Update aplikasi untuk pakai schema baru. Lakukan test.  
3. Setelah beberapa waktu dan yakin, drop `orders_raw_backup`.

### 6) Indeks & optimasi
- Tambahkan index pada FK dan kolom yang sering di-`WHERE`:
```sql
CREATE INDEX idx_order_customer ON orders(customer_id);
CREATE INDEX idx_order_items_order ON order_items(order_id);
```
- Pertimbangkan composite index jika query memerlukannya.

### 7) Rollback plan
- Simpan `orders_raw_backup` sampai 7—30 hari.  
- Siapkan skrip untuk memindahkan kembali jika terjadi masalah (pakai `RENAME TABLE` utk swap kembali).

---

## Bagaimana AI membantu secara spesifik (prompt & contoh)
### Prompt: "Analisis schema + buat DDL + migration"
```
Saya punya DDL dan contoh data untuk table X (paste DDL + 10 sample rows).
Tolong:
1) Jelaskan pelanggaran 1NF/2NF/3NF.
2) Rekomendasikan nama tabel dan kolom terpisah.
3) Buatkan DDL (MariaDB) lengkap dengan PK/FK.
4) Buatkan migration SQL langkah demi langkah (safe).
5) Buatkan queries validasi (count/checksum/integrity).
Format: numbered steps + SQL blocks.
```

### Prompt: "Buat skrip Python ETL"
```
Buatkan skrip Python (pakai mysql-connector-python atau PyMySQL) yang:
- Membaca baris dari orders_raw,
- Menyisipkan ke customers/products jika belum ada (upsert),
- Membuat order & order_items,
- Menggunakan transaksi,
- Logging progress.
```

---

## Best practices & catatan penting
- **Selalu backup** sebelum operasi besar.  
- **Coba di staging**; cek performance.  
- **Gunakan transaksi** untuk migrasi sehingga bisa rollback jika error.  
- **Matikan FOREIGN_KEY_CHECKS sementara** hanya jika butuh impor masal, lalu nyalakan lagi:
```sql
SET FOREIGN_KEY_CHECKS = 0;
-- import
SET FOREIGN_KEY_CHECKS = 1;
```
- **AI tidak menggantikan review manusia**: selalu periksa tipe data, constraint, dan business rules sebelum deploy.  
- **Dokumentasi**: simpan mapping (kolom lama → tabel/kolom baru) supaya dev lain paham.

---

## Checklist singkat sebelum push ke produksi
- [ ] Backup full DB dibuat.  
- [ ] Skrip migration diuji di staging.  
- [ ] Semua counts & checksums valid.  
- [ ] Aplikasi diuji terhadap schema baru.  
- [ ] Index ditambahkan & query di-tune.  
- [ ] Rollback plan tersedia.

---

## Penutup
Itu dia — format artikel praktis step-by-step untuk **memanfaatkan AI** dalam proses normalisasi MariaDB dari 1NF→2NF→3NF. Kalau mau, gua bisa langsung:
- **Generate** skrip migration + DDL jika kamu paste `SHOW CREATE TABLE` + ~10 contoh row dari tabel kamu, atau  
- **Buatkan skrip Python** ETL otomatis berdasarkan schema kamu.

Kirim DDL + sample data kalo mau gua bikin migration script nyata.
