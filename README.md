# 🍜 Clumbo Udon — Production Deployment

> UAS Sistem Operasi + Jaringan Komputer — STMIK Tazkia

Static HTML site yang dideploy ke VPS dengan domain HTTPS (via nginx + certbot), auto-deploy lewat GitHub Actions, monitoring via Uptime Kuma, dan backup harian otomatis menggunakan layanan gratis dari Backblaze.

---

## 1. Architecture Diagram

```text
Developer laptop
      │ git push (main)
      ▼
GitHub Repo ──trigger──► GitHub Actions (CI/CD)
                                 │
                                 │ SSH deploy
                                 ▼
                         ┌─────────────── VPS Server ───────────────┐
                         │                                          │
   User Browser          │   nginx :80,:443  ──proxy_pass──►   App  │
   ──resolve domain──►   │   (SSL via certbot)                Container│
   (Domain DNS A record) │        ▲                          (Docker)│
                         │        │ health check                    │
                         │   Uptime Kuma (sibling container)        │
                         │                                          │
                         │   cron job ──daily backup──►  Cloud      │
                         │                               Storage    │
                         │                             (S3/R2/B2)   │
                         └──────────────────────────────────────────┘


**Alur:**
1. Developer `git push` ke branch `main` di GitHub Repo.
2. GitHub Actions otomatis jalan: build (jika perlu) → SSH ke VPS → `docker compose pull && up -d`.
3. User akses domain → DNS resolve ke IP VPS → request HTTPS ke nginx → nginx `proxy_pass` ke App Container.
4. nginx jadi reverse proxy; SSL cert dikelola **certbot** (Let's Encrypt), auto-renew via systemd timer / cron.
5. **Uptime Kuma** polling endpoint tiap menit untuk cek uptime.
6. **Cron job** backup data aplikasi tiap hari ke Cloud Storage eksternal menggunakan layanan gratis dari Backblaze.

### Tech Stack
| Layer | Tools |
|---|---|
| App | Static HTML (`html/index.html`) |
| Web server / reverse proxy | nginx |
| SSL | certbot (Let's Encrypt) |
| Container | Docker + Docker Compose |
| CI/CD | GitHub Actions |
| Monitoring | Uptime Kuma |
| Backup | cron job → Backblaze B2 |
| VPS Provider | `<Isi Nama Provider VPS Temanmu, misal: Herza/Biznet>` |
| Domain | `<Isi Nama Domain Yang Kamu Beli, misal: clumboudon.my.id>` |

## 2. Struktur Repo

```text
.
UAS-WEB-DEPLOYMENT/
├── .github/
│   └── workflows/
│       └── deploy.yml        # CI/CD pipeline — trigger saat push ke main
├── html/
│   └── index.html            # aplikasi static yang di-serve nginx (Clumbo Udon)
├── docker-compose.yml        # definisi App Container (nginx + static html)
└── README.md


---

## 3. Setup & Deployment

### 3.1 Domain & DNS
*   Main Domain: `<domain-kamu.com>`
*   Sub Domain: `<www.domain-kamu.com>`
*   Monitoring Domain: `<monitor.domain-kamu.com>`
*   DNS A record mengarah ke IP VPS: `<IP-VPS-Temanmu>`

### 3.2 VPS
*   Provider: `<Nama Provider>`
*   OS: `<Ubuntu 24.04 / Sesuai VPS Temanmu>`
*   Spesifikasi: `<Spesifikasi VPS Temanmu>`

### 3.3 nginx + HTTPS (certbot)

```nginx
server {
    server_name <domain-kamu.com> [www.domain-kamu.com](https://www.domain-kamu.com);

    location / {
        proxy_pass [http://127.0.0.1:8080](http://127.0.0.1:8080);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/<domain-kamu.com>/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/<domain-kamu.com>/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
SSL cert di-generate dengan:

Bash
sudo certbot --nginx -d <domain-kamu.com>
Auto-renew dicek via:

Bash
sudo systemctl status certbot.timer
# atau
sudo certbot renew --dry-run
3.4 CI/CD (GitHub Actions)
File: .github/workflows/deploy.yml

Trigger: setiap push ke main →

Checkout code

SSH ke VPS

docker compose pull && docker compose up -d

Secrets yang dibutuhkan di GitHub repo (Settings → Secrets):

VPS_HOST

VPS_USER

VPS_SSH_KEY

3.5 Container (Docker Compose)
YAML
version: '3.8'

services:
  web-uas:
    image: nginx:latest
    container_name: web_uas_clumboudon
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    restart: always

  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: monitoring_uas_clumboudon
    ports:
      - "3001:3001" 
    volumes:
      - uptime-kuma-data:/app/data
    restart: always

volumes:
  uptime-kuma-data:
3.6 Monitoring
Dashboard Uptime Kuma: http://monitor.<domain-kamu.com>

Interval check: 60 detik

3.7 Logging
Log aplikasi (nginx access/error): docker logs web_uas_clumboudon

3.8 Backup
Cron job harian (crontab -e di VPS):

Bash
0 2 * * * /home/<user-kamu>/scripts/backup.sh >> /var/log/backup.log 2>&1
Isi backup.sh:

Bash
#!/bin/bash
BACKUP_DIR="/home/<user-kamu>/tugas-uas/html"
DEST_ZIP="/home/<user-kamu>/backup_uas_$(date +%Y%m%d_%H%M%S).zip"
BUCKET_NAME="backup-uas-clumboudon"

echo "Memulai backup folder HTML..."
zip -r $DEST_ZIP $BACKUP_DIR

echo "Mengirim file ke Cloud Storage Backblaze..."
rclone copy $DEST_ZIP mybackup:$BUCKET_NAME --s3-no-check-bucket

rm $DEST_ZIP
4. Lessons Learned
Simulasi Web Deployment Nyata: Mendapatkan pengalaman langsung bagaimana mendeploy aplikasi static html ke server produksi menggunakan arsitektur modern (Docker & Nginx) bukan sekadar dijalankan di localhost komputer.

Keamanan & Otomatisasi: Belajar cara mengamankan web dengan SSL HTTPS (Certbot), membuat pipeline CI/CD otomatis menggunakan GitHub Actions, memonitor server, dan pentingnya kebijakan backup data berkala.

5. Referensi
https://software.endy.muhardin.com/devops/deployment-microservice-kere-hore-1/