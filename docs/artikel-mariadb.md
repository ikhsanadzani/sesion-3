# Pemanfaatan AI untuk Normalisasi Database (MariaDB 1NF → 2NF → 3NF)

## Ringkasan
- **Tujuan:** Menyusun database perpustakaan supaya bebas duplikasi & anomaly.  
- **Peran AI:** Membantu analisis schema awal (mentah), memberi rekomendasi tabel terstruktur, bikin DDL & skrip migrasi, plus query validasi.  
- **Prinsip:** Backup dulu, coba di staging, lalu deploy ke produksi.  

---

## 1. Gambaran Awal (Database Denormal)  
Misalnya kita punya satu tabel gabungan `perpustakaan_raw`:  

```sql
CREATE TABLE perpustakaan_raw (
  id INT AUTO_INCREMENT PRIMARY KEY,
  nama_anggota VARCHAR(200),
  alamat_anggota TEXT,
  no_hp VARCHAR(50),
  judul_buku VARCHAR(255),
  penulis VARCHAR(200),
  penerbit VARCHAR(200),
  tahun_terbit YEAR,
  tgl_pinjam DATE,
  tgl_kembali DATE
);
```

**Masalah:**
- Data anggota berulang tiap peminjaman.  
- Data buku berulang tiap dipinjam anggota berbeda.  
- Sulit tracking stok buku.  
- Tidak ada relasi yang jelas antar entitas.  

---

## 2. Normalisasi dengan Bantuan AI  

### Prompt untuk AI
Contoh instruksi ke ChatGPT/LLM:  

```
Aku punya tabel MariaDB bernama `perpustakaan_raw` (DDL di bawah) dengan beberapa contoh data. 
Tolong:
1. Jelaskan pelanggaran normalisasi (1NF/2NF/3NF).
2. Rekomendasikan tabel baru hasil normalisasi.
3. Buatkan DDL untuk setiap tabel (MariaDB).
4. Buat SQL migration untuk memindahkan data dari `perpustakaan_raw`.
5. Buat query validasi jumlah anggota, buku, dan transaksi.
```

---

## 3. Hasil Rekomendasi Normalisasi  

### Bentuk 3NF yang ideal:
- **anggota**: menyimpan data anggota perpustakaan.  
- **buku**: menyimpan koleksi buku.  
- **peminjaman**: menyimpan transaksi pinjam.  
- **detail_peminjaman** (opsional kalau satu transaksi bisa >1 buku).  

### DDL Contoh  

```sql
CREATE TABLE anggota (
  id_anggota INT AUTO_INCREMENT PRIMARY KEY,
  nama VARCHAR(200) NOT NULL,
  alamat TEXT,
  no_hp VARCHAR(50)
) ENGINE=InnoDB;

CREATE TABLE buku (
  id_buku INT AUTO_INCREMENT PRIMARY KEY,
  judul VARCHAR(255) NOT NULL,
  penulis VARCHAR(200),
  penerbit VARCHAR(200),
  tahun YEAR,
  stok INT DEFAULT 1
) ENGINE=InnoDB;

CREATE TABLE peminjaman (
  id_peminjaman INT AUTO_INCREMENT PRIMARY KEY,
  id_anggota INT NOT NULL,
  tgl_pinjam DATE,
  tgl_kembali DATE,
  FOREIGN KEY (id_anggota) REFERENCES anggota(id_anggota)
) ENGINE=InnoDB;

CREATE TABLE detail_peminjaman (
  id_detail INT AUTO_INCREMENT PRIMARY KEY,
  id_peminjaman INT NOT NULL,
  id_buku INT NOT NULL,
  FOREIGN KEY (id_peminjaman) REFERENCES peminjaman(id_peminjaman),
  FOREIGN KEY (id_buku) REFERENCES buku(id_buku)
) ENGINE=InnoDB;
```

---

## 4. Migrasi Data dari `perpustakaan_raw`  

1. **Isi tabel anggota:**
```sql
INSERT INTO anggota (nama, alamat, no_hp)
SELECT DISTINCT nama_anggota, alamat_anggota, no_hp
FROM perpustakaan_raw;
```

2. **Isi tabel buku:**
```sql
INSERT INTO buku (judul, penulis, penerbit, tahun, stok)
SELECT DISTINCT judul_buku, penulis, penerbit, tahun_terbit, 1
FROM perpustakaan_raw;
```

3. **Isi tabel peminjaman:**
```sql
INSERT INTO peminjaman (id_anggota, tgl_pinjam, tgl_kembali)
SELECT (SELECT id_anggota FROM anggota a WHERE a.nama = pr.nama_anggota LIMIT 1),
       pr.tgl_pinjam, pr.tgl_kembali
FROM perpustakaan_raw pr;
```

4. **Isi detail_peminjaman:**
```sql
INSERT INTO detail_peminjaman (id_peminjaman, id_buku)
SELECT p.id_peminjaman,
       (SELECT id_buku FROM buku b WHERE b.judul = pr.judul_buku LIMIT 1)
FROM perpustakaan_raw pr
JOIN peminjaman p ON p.tgl_pinjam = pr.tgl_pinjam AND p.tgl_kembali = pr.tgl_kembali;
```

---

## 5. Validasi Hasil  

- Jumlah anggota:
```sql
SELECT COUNT(*) FROM anggota;
```

- Jumlah buku:
```sql
SELECT COUNT(*) FROM buku;
```

- Jumlah transaksi peminjaman:
```sql
SELECT COUNT(*) FROM peminjaman;
```

- Cek konsistensi peminjaman:
```sql
SELECT COUNT(*) FROM detail_peminjaman;
```

---

## 6. Best Practices  
- Selalu backup sebelum migrasi (`mysqldump`).  
- Uji di staging, bukan langsung di produksi.  
- Gunakan transaksi saat migrasi.  
- Dokumentasikan mapping antar tabel.  

---

## 7. Penutup  
Dengan bantuan AI, proses normalisasi database perpustakaan jadi lebih cepat, karena AI bisa langsung memberikan:  
- Identifikasi pelanggaran normalisasi,  
- Skema 1NF → 2NF → 3NF,  
- Skrip SQL siap eksekusi,  
- Query validasi.  

Hasilnya, database lebih rapi, efisien, dan mudah dikembangkan (misalnya menambah modul denda, laporan bulanan, dsb).  

