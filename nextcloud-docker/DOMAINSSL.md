### Step 1: วาง SSL Certificate
## รวมไฟล์ใน Windows CMD
```
cd C:\Users\admin\Desktop\owncloud-stack\nginx\ssl
dir
# CMD
type certificate.crt Intemediate.crt > fullchain.pem

#  PowerShell 
Get-Content certificate.crt, Intemediate.crt | Set-Content fullchain.pem

# Copy ไปยัง Server ด้วย SCP
scp -P 717 fullchain.pem digitalsaran@159.138.255.6:~/nextcloud/nginx/ssl/
scp -P 717 privateKey.key digitalsaran@159.138.255.6:~/nextcloud/nginx/ssl/privkey.pem
ls -la ~/nextcloud/nginx/ssl/
chmod 600 ~/nextcloud/nginx/ssl/*.pem
ls -la ~/nextcloud/nginx/ssl/
# ผลที่ควรได้:

-rw------- fullchain.pem
-rw------- privkey.pem
```
## รวมไฟล์ใน Linux
```
# copy และ rename ให้ตรง
cp /path/to/your.crt ~/nextcloud/nginx/ssl/fullchain.pem
cp /path/to/your.key ~/nextcloud/nginx/ssl/privkey.pem

# รวม Certificate + Intermediate
cat certificate.crt Intermediate.crt > ~/nextcloud/nginx/ssl/fullchain.pem

# Copy Private Key
cp privateKey.key ~/nextcloud/nginx/ssl/privkey.pem

# ล็อค Permission
chmod 600 ~/nextcloud/nginx/ssl/*.pem
ls -la ~/nextcloud/nginx/ssl/
ผลที่ควรได้:
-rw------- fullchain.pem
-rw------- privkey.pem
```

### Step 2: เปิดใช้ SSL Config
```
ls nano ~/nextcloud/nginx/

แก้ domain ใน ssl.conf:
nano ~/nextcloud/nginx/default.conf 
ตรวจสอบบรรทัด server_name ให้ตรงกับ domain จริง:
server_name nextcloud.dms.go.th;  ← แก้ให้ตรง
```

### Step 3: แก้ docker-compose.yml
```
nano ~/nextcloud/docker-compose.yml
เพิ่ม mount ssl.conf เข้าไปใน nginx volumes:
nginx:
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ./nginx/ssl.conf:/etc/nginx/conf.d/ssl.conf
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - /data/nextcloud/files:/var/www/html:ro
```

### Step 4: แก้ Nextcloud Config
```
# แก้ให้ชี้ domain และ https
docker exec -u www-data nextcloud_app php occ config:system:set overwrite.cli.url --value="https://nextcloud.dms.go.th"
docker exec -u www-data nextcloud_app php occ config:system:set overwritehost --value="nextcloud.dms.go.th"
docker exec -u www-data nextcloud_app php occ config:system:set overwriteprotocol --value="https"
docker exec -u www-data nextcloud_app php occ config:system:set trusted_domains 0 --value="nextcloud.dms.go.th"
แก้ใน .env ด้วย:
nano ~/nextcloud/.env
NEXTCLOUD_DOMAIN=nextcloud.dms.go.th
OVERWRITEPROTOCOL=https
OVERWRITEHOST=nextcloud.dms.go.th
```

### Step 5: แก้ docker-compose.yml environment
```
nano ~/nextcloud/docker-compose.yml
environment:
      - OVERWRITEHOST=${NEXTCLOUD_DOMAIN}
      - OVERWRITEPROTOCOL=https
      - NEXTCLOUD_TRUSTED_DOMAINS=${NEXTCLOUD_DOMAIN}
```

### Step 6: Reload nginx และ Restart
```
cd ~/nextcloud

# reload nginx ก่อน (ไม่ต้อง down)
docker compose exec nginx nginx -t          # ทดสอบ config
docker compose exec nginx nginx -s reload   # reload

# ถ้า config ผิดค่อย down แล้ว up ใหม่
docker compose down
docker compose up -d
```
### Step 7: ทดสอบ
```
**# ทดสอบ SSL
curl -I https://nextcloud.dms.go.th

# ทดสอบ redirect HTTP → HTTPS
curl -I http://nextcloud.dms.go.th
```

ผลที่ควรได้:
```
HTTP/2 200
```
