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