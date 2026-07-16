# n8n + WAHA — WhatsApp Automation Stack

Stack Docker Compose untuk menjalankan [n8n](https://n8n.io) (workflow automation) dan [WAHA](https://waha.devlike.pro) (WhatsApp HTTP API) di VPS, lengkap dengan reverse proxy Nginx + SSL.

## Arsitektur

```
Internet ──> Nginx (80/443, SSL via Let's Encrypt)
              ├── n8n.example.com  ──> n8n  (127.0.0.1:5678)
              └── waha.example.com ──> WAHA (127.0.0.1:3000)
```

Kedua container hanya bind ke `127.0.0.1` — akses publik wajib lewat Nginx.

## Quick Start

```bash
git clone <repo-url> && cd n8n-waha

# Konfigurasi
cp .env-example .env
nano .env          # isi domain, API key, dan encryption key
chmod 600 .env

# Jalankan
docker compose up -d
docker compose ps
```

Generate secret yang kuat:

```bash
openssl rand -hex 32
```

Setup lengkap dari VPS kosong (firewall, Docker, Nginx, SSL, backup): lihat **[guide.md](guide.md)**.

## Struktur Repo

| Path | Isi |
|------|-----|
| `docker-compose.yml` | Definisi service n8n + WAHA |
| `.env-example` | Template konfigurasi — salin menjadi `.env` |
| `nginx/` | Contoh config reverse proxy untuk n8n & WAHA |
| `workflows/` | Contoh workflow n8n (import via UI n8n) |
| `waha-api.rest` | Contoh request API WAHA (VS Code REST Client) |
| `guide.md` | Panduan instalasi lengkap di VPS |

## Setelah Berjalan

1. **n8n** — buka `https://n8n.example.com`, buat akun owner pada kunjungan pertama (n8n v1+ memakai user management bawaan, bukan basic auth).
2. **WAHA** — buka dashboard `https://waha.example.com/dashboard`, login dengan `WAHA_DASHBOARD_USERNAME/PASSWORD`, mulai session dan scan QR.
3. Import [workflows/contoh-monitor-status.json](workflows/contoh-monitor-status.json) di n8n sebagai titik awal. Dari dalam n8n, WAHA bisa diakses via `http://waha:3000` (satu network Docker).

> Untuk integrasi yang lebih rapi, install community node [`@devlikeapro/n8n-nodes-waha`](https://www.npmjs.com/package/@devlikeapro/n8n-nodes-waha) di n8n (Settings → Community Nodes).

## Perintah Umum

```bash
docker compose logs -f n8n     # lihat log
docker compose pull && docker compose up -d   # update image
docker compose down            # stop (data aman di named volume)
```

## Lisensi

Bebas digunakan untuk keperluan pribadi maupun komersial.
