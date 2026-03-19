# สรุปการติดตั้ง Nextcloud 33 Production

## Infrastructure

| รายการ | ค่า |
|---|---|
| Server | Huawei Cloud ECS |
| OS | Ubuntu 24.04 LTS |
| Disk OS | 100GB (vda) |
| Disk Data | 1TB (vdb) mount `/data` |
| Domain | dmbcloud.dms.go.th |
| IP | 159.138.255.6 |

---

## Docker Services

| Container | Image | หน้าที่ |
|---|---|---|
| nextcloud_app | nextcloud:33 | Web application |
| nextcloud_db | postgres:16-alpine | Database |
| nextcloud_redis | redis:alpine | Cache/Session |
| nextcloud_nginx | nginx:stable | Reverse proxy + SSL |
| nextcloud_backup | postgres:16-alpine | Backup อัตโนมัติ |

---

## Security

| รายการ | สถานะ |
|---|---|
| SSL/HTTPS | ✅ cert ของหน่วยงาน |
| HTTP → HTTPS redirect | ✅ |
| Security Headers | ✅ |
| UFW Firewall | ✅ port 80, 443, 717 |
| SSH Custom Port | ✅ port 717 |
| Fail2ban | ✅ |
| Auto Security Updates | ✅ |
| Docker Rootless Linger | ✅ |

---

## Nextcloud Configuration

| รายการ | สถานะ |
|---|---|
| Database indexes | ✅ |
| Mimetype migration | ✅ |
| Background jobs (cron) | ✅ |
| Phone region (TH) | ✅ |
| Maintenance window | ✅ |
| Redis cache | ✅ |
| Email (SMTP) | ✅ outgoing.workd.go.th:587 STARTTLS |
| Server ID | ✅ dmbcloud-prod-01 |
| 2FA | ✅ |
| Incompatible apps removed | ✅ gpxedit, gpxmotion, gpxpod, files_trackdownloads, ransomware_protection, encryption |

---

## File Structure

```
~/nextcloud/               ← config (100GB disk)
├── .env                   ← credentials
├── docker-compose.yml
├── php.ini
└── nginx/
    ├── ssl.conf
    └── ssl/
        ├── fullchain.pem
        └── privkey.pem

/data/nextcloud/           ← data (1TB disk)
├── db/                    ← PostgreSQL
├── files/                 ← Nextcloud files
└── backup/                ← backup อัตโนมัติ
```

---

## คำสั่งที่ใช้บ่อย

```bash
# ดูสถานะ container
docker compose ps

# ดู logs
docker compose logs -f app

# restart app
docker compose restart app

# หยุดทั้งหมด
docker compose down

# รันใหม่
docker compose up -d

# backup database manual
docker exec nextcloud_db pg_dump -U nextcloud nextcloud > ~/backup_$(date +%F).sql

# ดู user ทั้งหมด
docker exec -u www-data nextcloud_app php occ user:list

# ดู storage แต่ละ user
docker exec -u www-data nextcloud_app php occ user:report

# ดู warnings
docker exec -u www-data nextcloud_app php occ status

# เพิ่ม missing indexes
docker exec -u www-data nextcloud_app php occ db:add-missing-indices

# flush cache
docker compose restart app

# เปิด/ปิด maintenance mode
docker exec -u www-data nextcloud_app php occ maintenance:mode --on
docker exec -u www-data nextcloud_app php occ maintenance:mode --off
```

---

## User Management

```bash
# สร้าง user
docker exec -u www-data nextcloud_app php occ user:add --password-from-env --group="users" <username>

# ดู quota และ storage
docker exec -u www-data nextcloud_app php occ user:info <username>

# ตั้ง quota
docker exec -u www-data nextcloud_app php occ user:setting <username> files quota 10GB

# reset password
docker exec -u www-data nextcloud_app php occ user:resetpassword <username>

# ลบ user
docker exec -u www-data nextcloud_app php occ user:delete <username>
```

---

## Disk Management

```bash
# เช็คพื้นที่
df -h

# ดูขนาดข้อมูล Nextcloud
sudo du -sh /data/nextcloud/*/

# เช็ค backup
ls -lh /data/nextcloud/backup/
```

---

## Update Nextcloud

```bash
# backup ก่อนเสมอ
docker exec nextcloud_db pg_dump -U nextcloud nextcloud > ~/backup_db_$(date +%F).sql

# เปลี่ยน version ใน docker-compose.yml
nano ~/nextcloud/docker-compose.yml
# image: nextcloud:XX

# pull และ update
docker compose pull app
docker compose stop app
docker compose up -d app
sleep 30
docker exec -u www-data nextcloud_app php occ upgrade
docker exec -u www-data nextcloud_app php occ maintenance:mode --off
```

---

## To-Do

- [ ] ตั้ง Cloudflare WAF (CNAME)
- [ ] Pentest / Security scan
- [ ] เพิ่ม users
- [ ] ตั้ง quota แต่ละ user
- [ ] Monitoring (UptimeRobot)
- [ ] ต่ออายุ SSL cert ก่อนหมดอายุ
