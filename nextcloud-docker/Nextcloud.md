# Nextcloud User Management Commands
- DICOM Viewer 
- Quota warning 
- Ransomware protection 
- GpxEdit 
- GpxPod 
- OpenStreetMap integration

- Backup + Update
```bash
# backup database
docker exec nextcloud_db pg_dump -U nextcloud nextcloud > ~/backup_db_$(date +%F).sql

# เปลี่ยน image
sed -i 's/nextcloud:29/nextcloud:30/' ~/nextcloud/docker-compose.yml

# pull และ update
cd ~/nextcloud
docker compose pull app
docker compose stop app
docker compose up -d app

# รอ 30 วินาที แล้วรัน upgrade
sleep 30
docker exec -u www-data nextcloud_app php occ upgrade
docker exec -u www-data nextcloud_app php occ maintenance:mode --off
docker exec -u www-data nextcloud_app php occ cache:flush

ตรวจสอบ
bashdocker exec -u www-data nextcloud_app php occ status
```

ผลที่ควรได้:
```
- version: 30.0.17.0
- versionstring: Nextcloud 30.0.17
อัปเดตทีละ version ครับ ข้ามไป 33 เลยไม่ได้ ต้องทำตามลำดับ:
29 → 30 → 31 → 32 → 33

cd ~/nextcloud

for VERSION in 30 31 32 33; do
  echo "=== Updating to Nextcloud $VERSION ==="
  
  # เปลี่ยน version
  sed -i "s/nextcloud:[0-9]*/nextcloud:$VERSION/" docker-compose.yml
  
  # pull และ update
  docker compose pull app
  docker compose stop app
  docker compose up -d app
  
  # รอ container พร้อม
  sleep 30
  
  # upgrade
  docker exec -u www-data nextcloud_app php occ upgrade
  docker exec -u www-data nextcloud_app php occ maintenance:mode --off
  docker exec -u www-data nextcloud_app php occ cache:flush
  
  echo "=== Done Nextcloud $VERSION ==="
  sleep 10
done

ตรวจสอบหลังเสร็จ
bashdocker exec -u www-data nextcloud_app php occ status

⚠️ ถ้า error ระหว่าง version ไหน หยุดและเอา error มาให้ดูก่อนครับ อย่า force ต่อ
```
## ดู User

```bash
top 
# ดู user ทั้งหมด
docker exec -u www-data nextcloud_app php occ user:list

# ดู quota และการใช้งานทุก user
docker exec -u www-data nextcloud_app php occ user:report

# ดูข้อมูล user เฉพาะคน
docker exec -u www-data nextcloud_app php occ user:info <username>
```

## ดู Quota/Used/Free เป็น GB พร้อม %

```bash
docker exec -u www-data nextcloud_app php occ user:info <username> | awk '
/used/    {used=$NF}
/free/    {free=$NF}
/total/   {total=$NF}
/relative/{rel=$NF}
/quota/   {quota=$NF}
END {
  printf "Quota  : %s\n", quota
  printf "Used   : %.2f GB (%.1f%%)\n", used/1073741824, rel
  printf "Free   : %.2f GB (%.1f%%)\n", free/1073741824, (free/total)*100
  printf "Total  : %.2f GB\n", total/1073741824
}'
```

## จัดการ User

```bash
# สร้าง user ใหม่
docker exec -u www-data nextcloud_app php occ user:add --password-from-env --group="users" <username>

# ลบ user
docker exec -u www-data nextcloud_app php occ user:delete <username>

# เปิด user
docker exec -u www-data nextcloud_app php occ user:enable <username>

# ปิด user
docker exec -u www-data nextcloud_app php occ user:disable <username>
```

## แก้ไข User

```bash
# เปลี่ยน password
docker exec -u www-data nextcloud_app php occ user:resetpassword <username>

# เปลี่ยน display name
docker exec -u www-data nextcloud_app php occ user:modify <username> displayname "ชื่อใหม่"

# เปลี่ยน email
docker exec -u www-data nextcloud_app php occ user:modify <username> email "email@domain.com"

# เปลี่ยน quota (ตัวอย่าง 10GB)
docker exec -u www-data nextcloud_app php occ user:setting <username> files quota 10GB

# ไม่จำกัด quota
docker exec -u www-data nextcloud_app php occ user:setting <username> files quota none
```

## จัดการ Group

```bash
# ดู group ทั้งหมด
docker exec -u www-data nextcloud_app php occ group:list

# สร้าง group
docker exec -u www-data nextcloud_app php occ group:add <groupname>

# เพิ่ม user เข้า group
docker exec -u www-data nextcloud_app php occ group:adduser <groupname> <username>

# ลบ user ออกจาก group
docker exec -u www-data nextcloud_app php occ group:removeuser <groupname> <username>
```
