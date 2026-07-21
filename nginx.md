# Materi Nginx: Dasar sampai Mahir

Nginx adalah web server sekaligus reverse proxy yang paling banyak dipakai untuk melayani website dan mengarahkan traffic ke aplikasi backend. Materi ini membahas dari instalasi, konfigurasi dasar, sampai reverse proxy, SSL, dan optimasi untuk kebutuhan DevOps.

---

## 1. Pengenalan Nginx

Nginx bisa berperan sebagai:

- **Web server** — menyajikan file statis (HTML, CSS, JS, gambar) langsung dari disk.
- **Reverse proxy** — meneruskan request ke aplikasi backend (Node.js, PHP, Python, dll), sekaligus menyembunyikan detail backend dari luar.
- **Load balancer** — membagi traffic ke beberapa server backend.

### Lokasi Penting

| Lokasi | Fungsi |
|--------|--------|
| `/etc/nginx/` | Direktori konfigurasi utama |
| `/etc/nginx/nginx.conf` | File konfigurasi global |
| `/etc/nginx/sites-available/` | Kumpulan konfigurasi site (belum tentu aktif) |
| `/etc/nginx/sites-enabled/` | Site yang aktif (biasanya symlink dari `sites-available`) |
| `/var/www/html/` | Direktori root web (tempat menyimpan file website) |
| `/var/log/nginx/access.log` | Catatan setiap pengunjung yang masuk |
| `/var/log/nginx/error.log` | Catatan jika terjadi error pada sistem |

---

## 2. Instalasi dan Manajemen Service

```bash
# Instalasi di Ubuntu/Debian
sudo apt update
sudo apt install nginx

# Mengelola service (lihat juga materi Linux bagian systemd)
sudo systemctl status nginx
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl enable nginx    # otomatis jalan saat boot

# Reload konfigurasi tanpa memutus koneksi yang sedang berjalan
sudo systemctl reload nginx
```

> 💡 Gunakan `reload`, bukan `restart`, setelah mengubah konfigurasi di server produksi — `reload` menerapkan config baru tanpa menjatuhkan koneksi yang sedang aktif.

---

## 3. Struktur Konfigurasi Dasar

Contoh konfigurasi minimal untuk menyajikan website statis:

```nginx
server {
    listen 80;
    server_name websitekamu.com;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Penjelasan tiap baris:

- **`server { }`** — satu blok mewakili satu website/domain (disebut *server block*).
- **`listen 80`** — port yang didengarkan (80 = HTTP biasa, 443 = HTTPS).
- **`server_name`** — domain yang dilayani blok ini.
- **`root`** — folder tempat file website disimpan.
- **`index`** — file default yang dicari saat mengakses folder.
- **`location / { }`** — aturan untuk path tertentu (`/` = semua path).
- **`try_files`** — urutan pencarian file; `=404` berarti tampilkan 404 jika semua gagal ditemukan.

### Mengaktifkan Konfigurasi Site

```bash
# Buat file konfigurasi baru
sudo nano /etc/nginx/sites-available/websitekamu.com

# Aktifkan dengan symlink ke sites-enabled
sudo ln -s /etc/nginx/sites-available/websitekamu.com /etc/nginx/sites-enabled/

# Nonaktifkan site (hapus symlink, bukan file aslinya)
sudo rm /etc/nginx/sites-enabled/websitekamu.com

# Cek sintaks konfigurasi sebelum reload — WAJIB dilakukan
sudo nginx -t

# Terapkan perubahan
sudo systemctl reload nginx
```

> ⚠️ **Selalu jalankan `nginx -t` sebelum `reload`.** Konfigurasi yang salah sintaks akan membuat `reload`/`restart` gagal dan bisa membuat website down.

---

## 4. Location Block — Mengatur Routing

`location` menentukan bagaimana Nginx merespons request berdasarkan path URL.

```nginx
server {
    listen 80;
    server_name websitekamu.com;
    root /var/www/html;

    # Cocok persis dengan path "/"
    location = / {
        try_files $uri /index.html;
    }

    # Prefix match — semua path yang diawali /api/
    location /api/ {
        proxy_pass http://localhost:3000/;
    }

    # Regex match — file dengan ekstensi tertentu
    location ~* \.(jpg|jpeg|png|gif|css|js)$ {
        expires 30d;
        add_header Cache-Control "public";
    }

    # Blokir akses ke file tersembunyi (contoh: .env, .git)
    location ~ /\. {
        deny all;
    }
}
```

Urutan prioritas matching: `=` (exact) → prefix terpanjang → regex (`~`/`~*`) → prefix biasa.

---

## 5. Reverse Proxy — Mengarahkan ke Aplikasi Backend

Ini fungsi paling umum Nginx di lingkungan DevOps: menerima request di port 80/443, lalu meneruskannya ke aplikasi yang berjalan di port lain (misalnya Node.js di port 3000).

```nginx
server {
    listen 80;
    server_name app.websitekamu.com;

    location / {
        proxy_pass http://localhost:3000;

        # Header standar agar backend tahu identitas request asli
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Wajib untuk WebSocket (contoh: aplikasi realtime, Socket.IO)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

> 💡 Tanpa header `X-Real-IP`/`X-Forwarded-For`, aplikasi backend akan mengira semua request datang dari `127.0.0.1` (IP Nginx), bukan IP pengunjung asli.

### Load Balancing — Beberapa Server Backend

```nginx
upstream backend_app {
    least_conn;                      # strategi: kirim ke server dengan koneksi paling sedikit
    server 10.0.0.11:3000;
    server 10.0.0.12:3000;
    server 10.0.0.13:3000 backup;    # hanya dipakai jika semua server lain down
}

server {
    listen 80;
    server_name app.websitekamu.com;

    location / {
        proxy_pass http://backend_app;
    }
}
```

Strategi load balancing yang umum: `round-robin` (default, bergiliran), `least_conn` (paling sedikit koneksi aktif), `ip_hash` (satu IP selalu ke server yang sama — berguna untuk session).

---

## 6. SSL/TLS dengan Let's Encrypt (Certbot)

HTTPS wajib untuk website produksi. Cara termudah dan gratis: **Certbot**.

```bash
# Instalasi Certbot
sudo apt install certbot python3-certbot-nginx

# Minta sertifikat sekaligus otomatis konfigurasi Nginx
sudo certbot --nginx -d websitekamu.com -d www.websitekamu.com

# Uji perpanjangan otomatis (sertifikat berlaku 90 hari)
sudo certbot renew --dry-run
```

Certbot otomatis menambahkan blok berikut ke konfigurasi:

```nginx
server {
    listen 443 ssl;
    server_name websitekamu.com;

    ssl_certificate /etc/letsencrypt/live/websitekamu.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/websitekamu.com/privkey.pem;

    root /var/www/html;
    index index.html;
}

# Redirect semua traffic HTTP ke HTTPS
server {
    listen 80;
    server_name websitekamu.com;
    return 301 https://$host$request_uri;
}
```

> 💡 Certbot otomatis membuat cron job/systemd timer untuk perpanjangan sertifikat — biasanya tidak perlu campur tangan manual lagi setelah setup awal.

---

## 7. Optimasi dan Praktik Terbaik

### Kompresi Gzip

```nginx
http {
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_min_length 1024;
}
```

### Caching File Statis

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

### Membatasi Ukuran Upload

```nginx
http {
    client_max_body_size 20M;
}
```

### Rate Limiting — Mencegah Abuse/Brute Force

```nginx
http {
    # Batasi 10 request per detik per IP
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

    server {
        location /login {
            limit_req zone=mylimit burst=20 nodelay;
            proxy_pass http://localhost:3000;
        }
    }
}
```

### Custom Error Page

```nginx
server {
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location = /404.html {
        root /var/www/html;
    }
}
```

---

## 8. Monitoring dan Troubleshooting

```bash
# Cek sintaks konfigurasi
sudo nginx -t

# Melihat log akses real-time
sudo tail -f /var/log/nginx/access.log

# Melihat log error real-time
sudo tail -f /var/log/nginx/error.log

# 10 IP yang paling sering mengakses (lihat juga linux-lengkap.md bagian text processing)
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Cek apakah Nginx sedang listen di port 80/443
sudo ss -tulpn | grep nginx
```

### Error Umum

| Error | Penyebab Umum | Solusi |
|-------|---------------|--------|
| `403 Forbidden` | Permission file/folder salah | `sudo chown -R www-data:www-data /var/www/html` (lihat [linux.md](linux.md) bagian permission) |
| `502 Bad Gateway` | Backend (`proxy_pass`) mati atau salah port | Cek `systemctl status` aplikasi backend, cek port dengan `ss -tulpn` |
| `504 Gateway Timeout` | Backend terlalu lambat merespons | Naikkan `proxy_read_timeout`, cek performa aplikasi backend |
| Konfigurasi tidak berubah setelah edit | Lupa `nginx -t` + `reload`, atau site belum di-symlink ke `sites-enabled` | Cek `sudo nginx -t`, lalu `sudo systemctl reload nginx` |

---

## 9. Cheat Sheet Ringkas

```bash
sudo nginx -t                         # Cek sintaks konfigurasi
sudo systemctl reload nginx           # Terapkan config baru tanpa downtime
sudo systemctl restart nginx          # Restart penuh
sudo tail -f /var/log/nginx/error.log # Pantau error real-time
sudo certbot --nginx -d domain.com    # Pasang HTTPS gratis
```

| Direktif | Fungsi |
|----------|--------|
| `listen` | Port yang didengarkan |
| `server_name` | Domain yang dilayani |
| `root` | Folder file statis |
| `location` | Aturan routing per path |
| `proxy_pass` | Teruskan request ke backend |
| `try_files` | Urutan pencarian file |
| `return` | Redirect/response langsung |
| `upstream` | Grup server untuk load balancing |

---

## Referensi

- Dokumentasi resmi: https://nginx.org/en/docs/
- Certbot: https://certbot.eff.org
- Materi terkait: [linux.md](linux.md) (permission, systemd), [docker.md](docker.md) (menjalankan Nginx sebagai container)
- Repository pembelajaran: https://github.com/DistritekDevOps/learning-devops
