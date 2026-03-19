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

## ห้องเก็บไฟล์แต่ละ user

```
sudo ls -la /data/nextcloud/files/data/

/data/nextcloud/files/data/ 
├── digitalsaran/
│   └── files/        ← ไฟล์ของ digitalsaran
├── user01/
│   └── files/        ← ไฟล์ของ user01
└── user02/
    └── files/        ← ไฟล์ของ user02
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
# Nextcloud Administration Guide

## 1. Monitoring

```bash
# ดู resource การใช้งาน
docker stats

# ดู disk เหลือ
df -h

# ดู log error realtime
docker compose logs -f app | grep -i error

# ดูสถานะ container
docker compose ps
```

---

## 2. Backup Strategy

ตอนนี้ backup แค่ใน server เดียวกัน **ควรเพิ่ม offsite backup**

```bash
# ส่ง backup ไป server อื่น
rsync -av /data/nextcloud/backup/ user@backup-server:/backup/nextcloud/

# หรือ rclone ไป S3/Google Drive
rclone copy /data/nextcloud/backup/ gdrive:nextcloud-backup/

# ดู backup files
ls -lh /data/nextcloud/backup/

# backup manual
docker exec nextcloud_db pg_dump -U nextcloud nextcloud > ~/backup_db_$(date +%F).sql
tar czf ~/backup_files_$(date +%F).tar.gz /data/nextcloud/files/
```

---

## 3. SSL Certificate

```bash
# เช็ควันหมดอายุ cert
openssl x509 -enddate -noout -in ~/nextcloud/nginx/ssl/fullchain.pem

# copy cert ใหม่เมื่อต่ออายุแล้ว
cp /path/to/new/fullchain.pem ~/nextcloud/nginx/ssl/
cp /path/to/new/privkey.pem ~/nextcloud/nginx/ssl/
chmod 600 ~/nextcloud/nginx/ssl/*.pem

# reload nginx
cd ~/nextcloud
docker compose exec nginx nginx -s reload
```

> ⚠️ ตั้ง reminder ก่อนหมดอายุ 30 วัน

---

## 4. Log Management

```bash
# ดู log size
sudo du -sh /data/nextcloud/files/data/nextcloud.log

# ตั้ง log level (0=debug, 1=info, 2=warning, 3=error, 4=fatal)
docker exec -u www-data nextcloud_app php occ log:manage --level=2

# ดู log ล่าสุด
docker exec -u www-data nextcloud_app php occ log:tail 20

# ดู log realtime
docker compose logs -f app
```

---

## 5. Database Maintenance

```bash
# vacuum database (ทำเดือนละครั้ง)
docker exec nextcloud_db psql -U nextcloud -c "VACUUM ANALYZE;"

# เช็ค database size
docker exec nextcloud_db psql -U nextcloud -c "\l+"

# เช็ค missing indexes
docker exec -u www-data nextcloud_app php occ db:add-missing-indices

# repair database
docker exec -u www-data nextcloud_app php occ maintenance:repair
```

---

## 6. Security Checklist (ทำทุกเดือน)

```bash
# เช็ค failed login
sudo fail2ban-client status nextcloud

# เช็ค banned IPs
sudo fail2ban-client status sshd

# เช็ค security warnings
docker exec -u www-data nextcloud_app php occ status

# scan files ทั้งหมด
docker exec -u www-data nextcloud_app php occ files:scan --all

# เช็ค ufw
sudo ufw status numbered
```

---

## 7. Update Strategy

| รายการ | ความถี่ | หมายเหตุ |
|---|---|---|
| Nextcloud minor update | ทุกเดือน | เช็ค changelog ก่อน |
| Nextcloud major update | ทดสอบก่อน 1-2 สัปดาห์ | อัปทีละ version |
| Docker images | ทุก 3 เดือน | `docker compose pull` |
| Ubuntu security updates | อัตโนมัติ | unattended-upgrades |
| SSL cert | ก่อนหมดอายุ 30 วัน | แจ้ง IT ต่ออายุ |

### ขั้นตอน Update Nextcloud

```bash
# 1. backup ก่อนเสมอ
docker exec nextcloud_db pg_dump -U nextcloud nextcloud > ~/backup_db_$(date +%F).sql

# 2. เปลี่ยน version ใน docker-compose.yml
nano ~/nextcloud/docker-compose.yml
# image: nextcloud:XX → image: nextcloud:YY

# 3. pull และ update
cd ~/nextcloud
docker compose pull app
docker compose stop app
docker compose up -d app

# 4. รอ 30 วินาที แล้ว upgrade
sleep 30
docker exec -u www-data nextcloud_app php occ upgrade
docker exec -u www-data nextcloud_app php occ maintenance:mode --off
docker compose restart app
```

---

## 8. Performance Tuning

```bash
# เช็ค Redis hit rate
docker exec nextcloud_redis redis-cli -a ${REDIS_PASSWORD} info stats | grep hit

# เช็ค memory usage
docker stats --no-stream

# เช็ค disk I/O
iostat -x 1 5

# เช็ค CPU/RAM
htop
```

### PHP Settings (~/nextcloud/php.ini)

```ini
memory_limit = 2048M
upload_max_filesize = 10G
post_max_size = 10G
max_execution_time = 3600
max_input_time = 3600
```

---

## 9. User Management

```bash
# ดู user ทั้งหมด
docker exec -u www-data nextcloud_app php occ user:list

# ดู storage แต่ละ user
docker exec -u www-data nextcloud_app php occ user:report

# ดู quota และ storage รายคน
docker exec -u www-data nextcloud_app php occ user:info <username>

# สร้าง user
docker exec -u www-data nextcloud_app php occ user:add --password-from-env --group="users" <username>

# ตั้ง quota
docker exec -u www-data nextcloud_app php occ user:setting <username> files quota 10GB

# reset password
docker exec -u www-data nextcloud_app php occ user:resetpassword <username>

# ปิด user
docker exec -u www-data nextcloud_app php occ user:disable <username>

# ลบ user
docker exec -u www-data nextcloud_app php occ user:delete <username>
```

### Best Practices

- ตั้ง **quota** ทุก user ไม่ปล่อยให้ใช้ไม่จำกัด
- สร้าง **group** แยกตามแผนก
- กำหนด **admin** สำรองอย่างน้อย 1 คน
- เปิด **2FA** บังคับทุก user

---

## 10. Disaster Recovery

### กรณี Database เสีย

```bash
# หยุด app ก่อน
docker compose stop app

# restore database
docker exec -i nextcloud_db psql -U nextcloud nextcloud < ~/backup_db_YYYY-MM-DD.sql

# รัน app ใหม่
docker compose start app
docker exec -u www-data nextcloud_app php occ maintenance:mode --off
```

### กรณี Files เสีย

```bash
# หยุดทุกอย่าง
docker compose down

# restore files
tar xzf ~/backup_files_YYYY-MM-DD.tar.gz -C /

# รันใหม่
docker compose up -d
docker exec -u www-data nextcloud_app php occ files:scan --all
```

### กรณี Server พัง ย้าย Server ใหม่

```bash
# 1. ติดตั้ง Docker บน server ใหม่
# 2. copy config
rsync -av ~/nextcloud/ newserver:~/nextcloud/

# 3. copy data
rsync -av /data/nextcloud/ newserver:/data/nextcloud/

# 4. รันบน server ใหม่
ssh newserver
cd ~/nextcloud
docker compose up -d
```

> ⚠️ ทดสอบ restore backup ทุก 3 เดือน เพื่อให้แน่ใจว่า backup ใช้งานได้จริง

---

## 11. Disk Management

```bash
# เช็คพื้นที่ทั้งหมด
df -h

# เช็คขนาดแต่ละโฟลเดอร์
sudo du -sh /data/nextcloud/*/

# เช็คไฟล์ใหญ่สุด
sudo du -sh /data/nextcloud/files/data/*/ | sort -rh | head -10

# ลบ backup เก่าเกิน 30 วัน (manual)
find /data/nextcloud/backup/ -mtime +30 -delete
```

### ถ้า Disk เต็ม

```bash
# วิธีที่ 1: ขยาย disk เดิมบน Huawei Console แล้วรัน
sudo growpart /dev/vdb 1
sudo resize2fs /dev/vdb

# วิธีที่ 2: เพิ่ม disk ใหม่
sudo mkfs.ext4 /dev/vdc
sudo mkdir -p /data2
sudo mount /dev/vdc /data2
```
---

## To-Do

- [ ] ตั้ง Cloudflare WAF (CNAME)
- [ ] Pentest / Security scan
- [ ] เพิ่ม users
- [ ] ตั้ง quota แต่ละ user
- [ ] Monitoring (UptimeRobot)
- [ ] ต่ออายุ SSL cert ก่อนหมดอายุ
