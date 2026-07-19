# Materi Dasar Linux

Materi ini membahas dasar-dasar penggunaan Linux untuk kebutuhan DevOps, mulai dari navigasi terminal, manajemen paket, manajemen user, hingga akses remote menggunakan SSH.

---

## 1. Pengenalan Linux

Linux adalah sistem operasi open-source yang paling banyak digunakan untuk server. Hampir semua infrastruktur DevOps (web server, container, cloud) berjalan di atas Linux, sehingga menguasai perintah dasarnya adalah keterampilan wajib.

Beberapa istilah penting:

- **Terminal / Shell** — tempat mengetik perintah (umumnya menggunakan `bash` atau `zsh`).
- **Root** — user tertinggi yang memiliki akses penuh ke sistem.
- **`sudo`** — menjalankan perintah dengan hak akses root.
- **Distro** — varian Linux, contohnya Ubuntu, Debian, CentOS. Materi ini menggunakan Ubuntu/Debian.

---

## 2. Navigasi dan Manajemen File

Perintah paling dasar yang digunakan sehari-hari di terminal.

```bash
# Melihat lokasi direktori saat ini
pwd

# Melihat isi folder (termasuk file tersembunyi, dengan detail)
ls -la

# Masuk ke folder Documents
cd Documents

# Naik satu tingkat ke folder induk
cd ..

# Membuat folder baru
mkdir proyek

# Membuat file baru
touch catatan.txt

# Menyalin file
cp catatan.txt backup.txt

# Mengganti nama / memindahkan file
mv catatan.txt notes.txt

# Menghapus file
rm notes.txt

# Menghapus folder beserta isinya (hati-hati!)
rm -r proyek

# Menampilkan isi file
cat backup.txt

# Membaca file panjang per halaman
less backup.txt

# Mengedit file lewat terminal (pilih salah satu editor)
nano backup.txt
vim backup.txt
```

> ⚠️ **Hati-hati dengan `rm`** — file yang dihapus lewat terminal tidak masuk ke trash/recycle bin.

---

## 3. Struktur Direktori Linux

| Direktori | Fungsi |
|-----------|--------|
| `/` | Root, direktori paling atas |
| `/home` | Folder pribadi tiap user |
| `/etc` | File konfigurasi sistem dan aplikasi |
| `/var` | Data yang berubah-ubah (log, cache) |
| `/var/log` | Log sistem dan aplikasi |
| `/usr` | Aplikasi dan library milik sistem |
| `/tmp` | File sementara, terhapus saat reboot |

---

## 4. Manajemen Paket (APT)

Di Ubuntu/Debian, instalasi aplikasi dilakukan lewat **APT** (Advanced Package Tool).

```bash
# Update daftar paket dari repository
sudo apt update

# Upgrade semua paket ke versi terbaru
sudo apt upgrade

# Instal aplikasi (contoh: nginx)
sudo apt install nginx

# Hapus aplikasi
sudo apt remove nginx

# Mencari paket
apt search nginx
```

### Menambahkan Repository

Beberapa paket berada di repository tambahan yang harus diaktifkan dulu:

```bash
sudo add-apt-repository universe
sudo apt update
```

---

## 5. Manajemen User dan Hak Akses

Di server, sebaiknya tidak bekerja langsung sebagai root. Buat user baru dan beri akses `sudo`:

```bash
# Membuat user baru (akan diminta password dan data diri)
sudo adduser userbaru

# Menambahkan user ke grup sudo agar bisa menjalankan perintah admin
sudo usermod -aG sudo userbaru

# Berpindah ke user lain
su - userbaru

# Melihat user yang sedang aktif
whoami
```

### Hak Akses File (Permission)

Setiap file dan folder di Linux memiliki aturan siapa yang boleh membaca, menulis, dan mengeksekusinya. Permission bisa dilihat dengan `ls -l`:

```bash
ls -l
# -rwxr-xr--  1 userbaru developer  120 Jul 18 10:00 script.sh
```

Cara membaca `-rwxr-xr--`:

| Bagian | Nilai | Arti |
|--------|-------|------|
| Karakter 1 | `-` | Tipe: `-` file biasa, `d` direktori, `l` symlink |
| Karakter 2–4 | `rwx` | Hak **owner** (pemilik file): read, write, execute |
| Karakter 5–7 | `r-x` | Hak **group**: read dan execute, tanpa write |
| Karakter 8–10 | `r--` | Hak **others** (user lain): hanya read |

Arti tiap huruf:

- **r (read)** — membaca isi file / melihat isi direktori.
- **w (write)** — mengubah file / menambah dan menghapus isi direktori.
- **x (execute)** — menjalankan file sebagai program / masuk (`cd`) ke direktori.

#### Mengubah Permission dengan `chmod`

Ada dua cara penulisan: **simbolik** (huruf) dan **numerik** (angka oktal).

**Mode simbolik** — menggunakan `u` (owner), `g` (group), `o` (others), `a` (all):

```bash
# Menambahkan hak eksekusi untuk semua
chmod +x script.sh

# Menambahkan hak write untuk group
chmod g+w file.txt

# Menghapus hak read untuk others
chmod o-r rahasia.txt

# Owner boleh semuanya, others hanya read
chmod u=rwx,o=r file.txt
```

**Mode numerik** — tiap hak diberi nilai: `r = 4`, `w = 2`, `x = 1`, lalu dijumlahkan per posisi (owner, group, others):

| Angka | Permission | Arti |
|-------|-----------|------|
| 7 | `rwx` | 4+2+1: akses penuh |
| 6 | `rw-` | 4+2: baca dan tulis |
| 5 | `r-x` | 4+1: baca dan eksekusi |
| 4 | `r--` | hanya baca |
| 0 | `---` | tanpa akses |

```bash
# rwxr-xr-x — umum untuk script dan direktori
chmod 755 script.sh

# rw-r--r-- — umum untuk file biasa
chmod 644 file.txt

# rwx------ — hanya owner yang bisa akses
chmod 700 ~/.ssh

# rw------- — wajib untuk private key SSH
chmod 600 ~/.ssh/id_ed25519

# Mengubah permission folder beserta seluruh isinya
chmod -R 755 /var/www/html
```

#### Mengubah Kepemilikan dengan `chown`

```bash
# Mengubah pemilik file
sudo chown userbaru file.txt

# Mengubah pemilik sekaligus group (format: user:group)
sudo chown userbaru:developer file.txt

# Mengubah kepemilikan folder beserta isinya
sudo chown -R www-data:www-data /var/www/html

# Mengubah group saja
sudo chgrp developer file.txt
```

> 💡 **Contoh kasus nyata:** setelah upload file website ke `/var/www/html`, jalankan `sudo chown -R www-data:www-data /var/www/html` agar nginx (yang berjalan sebagai user `www-data`) bisa membaca file tersebut. Jika permission salah, biasanya muncul error **403 Forbidden**.

> ⚠️ Hindari `chmod 777` (semua user boleh segalanya) — ini celah keamanan. Gunakan permission seketat mungkin: `755` untuk direktori/script, `644` untuk file biasa, `600` untuk file sensitif seperti private key.

---

## 6. Remote Access dengan SSH

SSH (Secure Shell) digunakan untuk mengakses server dari jarak jauh secara aman. Semua komunikasi terenkripsi, sehingga aman digunakan lewat internet.

```bash
# Koneksi ke server
ssh user@ip-server

# Koneksi dengan port khusus (default SSH adalah port 22)
ssh -p 2222 user@ip-server
```

### Login Tanpa Password (SSH Key)

Login dengan password rawan ditebak (brute force). Cara yang lebih aman adalah **SSH key**: sepasang kunci yang terdiri dari **private key** (disimpan di laptop kita, tidak boleh dibagikan) dan **public key** (disalin ke server).

**Langkah 1 — Buat SSH key di laptop/komputer lokal:**

```bash
ssh-keygen -t ed25519 -C "asdardevs@gmail.com"
```

Tekan Enter untuk menerima lokasi default. Hasilnya dua file di folder `~/.ssh/`:

| File | Fungsi |
|------|--------|
| `~/.ssh/id_ed25519` | **Private key** — rahasia, jangan pernah dibagikan |
| `~/.ssh/id_ed25519.pub` | **Public key** — aman dibagikan, disalin ke server |

**Langkah 2 — Salin public key ke server:**

```bash
# Cara termudah (otomatis)
ssh-copy-id user@ip-server
```

Jika `ssh-copy-id` tidak tersedia, bisa manual:

```bash
# Tampilkan public key, lalu salin isinya
cat ~/.ssh/id_ed25519.pub

# Di server: tempelkan ke file authorized_keys
mkdir -p ~/.ssh
echo "isi-public-key" >> ~/.ssh/authorized_keys

# Pastikan permission-nya benar, kalau tidak SSH akan menolak
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

**Langkah 3 — Uji coba login:**

```bash
ssh user@ip-server
# Seharusnya langsung masuk tanpa diminta password
```

> 💡 Public key yang diizinkan login disimpan di server pada file `~/.ssh/authorized_keys`. Satu server bisa menyimpan banyak public key (satu baris per key), misalnya untuk beberapa anggota tim.

### Menonaktifkan Login Password (Hardening)

Setelah SSH key **terbukti berfungsi**, matikan login password agar server tidak bisa di-brute force:

```bash
# Edit konfigurasi SSH server
sudo nano /etc/ssh/sshd_config
```

Ubah baris berikut:

```
PasswordAuthentication no
PermitRootLogin no
```

Lalu restart service SSH:

```bash
sudo systemctl restart ssh
```

> ⚠️ **Jangan tutup sesi SSH yang sedang aktif** sebelum memastikan bisa login dari terminal baru dengan SSH key. Jika password sudah dimatikan tapi key bermasalah, kamu bisa terkunci dari server.

### SSH Config — Alias Koneksi

Daripada mengetik `ssh -p 2222 user@203.0.113.10` berulang kali, buat alias di file `~/.ssh/config` (di laptop):

```
Host server-belajar
    HostName 203.0.113.10
    User userbaru
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

Sekarang cukup:

```bash
ssh server-belajar
```

### Transfer File dengan SCP

SSH juga bisa dipakai untuk menyalin file antara lokal dan server:

```bash
# Upload file dari lokal ke server
scp file.txt user@ip-server:/home/user/

# Download file dari server ke lokal
scp user@ip-server:/var/log/nginx/access.log .

# Upload folder beserta isinya
scp -r proyek/ user@ip-server:/var/www/
```

---

## 7. Manajemen Service (systemd)

Aplikasi server (seperti nginx) berjalan sebagai **service** yang dikelola `systemctl`:

```bash
# Melihat status service
sudo systemctl status nginx

# Menjalankan / menghentikan / restart service
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx

# Menjalankan service otomatis saat boot
sudo systemctl enable nginx
```

---

## 8. Monitoring Sistem

```bash
# Melihat proses yang berjalan (CPU & RAM)
top

# Melihat penggunaan disk
df -h

# Melihat ukuran folder
du -sh /var/log

# Melihat penggunaan memori
free -h

# Melihat log secara real-time
tail -f /var/log/nginx/access.log
```

---

## Referensi

- Repository pembelajaran: https://github.com/DistritekDevOps/learning-devops
- Materi lanjutan Linux (menengah–mahir): lihat [linux-lengkap.md](linux-lengkap.md)
- Materi lanjutan web server (Apache & Nginx): lihat [nginx.md](nginx.md)
