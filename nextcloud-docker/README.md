# Nextcloud Docker Installation Guide
### Ubuntu 24.04 LTS + PostgreSQL + Cloudflare Tunnel
---
โครงสร้างรวมทั้งหมด:
```
/data/nextcloud/          ← disk 1TB
├── db/                   ← PostgreSQL data
├── files/                ← Nextcloud files
└── backup/               ← backup files

~/nextcloud/              ← config (disk 100GB)
├── secrets/
│   └── .env
├── nginx/
│   ├── default.conf
│   └── ssl/
│       ├── fullchain.pem
│       └── privkey.pem
└── docker-compose.yml

## ⚠️ Checklist ก่อน Go-Live

- [ ] SSH port ใหม่ใช้งานได้
- [ ] UFW เปิดเฉพาะ port ที่จำเป็น
- [ ] Fail2ban ทำงานปกติ: `sudo fail2ban-client status`
- [ ] Disk /data mount ถูกต้อง: `df -h`
- [ ] Docker ทุก container `Up`: `docker compose ps`
- [ ] เข้า Nextcloud ได้: `http://159.138.255.6`
- [ ] Login ด้วย admin ได้
- [ ] Upload/Download ไฟล์ได้
- [ ] Backup ทำงาน: `docker compose logs backup`
- [ ] Auto Security Updates เปิดอยู่
