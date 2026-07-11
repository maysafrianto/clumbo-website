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
Alur:Developer git push ke branch main di GitHub Repo.GitHub Actions otomatis jalan: build (jika perlu) → SSH ke VPS → docker compose pull && up -d.User akses domain → DNS resolve ke IP VPS → request HTTPS ke nginx → nginx proxy_pass ke App Container.nginx jadi reverse proxy; SSL cert dikelola certbot (Let's Encrypt), auto-renew via systemd timer / cron.Uptime Kuma polling endpoint tiap menit untuk cek uptime.Cron job backup data aplikasi tiap hari ke Cloud Storage eksternal menggunakan layanan gratis dari Backblaze.Tech StackLayerToolsAppStatic HTML (html/index.html)Web server / reverse proxynginxSSLcertbot (Let's Encrypt)ContainerDocker + Docker ComposeCI/CDGitHub ActionsMonitoringUptime KumaBackupcron job → Backblaze B2VPS Provider<Isi Nama Provider VPS Temanmu, misal: Herza/Biznet>Domain<Isi Nama Domain Yang Kamu Beli, misal: clumboudon.my.id>2. Struktur RepoPlaintext.
UAS-WEB-DEPLOYMENT/
├── .github/
│   └── workflows/
│       └── deploy.yml        # CI/CD pipeline — trigger saat push ke main
├── html/
│   └── index.html            # aplikasi static yang di-serve nginx (Clumbo Udon)
├── docker-compose.yml        # definisi App Container (nginx + static html)
└── README.md
3. Setup & Deployment3.1 Domain & DNSMain Domain: <domain-kamu.com>Sub Domain: <www.domain-kamu.com>Monitoring Domain: <monitor.domain-kamu.com>DNS A record mengarah ke IP VPS: <IP-VPS-Temanmu>3.2 VPSProvider: <Nama Provider>OS: <Ubuntu 24.04 / Sesuai VPS Temanmu>Spesifikasi: <Spesifikasi VPS Temanmu>3.3 nginx + HTTPS (certbot)Nginxserver {
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
SSL cert di-generate dengan:Bashsudo certbot --nginx -d <domain-kamu.com>
Auto-renew dicek via:Bashsudo systemctl status certbot.timer
# atau
sudo certbot renew --dry-run
3.4 CI/CD (GitHub Actions)File: .github/workflows/deploy.ymlTrigger: setiap push ke main →Checkout codeSSH ke VPSdocker compose pull && docker compose up -dSecrets yang dibutuhkan di GitHub repo (Settings → Secrets):VPS_HOSTVPS_USERVPS_SSH_KEY3.5 Container (Docker Compose)YAMLversion: '3.8'

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
3.6 MonitoringDashboard Uptime Kuma: http://monitor.<domain-kamu.com>Interval check: 60 detik3.7 LoggingLog aplikasi (nginx access/error): docker logs web_uas_clumboudon3.8 BackupCron job harian (crontab -e di VPS):Bash0 2 * * * /home/<user-kamu>/scripts/backup.sh >> /var/log/backup.log 2>&1
Isi backup.sh:Bash#!/bin/bash
BACKUP_DIR="/home/<user-kamu>/tugas-uas/html"
DEST_ZIP="/home/<user-kamu>/backup_uas_$(date +%Y%m%d_%H%M%S).zip"
BUCKET_NAME="backup-uas-clumboudon"

echo "Memulai backup folder HTML..."
zip -r $DEST_ZIP $BACKUP_DIR

echo "Mengirim file ke Cloud Storage Backblaze..."
rclone copy $DEST_ZIP mybackup:$BUCKET_NAME --s3-no-check-bucket

rm $DEST_ZIP
4. Lessons LearnedSimulasi Web Deployment Nyata: Mendapatkan pengalaman langsung bagaimana mendeploy aplikasi static html ke server produksi menggunakan arsitektur modern (Docker & Nginx) bukan sekadar dijalankan di localhost komputer.Keamanan & Otomatisasi: Belajar cara mengamankan web dengan SSL HTTPS (Certbot), membuat pipeline CI/CD otomatis menggunakan GitHub Actions, memonitor server, dan pentingnya kebijakan backup data berkala.5. 

Referensihttps://software.endy.muhardin.com/devops/deployment-microservice-kere-hore-1/