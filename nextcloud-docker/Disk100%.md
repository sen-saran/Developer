รายการรายละเอียดอาการ 
Nextcloud แสดง Internal Server Error
สาเหตุDisk /dev/vdb เต็ม 100% (983 GB / 983 GB)
ต้นเหตุจริงScript backup 
สร้างไฟล์ files_YYYY-MM-DD.tar.gz ทุกวัน ~109 GB โดยไม่ลบอันเก่า

🗂️ ไฟล์ที่กินพื้นที่
files_2026-04-28.tar.gz   109 GB
files_2026-04-29.tar.gz   109 GB
files_2026-04-30.tar.gz   109 GB
files_2026-05-02.tar.gz   109 GB
files_2026-05-03.tar.gz   109 GB
files_2026-05-04.tar.gz   109 GB
files_2026-05-05.tar.gz    75 GB  ← เก็บไว้
files_2026-05-06.tar.gz   108 GB  ← เก็บไว้
─────────────────────────────────
รวมที่ลบ                  ~654 GB

🛠️ วิธีแก้ที่ใช้
ขั้นที่ 1 — ตรวจหาไฟล์ขนาดใหญ่
find /data -name "*.sql" -o -name "*.dump" -o -name "*.tar.gz" | xargs du -sh | sort -rh

ขั้นที่ 2 — ลบ backup เก่า เก็บแค่ 2 วันล่าสุด
sudo bash -c 'ls -t /data/nextcloud/backup/files_*.tar.gz | tail -n +3 | xargs rm -f'

ขั้นที่ 3 — ตรวจสอบพื้นที่
df -h /data
# ผลลัพธ์: 984G total, 330G used, 604G free (36%)

✅ ผลลัพธ์
ก่อน:   984G used  0G free   100% 🚨
หลัง:   330G used  604G free  36% ✅

🔒 ป้องกันไม่ให้เกิดซ้ำ
bashsudo crontab -e
0 1 * * * bash -c 'ls -t /data/nextcloud/backup/files_*.tar.gz | tail -n +4 | xargs rm -f'




df
find /data -name "*.sql" -o -name "*.dump" -o -name "*.tar.gz" | xargs du -sh 2>/dev/null | sort -rh
ls -lh /data/nextcloud/backup/files_*.tar.gz
sudo bash -c 'ls -t /data/nextcloud/backup/files_*.tar.gz | tail -n +3 | xargs rm -f'
df -h /data
ls -lh /data/nextcloud/backup/
docker exec -u www-data nextcloud_app php occ maintenance:mode --off
docker restart nextcloud_app
sudo crontab -e
0 1 * * * bash -c 'ls -t /data/nextcloud/backup/files_*.tar.gz | tail -n +4 | xargs rm -f'
docker restart nextcloud_app



