# Materi MySQL: Pemula sampai Mahir

MySQL adalah salah satu Relational Database Management System (RDBMS) paling populer, banyak dipakai untuk menyimpan data aplikasi web. Materi ini membahas dari instalasi, dasar SQL, sampai optimasi dan administrasi untuk kebutuhan DevOps.

---

## 1. Pengenalan MySQL

- **Database** — kumpulan data yang terorganisir.
- **Table** — struktur data berbentuk baris (row) dan kolom (column), mirip spreadsheet.
- **Row (record)** — satu entri data dalam table.
- **Column (field)** — atribut/kolom data, punya tipe data tertentu (contoh: `INT`, `VARCHAR`).
- **Primary Key (PK)** — kolom unik yang mengidentifikasi tiap row.
- **Foreign Key (FK)** — kolom yang mereferensikan primary key di table lain, untuk membuat relasi antar table.
- **SQL** — bahasa query untuk berinteraksi dengan database (Structured Query Language).

---

## 2. Instalasi dan Akses Dasar

```bash
# Instalasi di Ubuntu/Debian
sudo apt update
sudo apt install mysql-server

# Amankan instalasi (set password root, hapus user anonim, dll)
sudo mysql_secure_installation

# Cek status service
sudo systemctl status mysql

# Login ke MySQL sebagai root
sudo mysql -u root -p
```

```bash
# Login dari luar (jika sudah ada user dan password)
mysql -u namauser -p -h ip-server
```

---

## 3. Manajemen Database

```sql
-- Melihat semua database
SHOW DATABASES;

-- Membuat database baru
CREATE DATABASE toko_online;

-- Menggunakan (masuk ke) database tertentu
USE toko_online;

-- Menghapus database
DROP DATABASE toko_online;
```

---

## 4. Membuat dan Mengatur Table

```sql
-- Membuat table
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nama VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    umur INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Melihat semua table di database aktif
SHOW TABLES;

-- Melihat struktur table
DESCRIBE users;
-- atau
SHOW COLUMNS FROM users;

-- Mengubah struktur table
ALTER TABLE users ADD COLUMN alamat VARCHAR(255);
ALTER TABLE users MODIFY COLUMN umur SMALLINT;
ALTER TABLE users DROP COLUMN alamat;
ALTER TABLE users RENAME COLUMN nama TO nama_lengkap;

-- Menghapus table
DROP TABLE users;

-- Mengosongkan isi table tapi struktur tetap ada
TRUNCATE TABLE users;
```

### Tipe Data Umum

| Tipe | Fungsi |
|------|--------|
| `INT` | Angka bulat |
| `DECIMAL(10,2)` | Angka desimal presisi (contoh: harga) |
| `VARCHAR(n)` | Teks pendek, panjang maksimal `n` karakter |
| `TEXT` | Teks panjang |
| `DATE` | Tanggal (`YYYY-MM-DD`) |
| `DATETIME` / `TIMESTAMP` | Tanggal dan waktu |
| `BOOLEAN` | Nilai true/false (disimpan sebagai `TINYINT`) |
| `ENUM('a','b','c')` | Nilai terbatas dari daftar pilihan |

---

## 5. CRUD Dasar (Create, Read, Update, Delete)

### Create — Menambah Data

```sql
INSERT INTO users (nama, email, umur) VALUES ('Reza', 'reza@mail.com', 25);

-- Menambah beberapa data sekaligus
INSERT INTO users (nama, email, umur) VALUES
    ('Andi', 'andi@mail.com', 30),
    ('Sari', 'sari@mail.com', 22);
```

### Read — Membaca Data

```sql
-- Ambil semua data
SELECT * FROM users;

-- Ambil kolom tertentu
SELECT nama, email FROM users;

-- Filter dengan WHERE
SELECT * FROM users WHERE umur > 25;

-- Urutkan hasil
SELECT * FROM users ORDER BY umur DESC;

-- Batasi jumlah hasil
SELECT * FROM users LIMIT 5;

-- Kombinasi: pagination
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 20;
```

### Update — Mengubah Data

```sql
UPDATE users SET umur = 26 WHERE nama = 'Reza';

-- ⚠️ Tanpa WHERE, semua row akan berubah!
```

### Delete — Menghapus Data

```sql
DELETE FROM users WHERE id = 3;

-- ⚠️ Tanpa WHERE, semua row akan terhapus!
```

---

## 6. Filtering dan Operator

```sql
-- Operator perbandingan
SELECT * FROM users WHERE umur >= 18 AND umur <= 30;

-- IN — cocok dengan salah satu nilai
SELECT * FROM users WHERE nama IN ('Reza', 'Andi');

-- BETWEEN — rentang nilai
SELECT * FROM users WHERE umur BETWEEN 20 AND 30;

-- LIKE — pencarian pola teks
SELECT * FROM users WHERE email LIKE '%@gmail.com';
SELECT * FROM users WHERE nama LIKE 'A%';   -- diawali huruf A

-- IS NULL / IS NOT NULL
SELECT * FROM users WHERE alamat IS NULL;

-- Kombinasi AND / OR
SELECT * FROM users WHERE umur > 20 AND (nama = 'Reza' OR nama = 'Andi');
```

---

## 7. Fungsi Agregat dan GROUP BY

```sql
-- Menghitung jumlah row
SELECT COUNT(*) FROM users;

-- Rata-rata, total, min, max
SELECT AVG(umur), SUM(umur), MIN(umur), MAX(umur) FROM users;

-- Mengelompokkan data
SELECT umur, COUNT(*) AS jumlah
FROM users
GROUP BY umur;

-- Filter hasil GROUP BY dengan HAVING (WHERE tidak bisa dipakai di sini)
SELECT umur, COUNT(*) AS jumlah
FROM users
GROUP BY umur
HAVING jumlah > 1;
```

> 💡 `WHERE` menyaring row sebelum dikelompokkan, `HAVING` menyaring hasil setelah `GROUP BY`.

---

## 8. Relasi Antar Table (JOIN)

Contoh dua table yang berelasi:

```sql
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    produk VARCHAR(100),
    total DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### Jenis JOIN

```sql
-- INNER JOIN — hanya data yang punya pasangan di kedua table
SELECT users.nama, orders.produk, orders.total
FROM orders
INNER JOIN users ON orders.user_id = users.id;

-- LEFT JOIN — semua data dari table kiri, meski tidak punya pasangan
SELECT users.nama, orders.produk
FROM users
LEFT JOIN orders ON users.id = orders.user_id;

-- RIGHT JOIN — semua data dari table kanan
SELECT users.nama, orders.produk
FROM orders
RIGHT JOIN users ON orders.user_id = users.id;
```

| JOIN | Hasil |
|------|-------|
| `INNER JOIN` | Hanya row yang cocok di kedua table |
| `LEFT JOIN` | Semua row table kiri + yang cocok dari kanan (NULL jika tidak ada) |
| `RIGHT JOIN` | Semua row table kanan + yang cocok dari kiri (NULL jika tidak ada) |

---

## 9. Index — Mempercepat Query

Index membuat pencarian jauh lebih cepat, terutama pada table besar, dengan trade-off sedikit lebih lambat saat insert/update.

```sql
-- Membuat index pada kolom yang sering dipakai di WHERE/JOIN
CREATE INDEX idx_email ON users(email);

-- Melihat index yang ada
SHOW INDEX FROM users;

-- Menghapus index
DROP INDEX idx_email ON users;

-- Menganalisis apakah query menggunakan index
EXPLAIN SELECT * FROM users WHERE email = 'reza@mail.com';
```

> 💡 `PRIMARY KEY` dan `UNIQUE` otomatis membuat index. Buat index tambahan pada kolom yang sering dipakai untuk `WHERE`, `JOIN`, atau `ORDER BY` — tapi jangan berlebihan karena tiap index menambah beban saat insert/update/delete.

---

## 10. Transaction — Menjaga Konsistensi Data

Transaction memastikan sekumpulan query berjalan sebagai satu unit: semua berhasil, atau semua dibatalkan (rollback).

```sql
START TRANSACTION;

UPDATE users SET umur = umur - 1 WHERE id = 1;
UPDATE users SET umur = umur + 1 WHERE id = 2;

-- Jika semua query berhasil dan sesuai harapan
COMMIT;

-- Jika terjadi kesalahan, batalkan semua perubahan
ROLLBACK;
```

**Contoh kasus nyata:** transfer saldo antar akun — saldo pengirim berkurang dan saldo penerima bertambah harus terjadi bersamaan. Jika salah satu gagal, keduanya harus dibatalkan agar data tidak rusak.

---

## 11. User Management dan Privileges

```sql
-- Membuat user baru
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'password_kuat';

-- Membuat user yang bisa akses dari IP manapun
CREATE USER 'appuser'@'%' IDENTIFIED BY 'password_kuat';

-- Memberi hak akses ke database tertentu
GRANT SELECT, INSERT, UPDATE, DELETE ON toko_online.* TO 'appuser'@'localhost';

-- Memberi semua hak akses ke satu database (hati-hati!)
GRANT ALL PRIVILEGES ON toko_online.* TO 'appuser'@'localhost';

-- Menerapkan perubahan privilege
FLUSH PRIVILEGES;

-- Melihat privilege user
SHOW GRANTS FOR 'appuser'@'localhost';

-- Mencabut hak akses
REVOKE INSERT ON toko_online.* FROM 'appuser'@'localhost';

-- Menghapus user
DROP USER 'appuser'@'localhost';
```

> ⚠️ Jangan gunakan user `root` untuk aplikasi. Buat user khusus dengan privilege seminimal mungkin sesuai kebutuhan aplikasi (prinsip *least privilege*).

### User dengan Akses ke Semua Database

Kadang dibutuhkan user "admin" yang bisa mengakses semua database di server (misalnya untuk tool manajemen seperti phpMyAdmin/Adminer, atau user untuk keperluan backup). Gunakan `*.*` untuk menargetkan seluruh database dan table:

```sql
-- Membuat user
CREATE USER 'adminuser'@'localhost' IDENTIFIED BY 'password_kuat';

-- Memberi semua hak akses ke SEMUA database (setara root, gunakan dengan sangat hati-hati)
GRANT ALL PRIVILEGES ON *.* TO 'adminuser'@'localhost' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

| Target | Cakupan |
|--------|---------|
| `toko_online.*` | Semua table di database `toko_online` saja |
| `toko_online.users` | Hanya table `users` di database `toko_online` |
| `*.*` | Semua table di semua database (akses setara root) |

`WITH GRANT OPTION` memberi izin agar user tersebut juga bisa memberikan (`GRANT`) privilege ke user lain. Biasanya hanya dipakai untuk user admin, bukan user aplikasi biasa.

> ⚠️ `GRANT ALL PRIVILEGES ON *.*` setara dengan membuat root baru. Hanya gunakan untuk keperluan administrasi (DBA, tool monitoring/backup), jangan untuk user yang dipakai aplikasi sehari-hari.

Kalau tujuannya sekadar **backup semua database**, cukup beri privilege yang relevan saja, tanpa `ALL PRIVILEGES`:

```sql
CREATE USER 'backupuser'@'localhost' IDENTIFIED BY 'password_kuat';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER ON *.* TO 'backupuser'@'localhost';
FLUSH PRIVILEGES;
```

### Mengubah Password User

```sql
-- Cara modern (MySQL 5.7.6+ / 8.0), user mengubah password sendiri
ALTER USER 'appuser'@'localhost' IDENTIFIED BY 'password_baru';

-- Mengubah password root
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password_baru';

-- Menerapkan perubahan
FLUSH PRIVILEGES;
```

```bash
# Mengubah password lewat command line tanpa masuk ke prompt MySQL
mysqladmin -u root -p password 'password_baru'

# Jika lupa password root, reset lewat safe mode:
sudo systemctl stop mysql
sudo mysqld_safe --skip-grant-tables &
mysql -u root
```

```sql
-- Di dalam safe mode, jalankan:
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password_baru';
```

```bash
# Setelah selesai, keluar dan restart mysql secara normal
sudo systemctl restart mysql
```

> 💡 Setelah mengubah password user yang dipakai aplikasi, jangan lupa update juga file `.env` atau connection string aplikasi terkait — kalau tidak, aplikasi akan gagal connect ke database.

---

## 12. Backup dan Restore

```bash
# Backup satu database ke file .sql
mysqldump -u root -p toko_online > backup_toko.sql

# Backup semua database
mysqldump -u root -p --all-databases > backup_semua.sql

# Restore dari file backup
mysql -u root -p toko_online < backup_toko.sql

# Backup terkompresi (untuk database besar)
mysqldump -u root -p toko_online | gzip > backup_toko.sql.gz

# Restore dari file terkompresi
gunzip < backup_toko.sql.gz | mysql -u root -p toko_online
```

> 💡 Untuk server production, jadwalkan backup otomatis dengan `cron` dan simpan hasilnya di lokasi terpisah (misalnya cloud storage), bukan hanya di server yang sama.

---

## 13. Monitoring dan Troubleshooting

```sql
-- Melihat proses/query yang sedang berjalan
SHOW PROCESSLIST;

-- Menghentikan query yang macet
KILL <process_id>;

-- Melihat status server
SHOW STATUS;

-- Melihat variabel konfigurasi
SHOW VARIABLES LIKE 'max_connections';
```

```bash
# Melihat log error MySQL
sudo tail -f /var/log/mysql/error.log

# Lokasi file konfigurasi utama
/etc/mysql/mysql.conf.d/mysqld.cnf
```

### Query Lambat (Slow Query)

```sql
-- Aktifkan log untuk query yang lambat
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;  -- catat query di atas 2 detik
```

Gunakan `EXPLAIN` di depan query untuk menganalisis kenapa query lambat (lihat bagian 9).

---

## 14. Optimasi Lanjutan

- **Normalisasi** — memecah data ke beberapa table untuk menghindari duplikasi (misal: pisahkan `users` dan `orders`, bukan digabung jadi satu table besar).
- **Denormalisasi** — kadang sengaja menggabungkan data untuk mempercepat read pada aplikasi dengan traffic tinggi, dengan trade-off kompleksitas update.
- **Connection Pooling** — aplikasi sebaiknya memakai connection pool, bukan membuka koneksi baru tiap query, agar tidak membebani `max_connections`.
- **EXPLAIN ANALYZE** — versi lebih detail dari `EXPLAIN`, menunjukkan waktu eksekusi aktual tiap langkah query.
- **Partitioning** — membagi table besar menjadi beberapa partisi fisik (misal per tahun) agar query lebih cepat pada data historis besar.

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 5;
```

---

## 15. View — Query Tersimpan

View adalah "table virtual" hasil dari query yang disimpan, berguna untuk menyederhanakan query kompleks yang sering dipakai berulang.

```sql
CREATE VIEW ringkasan_order AS
SELECT users.nama, COUNT(orders.id) AS total_order, SUM(orders.total) AS total_belanja
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.id;

-- Dipakai seperti table biasa
SELECT * FROM ringkasan_order WHERE total_belanja > 1000000;

-- Menghapus view
DROP VIEW ringkasan_order;
```

> 💡 View tidak menyimpan data sendiri — setiap kali dipanggil, query di baliknya dijalankan ulang terhadap data terbaru.

---

## 16. Stored Procedure, Function, dan Trigger

### Stored Procedure

Kumpulan query yang disimpan di database dan bisa dipanggil berulang, berguna untuk logika yang sering dipakai di banyak tempat.

```sql
DELIMITER $$

CREATE PROCEDURE TambahUser(IN p_nama VARCHAR(100), IN p_email VARCHAR(100))
BEGIN
    INSERT INTO users (nama, email) VALUES (p_nama, p_email);
END $$

DELIMITER ;

-- Memanggil procedure
CALL TambahUser('Budi', 'budi@mail.com');

-- Menghapus procedure
DROP PROCEDURE TambahUser;
```

### Trigger — Aksi Otomatis Saat Data Berubah

Trigger menjalankan aksi otomatis saat `INSERT`, `UPDATE`, atau `DELETE` terjadi pada table tertentu.

```sql
CREATE TABLE log_perubahan_umur (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    umur_lama INT,
    umur_baru INT,
    waktu TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

DELIMITER $$

CREATE TRIGGER catat_perubahan_umur
BEFORE UPDATE ON users
FOR EACH ROW
BEGIN
    IF OLD.umur <> NEW.umur THEN
        INSERT INTO log_perubahan_umur (user_id, umur_lama, umur_baru)
        VALUES (OLD.id, OLD.umur, NEW.umur);
    END IF;
END $$

DELIMITER ;
```

`OLD` merujuk ke nilai sebelum perubahan, `NEW` ke nilai sesudahnya. Trigger bisa `BEFORE`/`AFTER` dikombinasikan dengan `INSERT`/`UPDATE`/`DELETE`.

> ⚠️ Gunakan trigger secukupnya — logika yang tersembunyi di trigger membuat aplikasi lebih sulit di-debug karena perubahan data bisa terjadi "diam-diam" tanpa terlihat di kode aplikasi.

---

## 17. Tipe Data JSON

MySQL (5.7+) mendukung kolom bertipe `JSON`, berguna untuk data semi-terstruktur yang skemanya sering berubah.

```sql
CREATE TABLE produk (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nama VARCHAR(100),
    atribut JSON
);

INSERT INTO produk (nama, atribut)
VALUES ('Kaos', '{"warna": "merah", "ukuran": ["S", "M", "L"]}');

-- Mengambil nilai tertentu dari JSON dengan ->>
SELECT nama, atribut->>'$.warna' AS warna FROM produk;

-- Filter berdasarkan isi JSON
SELECT * FROM produk WHERE atribut->>'$.warna' = 'merah';

-- Update sebagian isi JSON
UPDATE produk SET atribut = JSON_SET(atribut, '$.warna', 'biru') WHERE id = 1;
```

> 💡 Kolom JSON praktis untuk data fleksibel, tapi query di dalamnya lebih lambat dibanding kolom biasa dan tidak bisa di-index langsung seperti kolom normal (perlu *generated column* + index jika butuh performa). Gunakan hanya kalau strukturnya memang tidak bisa dinormalisasi ke table biasa.

---

## 18. Replication — Master-Replica

Replication menyalin data dari satu server (**master/source**) ke satu atau lebih server (**replica**) secara real-time, berguna untuk backup live, membagi beban baca (read scaling), dan high availability.

### Langkah Ringkas Setup

**Di server master**, aktifkan binary log di `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```ini
[mysqld]
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_do_db = toko_online
```

```sql
-- Buat user khusus replikasi
CREATE USER 'replika'@'%' IDENTIFIED BY 'password_kuat';
GRANT REPLICATION SLAVE ON *.* TO 'replika'@'%';
FLUSH PRIVILEGES;

-- Catat posisi log saat ini (dipakai saat setup replica)
SHOW MASTER STATUS;
```

**Di server replica**, set `server-id` berbeda, lalu:

```sql
CHANGE MASTER TO
    MASTER_HOST='ip-server-master',
    MASTER_USER='replika',
    MASTER_PASSWORD='password_kuat',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=1234;

START SLAVE;

-- Cek status, pastikan Slave_IO_Running & Slave_SQL_Running = Yes
SHOW SLAVE STATUS\G
```

> 💡 Replica cocok untuk memisahkan beban: tulis (`INSERT`/`UPDATE`/`DELETE`) ke master, baca (`SELECT`) sebagian dialihkan ke replica — mengurangi beban server utama pada aplikasi dengan traffic baca tinggi.

---

## 19. Keamanan Database (Hardening)

Melengkapi dasar `mysql_secure_installation` dan *least privilege* di bagian 11.

```sql
-- Jangan pernah izinkan root login dari luar (%), hanya localhost
-- Cek user yang ada dan host akses masing-masing
SELECT user, host FROM mysql.user;

-- Hapus user anonim jika masih ada
DROP USER ''@'localhost';

-- Hapus database test bawaan
DROP DATABASE IF EXISTS test;
```

Praktik keamanan lain:

- **Jangan expose port 3306 ke publik** — batasi dengan firewall (`ufw`, lihat [linux-lengkap.md](linux-lengkap.md) bagian keamanan) hanya untuk IP aplikasi/backend yang membutuhkan.
- **Gunakan SSL/TLS untuk koneksi** jika aplikasi dan database berada di server berbeda, agar kredensial dan data tidak lewat sebagai plain text di jaringan.
- **Enkripsi kolom sensitif** (misalnya nomor kartu, NIK) di level aplikasi sebelum disimpan, jangan mengandalkan enkripsi database saja.
- **Backup terenkripsi** — file hasil `mysqldump` berisi seluruh data mentah; simpan di lokasi aman dan pertimbangkan enkripsi (`gpg`) sebelum diunggah ke cloud storage.
- **Audit log** — aktifkan `general_log` sementara saat investigasi masalah, tapi matikan lagi di produksi karena berat untuk I/O.

---

## 20. Cheat Sheet Ringkas

| Kategori | Perintah |
|----------|----------|
| Database | `CREATE DATABASE`, `USE`, `DROP DATABASE` |
| Table | `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`, `DESCRIBE` |
| Data | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| Filter | `WHERE`, `LIKE`, `IN`, `BETWEEN`, `IS NULL` |
| Agregat | `COUNT`, `SUM`, `AVG`, `GROUP BY`, `HAVING` |
| Relasi | `INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN` |
| Performa | `CREATE INDEX`, `EXPLAIN` |
| Transaksi | `START TRANSACTION`, `COMMIT`, `ROLLBACK` |
| User | `CREATE USER`, `GRANT`, `REVOKE` |
| Backup | `mysqldump`, redirect `<` untuk restore |
| View | `CREATE VIEW`, `DROP VIEW` |
| Procedure/Trigger | `CREATE PROCEDURE`, `CALL`, `CREATE TRIGGER` |
| JSON | `->>`, `JSON_SET` |
| Replication | `CHANGE MASTER TO`, `START SLAVE`, `SHOW SLAVE STATUS` |

---

## Referensi

- Dokumentasi resmi: https://dev.mysql.com/doc/
- Repository pembelajaran: https://github.com/DistritekDevOps/learning-devops
