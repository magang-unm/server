# Panduan: Analisis Log, Prioritisasi Vulnerability & Troubleshooting dengan AI Agent

> Panduan praktis (Juli 2026) untuk tiga tugas sysadmin yang paling siap diotomasi dengan AI agent:
> analisis log (80%), prioritisasi vulnerability (67%), troubleshooting (55%).

Semua contoh di bawah memakai **Claude Code** via SSH ke server Linux — pendekatan yang paling sederhana karena tidak butuh platform tambahan.

---

## 0. Persiapan: Akses Agent ke Server

Sebelum ketiga tugas bisa jalan, siapkan akses SSH yang aman untuk agent:

```bash
# 1. Buat SSH key khusus agent (ed25519 — lebih kecil, cepat, dan aman dari RSA)
ssh-keygen -t ed25519 -f ~/.ssh/id_agent -C "claude-agent"
chmod 600 ~/.ssh/id_agent

# 2. Buat user khusus di server dengan akses terbatas (bukan root!)
sudo adduser aiagent
sudo usermod -aG adm aiagent        # grup adm = bisa baca /var/log tanpa sudo

# 3. Daftarkan public key ke server
ssh-copy-id -i ~/.ssh/id_agent.pub aiagent@server-kamu

# 4. Tambahkan ke ~/.ssh/config supaya mudah dipanggil
cat >> ~/.ssh/config <<'EOF'
Host prod-server
    HostName <IP-SERVER>
    User aiagent
    IdentityFile ~/.ssh/id_agent
EOF

# 5. Tes koneksi dari terminal dulu sebelum dipakai agent
ssh prod-server "hostname && uptime"
```

**Prinsip:** untuk analisis log dan troubleshooting, akses *read-only* sudah cukup. Jangan beri sudo penuh; kalau perlu, batasi lewat `/etc/sudoers.d/` hanya untuk perintah tertentu (misalnya `systemctl status`).

Referensi setup: [Manage Servers with Claude Code via SSH](https://computingforgeeks.com/claude-code-ssh-server-management/), [SSH Key Setup on Claude Code](https://usagebar.com/blog/how-to-do-ssh-key-setup-on-claude-code), [Claude Code SSH Like a Pro](https://medium.com/@joe.njenga/how-i-use-claude-code-ssh-to-connect-to-any-remote-server-like-a-pro-f08ed35e5569).

---

## 1. Analisis Log (80%)

Keunggulan LLM di sini: bisa langsung memahami log tak terstruktur tanpa aturan parsing manual, dan menemukan pola lintas jutaan baris yang dilewatkan alert berbasis rule.

### Cara pakai (on-demand)

Jalankan Claude Code dari laptop, minta dia menarik log lewat SSH:

```
claude "SSH ke prod-server, ambil /var/log/nginx/error.log dan access.log
24 jam terakhir. Rangkum: (1) error paling sering + akar masalahnya,
(2) lonjakan traffic tidak wajar, (3) IP mencurigakan (brute force / scanning),
(4) rekomendasi perbaikan."
```

Contoh tugas lain yang efektif:

```
# Deteksi anomali dari journal systemd
claude "SSH ke prod-server, jalankan 'journalctl --since \"1 hour ago\" -p warning'
lalu jelaskan apakah ada yang perlu ditindaklanjuti"

# Korelasi antar log — kekuatan utama LLM
claude "Bandingkan error nginx dengan log aplikasi di waktu yang sama,
cari korelasinya"
```

### Cara pakai (rutin/terjadwal)

Buat laporan harian otomatis via cron di laptop/VM operasional:

```bash
# /etc/cron.d/ atau crontab -e — setiap pagi jam 7
0 7 * * * claude -p "SSH ke prod-server, analisa log nginx & syslog 24 jam terakhir, tulis ringkasan ke ~/reports/log-$(date +\%F).md" >> ~/reports/cron.log 2>&1
```

Flag `-p` (print mode) menjalankan Claude Code non-interaktif — cocok untuk otomasi.

### Skala lebih besar

Kalau server banyak atau volume log besar, gabungkan dengan platform observability yang sudah punya AI bawaan seperti [OpenObserve](https://openobserve.ai/blog/best-log-analysis-tools/), atau pakai [mcp-ssh-manager](https://github.com/bvisible/mcp-ssh-manager) supaya satu sesi agent bisa memantau banyak server sekaligus.

---

## 2. Prioritisasi Vulnerability (67%)

Fakta kunci: **96% CVE dengan CVSS ≥ 9.0 tidak pernah dieksploitasi di dunia nyata** — memprioritaskan hanya berdasar CVSS berarti membuang sebagian besar effort ke lubang yang salah. Pendekatan yang benar menggabungkan tiga sinyal:

| Sinyal | Menjawab | Sumber |
| ------ | -------- | ------ |
| CVSS   | Seberapa parah *secara teori* | Hasil scan (Trivy dll.) |
| EPSS   | Seberapa besar *kemungkinan dieksploitasi* 30 hari ke depan | [API FIRST.org](https://api.first.org/data/v1/epss) (gratis) |
| Konteks kode/server | Apakah kita *benar-benar terekspos* | LLM menganalisis codebase/config |

### Workflow praktis

```bash
# 1. Install Trivy (scanner open-source)
brew install trivy          # macOS
# atau: apt install trivy   # Debian/Ubuntu

# 2. Scan server / image / project
trivy image nginx:latest --format json -o scan.json
trivy fs /path/ke/project --scanners vuln --format json -o scan.json
# di server langsung: trivy rootfs / --format json -o scan.json
```

```
# 3. Serahkan prioritisasi ke agent
claude "Baca scan.json. Untuk setiap CVE:
1. Ambil skor EPSS dari https://api.first.org/data/v1/epss?cve=<CVE-ID>
2. Cek konteks: apakah paket/fungsi rentan itu benar-benar dipakai
   dan terjangkau dari jalur yang menghadap internet (lihat nginx.conf
   dan reverse_proxy.conf di repo ini)
3. Buat tabel prioritas: [CVE, CVSS, EPSS, terekspos?, urgensi, aksi]
Urutkan berdasar risiko NYATA, bukan CVSS semata."
```

Agent akan menghasilkan antrian remediasi yang sesuai risiko dunia nyata: CVE dengan CVSS 7.2 di jalur yang memproses input internet bisa lebih urgent daripada CVSS 9.8 di library yang tidak pernah dipanggil.

Referensi: [AI Vulnerability Prioritisation: EPSS + LLM Context](https://aquilax.ai/blog/ai-vulnerability-prioritization-remediation), [VulnWatch — Databricks](https://www.databricks.com/blog/vulnwatch-ai-enhanced-prioritization-vulnerabilities), [AI Vulnerability Triage Guide](https://winfunc.com/blog/ai-vulnerability-triage).

---

## 3. Troubleshooting (55%)

Pola kerjanya: agent mengamati → mendiagnosis → **mengusulkan** perbaikan → manusia menyetujui → agent mengeksekusi & memverifikasi. Diagnosis boleh otonom; eksekusi perbaikan tetap lewat approval.

### Contoh sesi nyata

```
# Service down
claude "SSH ke prod-server. Nginx mengembalikan 502. Diagnosa:
cek status nginx & upstream service, log error terkait, port yang listen,
disk & memory. Jelaskan akar masalahnya dan USULKAN perbaikan —
jangan eksekusi apa pun sebelum saya setujui."

# Server lambat
claude "SSH ke prod-server, server terasa lambat. Cek load average, proses
paling rakus (top/ps), iowait, koneksi network, dan disk. Rangkum temuanmu."

# Audit konfigurasi (cocok dengan repo ini)
claude "Bandingkan nginx.conf di repo ini dengan yang aktif di prod-server
(/etc/nginx/nginx.conf). Ada drift? Ada misconfiguration?"
```

Kalimat *"jangan eksekusi sebelum saya setujui"* penting — itu yang membuat pola ini aman. Claude Code juga secara default meminta izin sebelum menjalankan perintah yang mengubah state.

### Tips efektivitas

- **Buat runbook** (file md berisi prosedur standar: cara restart service, lokasi log, arsitektur) dan minta agent membacanya dulu — diagnosis jadi jauh lebih akurat.
- **Minta agent mendokumentasikan** setiap sesi troubleshooting ke file, jadi insiden berikutnya punya riwayat.
- Pengalaman praktisi memakai pola ini: [Claude Code as Sysadmin](https://medium.com/@marc.bara.iniesta/claude-code-as-sysadmin-what-it-actually-looks-like-969179b995f3), [How Claude via SSH Simplified Deployment](https://dev.to/quolu/how-allowing-claude-to-access-my-server-via-ssh-simplified-deployment-dramatically-12j8).

---

## Aturan Main (Guardrails)

1. **User khusus, non-root, least privilege** — agent hanya bisa apa yang diizinkan.
2. **Read-only dulu** — analisis log & diagnosis otonom; perubahan (restart, edit config, delete) wajib approval manusia.
3. **Log semua aktivitas agent** untuk audit.
4. **Uji polanya di staging** sebelum diarahkan ke produksi.

---

## Sumber

- [Manage Servers with Claude Code via SSH — ComputingForGeeks](https://computingforgeeks.com/claude-code-ssh-server-management/)
- [How I Use Claude Code SSH — Joe Njenga](https://medium.com/@joe.njenga/how-i-use-claude-code-ssh-to-connect-to-any-remote-server-like-a-pro-f08ed35e5569)
- [SSH Key Setup on Claude Code — Usagebar](https://usagebar.com/blog/how-to-do-ssh-key-setup-on-claude-code)
- [Best Log Analysis Tools 2026 — OpenObserve](https://openobserve.ai/blog/best-log-analysis-tools/)
- [Top Platforms for Log Analysis With AI — Energent](https://www.energent.ai/energent/compare/en/log-analysis-with-ai)
- [mcp-ssh-manager — GitHub](https://github.com/bvisible/mcp-ssh-manager)
- [AI Vulnerability Prioritisation: EPSS + LLM Context — Aquilax](https://aquilax.ai/blog/ai-vulnerability-prioritization-remediation)
- [VulnWatch: AI-Enhanced Prioritization — Databricks](https://www.databricks.com/blog/vulnwatch-ai-enhanced-prioritization-vulnerabilities)
- [AI Vulnerability Triage: A Practical Guide — winfunc](https://winfunc.com/blog/ai-vulnerability-triage)
- [Claude Code as Sysadmin — Marc Bara](https://medium.com/@marc.bara.iniesta/claude-code-as-sysadmin-what-it-actually-looks-like-969179b995f3)
- [How Claude via SSH Simplified Deployment — DEV](https://dev.to/quolu/how-allowing-claude-to-access-my-server-via-ssh-simplified-deployment-dramatically-12j8)
