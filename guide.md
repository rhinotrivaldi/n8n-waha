# Panduan Instalasi n8n & WAHA di VPS dengan Docker

## 📝 Daftar Isi
1. [Spesifikasi Minimal VPS](#-spesifikasi-minimal-vps)
2. [Setup Awal VPS](#-setup-awal-vps)
3. [Konfigurasi Keamanan](#-konfigurasi-keamanan)
4. [Instalasi Docker](#-instalasi-docker)
5. [Deploy n8n + WAHA](#-deploy-n8n--waha)
6. [Optimasi & Monitoring](#-optimasi--monitoring)
7. [Troubleshooting](#-troubleshooting)

---

## 🖥️ Spesifikasi Minimal VPS
- **OS**: Ubuntu 20.04/22.04 LTS
- **CPU**: 2 vCPU cores
- **RAM**: 4 GB
- **Storage**: 25 GB SSD
- **Port Terbuka**: 22 (SSH), 80 (HTTP), 443 (HTTPS)

---

## 🛠️ Setup Awal VPS

### Login ke VPS
```bash
ssh -p 22 username@ip-vps
```

### Update Sistem
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git nano htop ufw
```

---

## 🔒 Konfigurasi Keamanan

### 1. Setup Firewall (UFW)
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### 2. Hardening SSH
Edit file config:

```bash
sudo nano /etc/ssh/sshd_config
```

Ubah parameter:

```ini
Port 2222
PermitRootLogin no
PasswordAuthentication no
AllowUsers your_username
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

---

## 🐳 Instalasi Docker

### 1. Install Docker Engine
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Install Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 3. Verifikasi Instalasi
```bash
docker --version && docker-compose --version
```

---

## 🚀 Deploy n8n + WAHA

### 1. Buat Direktori Proyek
```bash
mkdir ~/automation && cd ~/automation
```

### 2. Buat file .env untuk konfigurasi keamanan
```bash
nano .env
```

Isi dengan:

```env
# n8n credentials
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=password_kuat_n8n
N8N_ENCRYPTION_KEY=kunci_enkripsi_yang_sangat_rahasia_dan_panjang
N8N_PORT=5678

# URL dan domain
N8N_HOST=n8n.example.com
N8N_PROTOCOL=https
N8N_EMAIL_MODE=smtp

# SMTP settings
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=user@example.com
SMTP_PASS=smtp_password
SMTP_SENDER=n8n@example.com

# WAHA configuration
WAHA_PORT=3000
WHATSAPP_DEFAULT_ENGINE=NOWEB
WAHA_API_KEY=api_key_yang_sangat_rahasia
```

### 3. Buat docker-compose.yml yang menggunakan file .env
```bash
nano docker-compose.yml
```

Isi dengan:

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports:
      - "${N8N_PORT}:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=${N8N_BASIC_AUTH_ACTIVE}
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_HOST=${N8N_HOST}
      - N8N_PROTOCOL=${N8N_PROTOCOL}
      - N8N_EMAIL_MODE=${N8N_EMAIL_MODE}
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_PORT=${SMTP_PORT}
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASS=${SMTP_PASS}
      - SMTP_SENDER=${SMTP_SENDER}
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - automation_network

  waha:
    image: devlikeapro/waha
    restart: unless-stopped
    ports:
      - "${WAHA_PORT}:3000"
    volumes:
      - waha_sessions:/app/.sessions
    environment:
      - WHATSAPP_DEFAULT_ENGINE=${WHATSAPP_DEFAULT_ENGINE}
      - API_KEY=${WAHA_API_KEY}
    networks:
      - automation_network

networks:
  automation_network:
    driver: bridge

volumes:
  n8n_data:
  waha_sessions:
```

### 4. Jalankan Kontainer
```bash
docker-compose up -d
```

### 5. Verifikasi
- n8n: http://ip-vps:5678 atau https://n8n.example.com
- WAHA API: http://ip-vps:3000

Cek status:

```bash
docker-compose ps
```

### 6. Keamanan File .env
Pastikan file .env memiliki permission yang aman:

```bash
chmod 600 .env
```

---

## ⚡ Optimasi & Monitoring

### 1. Reverse Proxy dengan Nginx
```bash
sudo apt install -y nginx certbot python3-certbot-nginx
sudo nano /etc/nginx/sites-available/n8n
```

Contoh config:

```nginx
server {
    listen 80;
    server_name n8n.example.com;
    
    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Aktifkan konfigurasi dan SSL:

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo certbot --nginx -d n8n.example.com
```

Buat juga config untuk WAHA API jika diperlukan:

```bash
sudo nano /etc/nginx/sites-available/waha
```

Contoh config:

```nginx
server {
    listen 80;
    server_name waha.example.com;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Aktifkan konfigurasi dan SSL:

```bash
sudo ln -s /etc/nginx/sites-available/waha /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo certbot --nginx -d waha.example.com
```

### 2. Backup Otomatis
Buat script `/usr/local/bin/backup_automation.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/home/user/backups"
DATE=$(date +%Y%m%d)

# Buat direktori backup jika belum ada
mkdir -p $BACKUP_DIR

# Backup n8n data
docker run --rm --volumes-from automation_n8n_1 -v $BACKUP_DIR:/backup ubuntu tar czf /backup/n8n_backup_$DATE.tar.gz /home/node/.n8n

# Backup file .env
cp /home/user/automation/.env $BACKUP_DIR/env_backup_$DATE.txt

# Backup WAHA sessions
docker run --rm --volumes-from automation_waha_1 -v $BACKUP_DIR:/backup ubuntu tar czf /backup/waha_backup_$DATE.tar.gz /app/.sessions

# Rotasi backup (simpan hanya 7 backup terakhir)
find $BACKUP_DIR -name "n8n_backup_*.tar.gz" -type f -mtime +7 -delete
find $BACKUP_DIR -name "env_backup_*.txt" -type f -mtime +7 -delete
find $BACKUP_DIR -name "waha_backup_*.tar.gz" -type f -mtime +7 -delete
```

Jadwalkan dengan crontab:

```bash
chmod +x /usr/local/bin/backup_automation.sh
crontab -e
```

Tambahkan baris:

```
0 3 * * * /usr/local/bin/backup_automation.sh
```

---

## 🐞 Troubleshooting

| Masalah | Solusi |
|---------|--------|
| Port tidak terbuka | `sudo ufw allow port/tcp` |
| Kontainer gagal start | `docker-compose logs -f nama_service` |
| WAHA disconnect | Tambahkan resource limit di docker-compose.yml |
| Lupa kredensial | Kredensial tersimpan di file `.env` |
| Permission denied pada volume | `sudo chown -R 1000:1000 /path/ke/volume` |
| SSL Certificates error | Perbarui sertifikat: `sudo certbot renew` |

---

## 📜 License
Dokumen ini dapat digunakan secara bebas untuk keperluan pribadi maupun komersial.

---

### Cara Menggunakan File Ini:
1. Simpan sebagai `INSTALL_GUIDE.md`
2. Edit file `.env` sesuai kebutuhan dengan kredensial yang kuat
3. Pastikan untuk mengganti:
   - Domain `n8n.example.com` dan `waha.example.com`
   - Password dan kunci enkripsi dengan nilai yang kuat dan unik
   - SMTP settings sesuai dengan provider email Anda

### Fitur Dokumen Ini:
✅ Format terstruktur dengan daftar isi  
✅ Sintaks code block yang siap copy-paste  
✅ Konfigurasi keamanan dengan file `.env`  
✅ Tabel troubleshooting cepat  
✅ Kompatibel dengan GitHub/GitLab Markdown