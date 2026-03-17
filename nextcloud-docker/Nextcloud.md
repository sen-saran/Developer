# Nextcloud User Management Commands
- DICOM Viewer 
- Quota warning 
- Ransomware protection 
- GpxEdit 
- GpxPod 
- OpenStreetMap integration 
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