# Panduan Instalasi n8n + WAHA di VPS dengan Docker

Panduan lengkap dari VPS kosong sampai stack berjalan dengan HTTPS. Untuk gambaran umum project, lihat [Readme.md](Readme.md).

## Daftar Isi

1. [Spesifikasi Minimal VPS](#spesifikasi-minimal-vps)
2. [Setup Awal VPS](#setup-awal-vps)
3. [Konfigurasi Keamanan](#konfigurasi-keamanan)
4. [Instalasi Docker](#instalasi-docker)
5. [Deploy n8n + WAHA](#deploy-n8n--waha)
6. [Reverse Proxy & SSL](#reverse-proxy--ssl)
7. [Backup Otomatis](#backup-otomatis)
8. [Update & Maintenance](#update--maintenance)
9. [Troubleshooting](#troubleshooting)

---

## Spesifikasi Minimal VPS

- **OS**: Ubuntu 22.04 / 24.04 LTS
- **CPU**: 2 vCPU
- **RAM**: 4 GB
- **Storage**: 25 GB SSD
- **Port terbuka**: 22 (SSH), 80 (HTTP), 443 (HTTPS)

---

## Setup Awal VPS

```bash
ssh username@ip-vps

sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git nano ufw
```

---

## Konfigurasi Keamanan

### 1. Firewall (UFW)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

> Port 5678 (n8n) dan 3000 (WAHA) **tidak perlu dibuka** — container hanya bind ke `127.0.0.1` dan diakses lewat Nginx.

### 2. Hardening SSH

```bash
sudo nano /etc/ssh/sshd_config
```

Ubah parameter berikut (pastikan SSH key Anda sudah terpasang sebelum mematikan password auth):

```ini
PermitRootLogin no
PasswordAuthentication no
AllowUsers your_username
```

```bash
sudo systemctl restart ssh
```

---

## Instalasi Docker

Script resmi sudah termasuk Docker Compose v2 (plugin) — tidak perlu install `docker-compose` terpisah:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

docker --version && docker compose version
```

---

## Deploy n8n + WAHA

### 1. Clone repo & konfigurasi

```bash
git clone <repo-url> ~/n8n-waha && cd ~/n8n-waha

cp .env-example .env
nano .env
chmod 600 .env
```

Wajib diisi di `.env`:

| Variabel | Keterangan |
|----------|------------|
| `N8N_HOST` / `WAHA_HOST` | Domain masing-masing service |
| `N8N_ENCRYPTION_KEY` | `openssl rand -hex 32` — **jangan pernah diganti** setelah n8n jalan (kredensial tersimpan terenkripsi dengan key ini) |
| `WAHA_API_KEY` | `openssl rand -hex 32` — dikirim sebagai header `X-Api-Key` |
| `WAHA_DASHBOARD_PASSWORD` | Password login dashboard WAHA |

### 2. Jalankan

```bash
docker compose up -d
docker compose ps
docker compose logs -f
```

### 3. Setup awal aplikasi

- **n8n**: buka `https://n8n.example.com` → buat akun **owner** pada kunjungan pertama. (n8n v1+ memakai user management bawaan; basic auth sudah dihapus.)
- **WAHA**: buka `https://waha.example.com/dashboard` → login → start session `default` → scan QR dari HP.

Tes kirim pesan (atau pakai [waha-api.rest](waha-api.rest)):

```bash
curl -X POST https://waha.example.com/api/sendText \
  -H "X-Api-Key: $WAHA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"session": "default", "chatId": "6281234567890@c.us", "text": "Halo dari WAHA!"}'
```

---

## Reverse Proxy & SSL

### 1. Install Nginx + Certbot

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

### 2. Pasang config

Template ada di folder [nginx/](nginx/) — sudah termasuk header WebSocket yang **wajib** untuk editor n8n:

```bash
sudo cp nginx/n8n.conf /etc/nginx/sites-available/n8n
sudo cp nginx/waha.conf /etc/nginx/sites-available/waha
sudo nano /etc/nginx/sites-available/n8n   # ganti n8n.example.com dengan domain Anda
sudo nano /etc/nginx/sites-available/waha  # ganti waha.example.com dengan domain Anda

sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/waha /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 3. SSL (Let's Encrypt)

```bash
sudo certbot --nginx -d n8n.example.com -d waha.example.com
```

Certbot memasang auto-renewal otomatis; cek dengan `sudo certbot renew --dry-run`.

---

## Backup Otomatis

Buat `/usr/local/bin/backup-n8n-waha.sh`:

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="$HOME/backups"
PROJECT_DIR="$HOME/n8n-waha"
DATE=$(date +%Y%m%d)

mkdir -p "$BACKUP_DIR"

# Backup named volume langsung (tidak tergantung nama container)
# Prefix volume = nama folder project, cek dengan: docker volume ls
docker run --rm -v n8n-waha_n8n_data:/data -v "$BACKUP_DIR":/backup alpine \
  tar czf "/backup/n8n_$DATE.tar.gz" -C /data .

docker run --rm -v n8n-waha_waha_sessions:/data -v "$BACKUP_DIR":/backup alpine \
  tar czf "/backup/waha_$DATE.tar.gz" -C /data .

cp "$PROJECT_DIR/.env" "$BACKUP_DIR/env_$DATE"
chmod 600 "$BACKUP_DIR/env_$DATE"

# Simpan 7 hari terakhir
find "$BACKUP_DIR" -type f -mtime +7 -delete
```

Jadwalkan tiap jam 3 pagi:

```bash
chmod +x /usr/local/bin/backup-n8n-waha.sh
(crontab -l 2>/dev/null; echo "0 3 * * * /usr/local/bin/backup-n8n-waha.sh") | crontab -
```

> Backup `.env` berisi secret — simpan salinan off-server di tempat yang aman (password manager / storage terenkripsi).

---

## Update & Maintenance

```bash
cd ~/n8n-waha
docker compose pull
docker compose up -d
docker image prune -f
```

Data aman di named volume; update image tidak menghapus workflow atau session WhatsApp.

---

## Troubleshooting

| Masalah | Solusi |
|---------|--------|
| Container gagal start | `docker compose logs -f <service>` |
| Editor n8n blank / "Connection lost" | Pastikan header WebSocket ada di config Nginx (`Upgrade`, `Connection "upgrade"`) |
| n8n error "secure cookie" saat akses via IP/HTTP | Akses lewat HTTPS + domain; jangan matikan secure cookie di produksi |
| Webhook n8n memanggil URL salah | Pastikan `N8N_HOST` & `N8N_PROTOCOL` benar (menjadi `WEBHOOK_URL`) |
| WAHA session disconnect | Cek `docker stats` (RAM), restart session via dashboard |
| Kredensial n8n hilang setelah restore | `N8N_ENCRYPTION_KEY` harus sama dengan saat backup dibuat |
| SSL expired | `sudo certbot renew` lalu `sudo systemctl reload nginx` |
| Permission denied pada volume | `docker compose logs` dulu — jangan `chown` sembarangan; volume n8n dimiliki UID 1000 |

---

## Lisensi

Dokumen ini bebas digunakan untuk keperluan pribadi maupun komersial.
