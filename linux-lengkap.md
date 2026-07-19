# Materi Linux Lengkap — Dasar sampai Mahir

Materi ini adalah kelanjutan dari [Materi Dasar Linux](linux.md), disusun bertingkat dari dasar sampai mahir. Cakupannya mengikuti kompetensi sertifikasi **LFCS (Linux Foundation Certified System Administrator)** dan kebutuhan nyata pekerjaan DevOps: text processing, manajemen proses, networking, rsync, cron, bash scripting, systemd, keamanan server, storage/LVM, sampai konsep di balik container.

**Peta jalur belajar:**

| Level | Bagian | Topik |
|-------|--------|-------|
| Dasar | 1 | Ringkasan materi dasar (prasyarat) |
| Menengah | 2–9 | Pipe & text processing, proses, networking, rsync, backup, cron, environment, tmux |
| Mahir | 10–16 | Bash scripting, systemd service sendiri, keamanan, storage & LVM, log, troubleshooting, namespaces & cgroups |

---

# LEVEL DASAR

## 1. Ringkasan Materi Dasar (Prasyarat)

Sebelum masuk ke sini, pastikan sudah menguasai [Materi Dasar Linux](linux.md) yang mencakup:

| Topik | Perintah kunci |
|-------|----------------|
| Navigasi & file | `pwd`, `ls`, `cd`, `mkdir`, `cp`, `mv`, `rm`, `cat`, `nano` |
| Struktur direktori | `/etc`, `/var/log`, `/home`, `/tmp` |
| Manajemen paket | `apt update`, `apt install`, `apt remove` |
| User & permission | `adduser`, `usermod -aG sudo`, `chmod`, `chown` |
| SSH | `ssh`, `ssh-keygen`, `ssh-copy-id`, `scp`, hardening |
| Service | `systemctl start/stop/status/enable` |
| Monitoring dasar | `top`, `df -h`, `free -h`, `tail -f` |

---

# LEVEL MENENGAH

## 2. Pipe, Redirection & Text Processing

Kekuatan terbesar shell Linux: perintah-perintah kecil bisa **dirangkai** menjadi satu alur kerja. Ini konsep yang dipakai di semua bagian selanjutnya.

### Redirection — Mengarahkan Output

Setiap program punya tiga jalur data: **stdin** (input), **stdout** (output normal), dan **stderr** (output error).

```bash
# > menulis output ke file (menimpa isi lama)
ls -la > daftar.txt

# >> menambahkan ke akhir file (tidak menimpa)
echo "baris baru" >> catatan.txt

# 2> mengarahkan error saja
perintah-salah 2> error.log

# Output dan error sekaligus ke satu file
perintah > semua.log 2>&1

# Membuang output (lubang hitam)
perintah > /dev/null 2>&1
```

### Pipe — Merangkai Perintah

`|` mengalirkan output perintah kiri menjadi input perintah kanan:

```bash
# Berapa banyak file di folder ini?
ls | wc -l

# Cari proses nginx
ps aux | grep nginx

# 10 baris log terakhir yang mengandung kata "error"
tail -100 /var/log/nginx/error.log | grep error

# Simpan sekaligus tampilkan (tee)
ls -la | tee daftar.txt
```

### grep — Mencari Teks

Perintah yang paling sering dipakai saat debugging:

```bash
# Cari kata di dalam file
grep "error" /var/log/nginx/error.log

# -i: abaikan besar-kecil huruf, -n: tampilkan nomor baris
grep -in "failed" auth.log

# -r: cari di seluruh folder (rekursif)
grep -r "DB_HOST" /var/www/

# -v: baris yang TIDAK mengandung kata
grep -v "^#" /etc/ssh/sshd_config    # sembunyikan komentar

# -c: hitung jumlah baris yang cocok
grep -c "404" access.log

# -E: regex (extended) — cari "error" ATAU "warning"
grep -E "error|warning" app.log
```

### sed, awk, cut, sort — Mengolah Teks

```bash
# sed: ganti teks (s = substitute, g = semua kemunculan)
sed 's/lama/baru/g' file.txt              # tampilkan hasilnya
sed -i 's/port 3000/port 8080/g' app.conf # -i: langsung ubah file

# awk: ambil kolom tertentu (default dipisah spasi)
ps aux | awk '{print $2, $11}'            # kolom 2 (PID) dan 11 (perintah)
df -h | awk '{print $5, $6}'              # persentase pakai + mount point

# cut: potong berdasarkan pemisah
cut -d: -f1 /etc/passwd                   # kolom 1, pemisah ":" = daftar user

# sort + uniq: urutkan lalu hitung duplikat
sort akses.txt | uniq -c | sort -rn       # hitung & urutkan terbanyak
```

**Contoh nyata gabungan** — 10 alamat IP yang paling sering mengakses web server:

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

### find — Mencari File

```bash
# Cari file berdasarkan nama
find /var/www -name "*.php"

# File lebih besar dari 100 MB (untuk cari pemakan disk)
find / -type f -size +100M 2>/dev/null

# File yang berubah dalam 1 hari terakhir
find /etc -mtime -1

# Cari lalu jalankan perintah pada tiap hasil
find /var/log -name "*.log" -mtime +30 -delete   # hapus log > 30 hari
```

---

## 3. Manajemen Proses & Job

`top` sudah dikenal di materi dasar — sekarang cara **mengendalikan** proses.

### Melihat & Mencari Proses

```bash
# Semua proses, semua user
ps aux

# Cari PID proses berdasarkan nama
ps aux | grep nginx
pgrep nginx              # langsung tampilkan PID-nya saja

# Pohon proses (siapa anak siapa)
ps auxf
```

### Menghentikan Proses — kill & Sinyal

`kill` sebenarnya *mengirim sinyal*, bukan sekadar membunuh:

```bash
# SIGTERM (15) — minta berhenti baik-baik (default, coba ini dulu)
kill 1234

# SIGKILL (9) — paksa mati seketika (jalan terakhir)
kill -9 1234

# Berdasarkan nama, tanpa cari PID dulu
pkill nginx
killall node
```

> 💡 Selalu coba `kill` biasa dulu. `kill -9` tidak memberi kesempatan proses menutup file/koneksi dengan rapi — bisa menyebabkan data korup.

### Kasus Nyata: "Port Sudah Dipakai"

```bash
# Siapa yang memakai port 80?
sudo lsof -i :80
# atau
sudo ss -tulpn | grep :80

# Matikan prosesnya
sudo kill <PID>
```

### Job Control — Proses di Latar Belakang

```bash
# Jalankan langsung di background dengan &
python3 server.py &

# Ctrl+Z: hentikan sementara proses yang sedang jalan
# lalu lanjutkan di background:
bg

# Daftar job di sesi ini, dan kembalikan ke depan
jobs
fg %1

# nohup: proses tetap hidup walau terminal ditutup
nohup python3 server.py > app.log 2>&1 &
```

> 💡 Untuk aplikasi produksi, jangan andalkan `nohup` — gunakan systemd service (bagian 11) agar otomatis restart dan tercatat log-nya.

---

## 4. Networking Praktis

Domain terbesar di ujian LFCS (25%) dan makanan sehari-hari DevOps.

### Identitas & Alamat Jaringan

```bash
# Alamat IP semua interface
ip addr        # atau singkat: ip a

# Tabel routing (lewat mana paket keluar)
ip route

# Nama host
hostnamectl

# IP publik server (dilihat dari luar)
curl ifconfig.me
```

### Memeriksa Port & Koneksi

```bash
# Port apa saja yang sedang listening, oleh proses apa
sudo ss -tulpn

# t=TCP u=UDP l=listening p=proses n=numerik
# Contoh output: nginx listen di 0.0.0.0:80

# Apakah port tertentu terbuka?
sudo ss -tulpn | grep :443
```

### Menguji Konektivitas

```bash
# Apakah host bisa dijangkau?
ping -c 4 google.com

# Di mana putusnya jalur ke server?
traceroute 8.8.8.8      # atau mtr 8.8.8.8 (real-time)

# Cek resolusi DNS
dig example.com          # detail lengkap
nslookup example.com     # versi ringkas

# Uji HTTP endpoint
curl -I https://example.com     # header saja (cek status code)
curl -L http://example.com      # ikuti redirect
curl -o file.zip https://situs/file.zip   # download
```

### Resolusi Nama Lokal

```bash
# Mapping nama -> IP secara manual (sebelum sampai ke DNS)
sudo nano /etc/hosts
```

```
127.0.0.1    localhost
203.0.113.10 server-belajar.local
```

### Sinkronisasi Waktu

Waktu server yang meleset merusak log, sertifikat TLS, dan job terjadwal:

```bash
timedatectl                          # cek status & zona waktu
sudo timedatectl set-timezone Asia/Makassar
sudo timedatectl set-ntp true        # aktifkan sinkronisasi otomatis
```

---

## 5. Transfer & Sinkronisasi File — rsync

`scp` (materi dasar) menyalin semua file dari nol setiap kali dijalankan. **rsync** lebih pintar: hanya mengirim **perubahan** (delta), bisa melanjutkan transfer yang putus, dan bisa menghapus file yang sudah tidak ada di sumber. Standar de-facto untuk backup dan deploy.

```bash
# Sinkronkan folder lokal ke server
# -a: archive (permission, timestamp, rekursif ikut)
# -v: verbose, -z: kompresi saat transfer
rsync -avz proyek/ user@ip-server:/var/www/proyek/

# Arah sebaliknya: server -> lokal
rsync -avz user@ip-server:/var/log/nginx/ ./log-backup/

# --delete: hapus file di tujuan yang sudah tidak ada di sumber
# (membuat tujuan benar-benar identik — hati-hati!)
rsync -avz --delete proyek/ user@ip-server:/var/www/proyek/

# --exclude: jangan ikutkan file/folder tertentu
rsync -avz --exclude "node_modules" --exclude ".git" \
  proyek/ user@ip-server:/var/www/proyek/

# -n / --dry-run: simulasi dulu, tampilkan apa yang AKAN terjadi
rsync -avzn --delete proyek/ user@ip-server:/var/www/proyek/
```

> ⚠️ **Perhatikan garis miring di akhir.** `proyek/` berarti "isi folder proyek", sedangkan `proyek` (tanpa `/`) berarti "folder proyek itu sendiri". Salah satu sumber bug rsync paling umum.

> 💡 Selalu jalankan `--dry-run` dulu sebelum memakai `--delete` — sekali salah arah, file di tujuan bisa terhapus massal.

---

## 6. Arsip, Kompresi & Backup

### tar — Membungkus dan Membuka

```bash
# Membuat arsip terkompresi (c=create, z=gzip, f=file)
tar czf backup.tar.gz /var/www/html

# Membuka arsip (x=extract)
tar xzf backup.tar.gz

# Membuka ke folder tertentu
tar xzf backup.tar.gz -C /tmp/restore

# Melihat isi tanpa membuka (t=list)
tar tzf backup.tar.gz
```

> 💡 Cara hafal: **c**reate / e**x**tract / lis**t**, selalu ditambah `zf`.

### zip / unzip

```bash
zip -r arsip.zip folder/
unzip arsip.zip -d tujuan/
```

### Pola Backup Sederhana

Gabungan `tar` + tanggal + `rsync` = backup otomatis (akan dijadwalkan dengan cron di bagian 7):

```bash
# Buat arsip dengan nama bertanggal
tar czf /backup/web-$(date +%F).tar.gz /var/www/html

# Kirim ke server backup
rsync -avz /backup/ user@server-backup:/backup/web01/

# Hapus backup lokal yang lebih tua dari 7 hari
find /backup -name "*.tar.gz" -mtime +7 -delete
```

---

## 7. Penjadwalan Tugas — Cron

Cron menjalankan perintah secara otomatis pada jadwal tertentu: backup malam hari, pembersihan log mingguan, dan sebagainya.

```bash
# Edit jadwal milik user saat ini
crontab -e

# Lihat jadwal aktif
crontab -l
```

### Format 5 Kolom

```
┌──────── menit        (0-59)
│ ┌────── jam          (0-23)
│ │ ┌──── tanggal      (1-31)
│ │ │ ┌── bulan        (1-12)
│ │ │ │ ┌ hari-minggu  (0-7, 0 & 7 = Minggu)
│ │ │ │ │
* * * * *  perintah
```

Contoh-contoh:

```bash
# Setiap hari jam 02.30 — jalankan script backup
30 2 * * * /home/user/backup.sh

# Setiap 15 menit
*/15 * * * * /home/user/cek-kesehatan.sh

# Setiap Senin jam 08.00
0 8 * * 1 /home/user/laporan-mingguan.sh

# Setiap kali server boot
@reboot /home/user/startup.sh
```

> 💡 **Tips penting:** cron berjalan dengan environment minimal — selalu pakai **path absolut** (`/usr/bin/python3`, bukan `python3`) dan arahkan output ke log: `30 2 * * * /home/user/backup.sh >> /var/log/backup.log 2>&1`. Cek eksekusi cron di `grep CRON /var/log/syslog`.

> 💡 Alternatif modern: **systemd timer** (dibahas di bagian 11) — kelebihannya log otomatis masuk `journalctl` dan jadwal yang terlewat saat server mati bisa dikejar (`Persistent=true`).

---

## 8. Environment Variable & Kustomisasi Shell

Konsep yang sama dengan `-e` di Docker — di shell, environment variable mengatur perilaku program.

```bash
# Melihat semua environment variable
env

# Melihat satu variabel
echo $HOME
echo $PATH

# Menyetel untuk sesi ini saja
export APP_ENV=production

# Variabel hanya untuk satu perintah
APP_ENV=staging node server.js
```

### PATH — Dari Mana Perintah Ditemukan

`PATH` adalah daftar folder yang dicari shell saat mengetik perintah:

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:...

# Menambah folder sendiri ke PATH
export PATH="$HOME/bin:$PATH"
```

### Membuat Permanen — ~/.bashrc

Semua `export` dan `alias` hilang saat terminal ditutup, kecuali disimpan di `~/.bashrc` (atau `~/.zshrc` untuk zsh):

```bash
nano ~/.bashrc
```

```bash
# Tambahkan di akhir file:
export PATH="$HOME/bin:$PATH"
export EDITOR=nano

# Alias — singkatan perintah
alias ll='ls -la'
alias masuk-server='ssh server-belajar'
alias log-nginx='sudo tail -f /var/log/nginx/access.log'
```

```bash
# Muat ulang tanpa buka terminal baru
source ~/.bashrc
```

---

## 9. tmux — Sesi Kerja yang Tidak Putus

Masalah klasik: menjalankan proses panjang lewat SSH, koneksi putus, proses ikut mati. **tmux** memisahkan sesi kerja dari koneksi — sesi tetap hidup di server walau SSH terputus.

```bash
sudo apt install tmux

# Buat sesi baru dengan nama
tmux new -s deploy

# ... kerjakan apa pun di dalamnya ...

# Lepas dari sesi (sesi tetap jalan): tekan  Ctrl+b  lalu  d

# Lihat daftar sesi
tmux ls

# Masuk kembali ke sesi (misal setelah SSH ulang)
tmux attach -t deploy

# Tutup sesi permanen: ketik exit di dalamnya
```

Shortcut dasar (semua diawali `Ctrl+b`):

| Tombol | Fungsi |
|--------|--------|
| `d` | Detach — keluar tanpa mematikan sesi |
| `%` | Belah layar vertikal |
| `"` | Belah layar horizontal |
| `panah` | Pindah antar panel |
| `c` | Window baru |
| `n` / `p` | Window berikutnya / sebelumnya |

> 💡 Kebiasaan baik: begitu SSH ke server untuk pekerjaan panjang (migrasi, update besar), langsung `tmux new -s kerja`. Kalau koneksi putus, tinggal `tmux attach -t kerja` — pekerjaan tidak hilang.

---

# LEVEL MAHIR

## 10. Bash Scripting

Otomasi adalah inti DevOps, dan bash script adalah alat otomasi paling dasar. Semua yang bisa diketik di terminal bisa dijadikan script.

### Script Pertama

```bash
#!/bin/bash
# baris pertama (shebang) menentukan interpreter

echo "Halo, $USER! Hari ini $(date +%A)."
```

```bash
chmod +x halo.sh    # beri izin eksekusi
./halo.sh           # jalankan
```

### Variabel & Argumen

```bash
#!/bin/bash

# Variabel — TANPA spasi di sekitar =
NAMA="server01"
echo "Memproses $NAMA"

# Menangkap output perintah
TANGGAL=$(date +%F)
JUMLAH_FILE=$(ls | wc -l)

# Argumen dari command line: ./script.sh arg1 arg2
echo "Argumen pertama : $1"
echo "Semua argumen   : $@"
echo "Jumlah argumen  : $#"
```

### Kondisi — if

```bash
#!/bin/bash

if [ -z "$1" ]; then
    echo "Cara pakai: $0 <nama-file>"
    exit 1          # exit code selain 0 = gagal
fi

if [ -f "$1" ]; then
    echo "File $1 ada."
elif [ -d "$1" ]; then
    echo "$1 adalah direktori."
else
    echo "$1 tidak ditemukan."
fi

# Membandingkan angka: -eq -ne -gt -lt -ge -le
PAKAI_DISK=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
if [ "$PAKAI_DISK" -gt 80 ]; then
    echo "PERINGATAN: disk terpakai ${PAKAI_DISK}%"
fi
```

Operator tes yang sering dipakai:

| Operator | Arti |
|----------|------|
| `-f file` | File ada |
| `-d dir` | Direktori ada |
| `-z str` | String kosong |
| `-n str` | String tidak kosong |
| `str1 = str2` | String sama |
| `-eq -gt -lt` | Angka: sama / lebih besar / lebih kecil |

### Perulangan — for & while

```bash
# Loop daftar
for SERVER in web01 web02 web03; do
    echo "Deploy ke $SERVER..."
    rsync -az app/ user@$SERVER:/var/www/app/
done

# Loop file
for F in /var/log/*.log; do
    echo "$F: $(wc -l < "$F") baris"
done

# while — baca file baris per baris
while read -r BARIS; do
    echo "-> $BARIS"
done < daftar-server.txt
```

### Function & Exit Code

```bash
#!/bin/bash

cek_service() {
    if systemctl is-active --quiet "$1"; then
        echo "✔ $1 berjalan"
        return 0
    else
        echo "✘ $1 MATI"
        return 1
    fi
}

cek_service nginx
cek_service postgresql

# $? = exit code perintah terakhir (0 = sukses)
ping -c1 -W2 8.8.8.8 > /dev/null 2>&1
if [ $? -eq 0 ]; then echo "Internet OK"; fi
```

### Script Aman — set -euo pipefail

```bash
#!/bin/bash
set -euo pipefail
# -e : berhenti begitu ada perintah gagal
# -u : error jika memakai variabel yang belum didefinisikan
# -o pipefail : pipe dianggap gagal jika salah satu bagiannya gagal
```

> 💡 Biasakan tiga baris ini di awal semua script produksi — mencegah script "jalan terus dalam keadaan rusak".

### Contoh Utuh: Script Backup

Menggabungkan hampir semua materi sebelumnya (tar, rsync, find, cron):

```bash
#!/bin/bash
set -euo pipefail

SUMBER="/var/www/html"
TUJUAN="/backup"
SERVER_BACKUP="user@203.0.113.20:/backup/web01/"
TANGGAL=$(date +%F)
ARSIP="$TUJUAN/web-$TANGGAL.tar.gz"

mkdir -p "$TUJUAN"

echo "[1/3] Membuat arsip $ARSIP"
tar czf "$ARSIP" "$SUMBER"

echo "[2/3] Sinkron ke server backup"
rsync -az "$TUJUAN/" "$SERVER_BACKUP"

echo "[3/3] Hapus arsip lokal > 7 hari"
find "$TUJUAN" -name "*.tar.gz" -mtime +7 -delete

echo "Backup selesai: $(du -sh "$ARSIP" | cut -f1)"
```

Jadwalkan tiap malam via cron: `30 2 * * * /home/user/backup.sh >> /var/log/backup.log 2>&1`

---

## 11. Systemd Lanjutan — Membuat Service Sendiri

Materi dasar mengajarkan *mengelola* service bawaan (nginx). Sekarang: menjadikan **aplikasi sendiri** sebagai service — auto-start saat boot, auto-restart saat crash, log terpusat. Inilah cara "benar" menjalankan aplikasi di server (pengganti `nohup`).

### Menulis Unit File

```bash
sudo nano /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=Aplikasi Node.js MyApp
# Jalankan setelah jaringan siap
After=network.target

[Service]
# User non-root untuk keamanan
User=www-data
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/node server.js
# Restart otomatis jika crash
Restart=on-failure
RestartSec=5
# Environment variable untuk aplikasi
Environment=NODE_ENV=production
Environment=PORT=3000
# Atau dari file: EnvironmentFile=/etc/myapp/env

[Install]
# Aktif pada boot normal (multi-user)
WantedBy=multi-user.target
```

### Mengaktifkan

```bash
# Muat ulang konfigurasi systemd (wajib setiap ubah unit file)
sudo systemctl daemon-reload

# Aktifkan saat boot + jalankan sekarang, sekaligus
sudo systemctl enable --now myapp

# Cek
systemctl status myapp
```

### journalctl — Membaca Log Service

```bash
# Log service tertentu
journalctl -u myapp

# Follow real-time (seperti tail -f)
journalctl -u myapp -f

# Sejak waktu tertentu
journalctl -u myapp --since "1 hour ago"
journalctl -u myapp --since today

# Hanya level error ke atas
journalctl -u myapp -p err

# Log boot terakhir & pemakaian disk journal
journalctl -b
journalctl --disk-usage
```

### systemd Timer — Alternatif Cron

```bash
sudo nano /etc/systemd/system/backup.timer
```

```ini
[Unit]
Description=Jadwal backup harian

[Timer]
OnCalendar=*-*-* 02:30:00
# Kejar jadwal yang terlewat saat server mati
Persistent=true

[Install]
WantedBy=timers.target
```

Timer memicu unit `.service` dengan nama sama (`backup.service`). Aktifkan dengan `sudo systemctl enable --now backup.timer`, lihat semua jadwal dengan `systemctl list-timers`.

---

## 12. Keamanan Server

Melanjutkan SSH hardening di materi dasar — tiga lapis tambahan.

### Firewall — ufw

Prinsip: **tutup semua, buka seperlunya.**

```bash
sudo apt install ufw

# Kebijakan default: tolak masuk, izinkan keluar
sudo ufw default deny incoming
sudo ufw default allow outgoing

# PENTING: izinkan SSH DULU sebelum enable — kalau tidak, terkunci!
sudo ufw allow OpenSSH        # atau: sudo ufw allow 22/tcp

# Buka port layanan yang memang publik
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Nyalakan
sudo ufw enable

# Cek aturan aktif
sudo ufw status verbose

# Hapus aturan
sudo ufw delete allow 80/tcp
```

### fail2ban — Blokir Brute Force Otomatis

fail2ban membaca log login gagal dan mem-blokir IP penyerang otomatis:

```bash
sudo apt install fail2ban

# Konfigurasi: JANGAN edit jail.conf, buat jail.local
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
maxretry = 5
findtime = 10m
bantime = 1h
```

```bash
sudo systemctl enable --now fail2ban

# Lihat status & IP yang diblokir
sudo fail2ban-client status sshd
```

### sudoers — Mengatur Hak sudo

```bash
# SELALU edit lewat visudo (memvalidasi sintaks sebelum simpan —
# file sudoers yang rusak bisa mengunci semua akses sudo)
sudo visudo
```

```
# Grup sudo boleh semua (bawaan Ubuntu)
%sudo   ALL=(ALL:ALL) ALL

# User deploy hanya boleh restart nginx, tanpa password
deploy  ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```

### Audit Login

```bash
last                      # riwayat login berhasil
lastb                     # percobaan login GAGAL (butuh sudo)
who                       # siapa yang sedang login sekarang
sudo tail -f /var/log/auth.log   # pantau autentikasi real-time
```

---

## 13. Storage & Disk

Bobot 20% di LFCS. Kebutuhan nyata: menambah disk baru ke server dan memperbesar partisi yang penuh.

### Melihat Disk & Partisi

```bash
lsblk                 # pohon disk, partisi, dan mount point
sudo fdisk -l         # detail tabel partisi
blkid                 # UUID tiap partisi (dipakai di fstab)
df -h                 # sisa ruang per filesystem
du -sh /var/*         # siapa pemakan ruang terbesar
```

### Menambahkan Disk Baru (Alur Lengkap)

```bash
# 1. Buat partisi di disk baru (misal /dev/sdb)
sudo fdisk /dev/sdb        # n (new) -> w (write)

# 2. Format dengan filesystem ext4
sudo mkfs.ext4 /dev/sdb1

# 3. Mount manual
sudo mkdir /data
sudo mount /dev/sdb1 /data

# 4. Permanen — tambahkan ke /etc/fstab (pakai UUID agar tahan
#    perubahan urutan disk)
blkid /dev/sdb1
sudo nano /etc/fstab
#   UUID=xxxx-xxxx  /data  ext4  defaults  0 2

# 5. Uji fstab TANPA reboot (fstab salah = gagal boot!)
sudo mount -a
```

### Swap

Cadangan RAM di disk — menyelamatkan server kecil dari kehabisan memori:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

free -h        # verifikasi
```

### LVM — Logical Volume Manager

LVM menyisipkan lapisan fleksibel antara disk fisik dan filesystem, sehingga partisi bisa **diperbesar tanpa unmount/reboot**. Tiga lapisan:

```
Disk fisik -> PV (Physical Volume) -> VG (Volume Group) -> LV (Logical Volume) -> filesystem
```

```bash
# 1. Jadikan disk sebagai Physical Volume
sudo pvcreate /dev/sdb /dev/sdc

# 2. Gabungkan ke Volume Group bernama "data"
sudo vgcreate data /dev/sdb /dev/sdc

# 3. Buat Logical Volume 100 GB bernama "web"
sudo lvcreate -L 100G -n web data

# 4. Format & mount seperti partisi biasa
sudo mkfs.ext4 /dev/data/web
sudo mount /dev/data/web /var/www

# Melihat kondisi
sudo pvs && sudo vgs && sudo lvs
```

**Keunggulan utama — memperbesar saat penuh, tanpa downtime:**

```bash
# Tambah 50 GB ke LV, lalu besarkan filesystem-nya sekaligus (-r)
sudo lvextend -L +50G -r /dev/data/web
```

Kalau VG kehabisan ruang: pasang disk baru → `pvcreate /dev/sdd` → `vgextend data /dev/sdd` → `lvextend` lagi.

---

## 14. Manajemen Log

### journalctl Persisten

Secara default di beberapa distro, journal hilang saat reboot. Aktifkan penyimpanan permanen:

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald

# Batasi ukurannya di /etc/systemd/journald.conf:
#   SystemMaxUse=500M
```

### logrotate — Log Tidak Memakan Disk

Log yang tidak dirotasi adalah penyebab klasik disk penuh. logrotate memotong, mengompres, dan menghapus log lama otomatis:

```bash
sudo nano /etc/logrotate.d/myapp
```

```
/var/www/myapp/logs/*.log {
    daily              # rotasi harian
    rotate 14          # simpan 14 file terakhir
    compress           # kompres yang lama (gzip)
    delaycompress      # kecuali yang paling baru
    missingok          # jangan error jika file belum ada
    notifempty         # lewati jika kosong
    copytruncate       # potong file tanpa restart aplikasi
}
```

```bash
# Uji konfigurasi tanpa benar-benar merotasi
sudo logrotate -d /etc/logrotate.d/myapp
```

---

## 15. Performa & Troubleshooting

### Membaca Load Average

```bash
uptime
# 10:00:00 up 30 days, load average: 0.52, 1.10, 2.30
#                                    1mnt  5mnt  15mnt
```

Angka load = rata-rata proses yang antre CPU. **Patokan: bandingkan dengan jumlah core** (`nproc`). Server 4 core dengan load 4.0 = terpakai penuh; load 8.0 = antrean 2x kapasitas. Load 15-menit tinggi tapi 1-menit rendah = badai sudah lewat.

### Alat Pantau

```bash
htop                # top versi nyaman (install: apt install htop)
free -h             # RAM — lihat kolom "available", bukan "free"
vmstat 2            # ringkasan CPU/RAM/IO tiap 2 detik
iostat -x 2         # beban per disk (install: apt install sysstat)
sudo iotop          # proses yang paling banyak menulis disk
```

> 💡 `free` sering disalahpahami: "free" kecil itu normal — Linux memakai RAM nganggur untuk cache. Yang penting kolom **available**.

### OOM Killer

Saat RAM benar-benar habis, kernel *membunuh* proses paling rakus (Out-Of-Memory Killer). Kalau aplikasi mati misterius tanpa log error:

```bash
sudo dmesg | grep -i "out of memory"
journalctl -k --since yesterday | grep -i oom
```

### Checklist Troubleshooting "Service Down"

Urutan baku yang menyelesaikan sebagian besar insiden:

```bash
# 1. Statusnya apa?
systemctl status myapp

# 2. Log-nya bilang apa?
journalctl -u myapp -n 50 --no-pager

# 3. Port-nya listening?
sudo ss -tulpn | grep 3000

# 4. Disk penuh?
df -h

# 5. Memori habis? (cek juga OOM)
free -h && sudo dmesg | tail -20

# 6. Bisa dijangkau dari dalam?
curl -I http://localhost:3000
```

---

## 16. Di Balik Container — Namespaces & cgroups

Jembatan ke [materi Docker](docker.md): container **bukan sihir** — ia dibangun dari dua fitur kernel Linux yang bisa dipakai langsung.

### Namespaces — Isolasi Pandangan

Namespace membuat proses "melihat dunia sendiri". Jenis-jenis utamanya:

| Namespace | Yang diisolasi |
|-----------|----------------|
| `pid` | Daftar proses — PID 1 di container ≠ PID 1 host |
| `net` | Interface jaringan, IP, port sendiri |
| `mnt` | Daftar mount / filesystem |
| `uts` | Hostname sendiri |
| `user` | Mapping user — root di container ≠ root di host |

Bisa dicoba tanpa Docker:

```bash
# Proses dengan PID namespace sendiri — di dalamnya, bash jadi PID 1
sudo unshare --pid --fork --mount-proc bash
ps aux    # hanya terlihat proses milik "dunia" ini
exit

# Lihat namespace milik sebuah proses
ls -la /proc/$$/ns/
```

### cgroups — Pembatasan Sumber Daya

Control groups membatasi CPU, memori, dan IO per kelompok proses. Inilah mesin di balik flag `docker run --memory 512m --cpus 1`:

```bash
# systemd memakai cgroups untuk semua service — lihat pohonnya
systemd-cgls

# Pemakaian resource per service, real-time
systemd-cgtop

# Batasi resource sebuah service systemd (bagian 11)
# tambahkan di [Service]:
#   MemoryMax=512M
#   CPUQuota=50%
```

> 💡 Dengan pemahaman ini, Docker jadi masuk akal: **image** = filesystem yang dibekukan, **container** = proses biasa di host yang dibungkus namespaces (isolasi) + cgroups (pembatasan). Karena hanya proses, itulah alasan container start dalam hitungan detik — dibanding VM yang harus boot OS lengkap.

---

## Jalur Belajar & Sertifikasi

Materi ini memetakan ke lima domain ujian **LFCS**:

| Domain LFCS | Bobot | Dibahas di |
|-------------|-------|-----------|
| Operations Deployment | 25% | Bagian 3, 7, 11, 16 + [docker.md](docker.md) |
| Networking | 25% | Bagian 4, 12 + [linux.md](linux.md) SSH |
| Storage | 20% | Bagian 13 |
| Essential Commands | 20% | Bagian 2, 5, 6 + [linux.md](linux.md) |
| Users and Groups | 10% | Bagian 8, 12 + [linux.md](linux.md) |

Urutan belajar yang disarankan: [linux.md](linux.md) → materi ini level menengah → praktik membuat script backup + service systemd sendiri → level mahir → lanjut ke [nginx.md](nginx.md) dan [docker.md](docker.md).

## Referensi

- Silabus LFCS resmi: https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/
- Roadmap DevOps: https://roadmap.sh/devops
- Dokumentasi man pages online: https://man7.org
- rsync manual: https://linux.die.net/man/1/rsync
- Panduan systemd: https://www.freedesktop.org/software/systemd/man/
- Repository pembelajaran: https://github.com/DistritekDevOps/learning-devops
