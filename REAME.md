# Docker for Developers 🚀

คู่มือสรุปคำสั่ง Docker ที่จำเป็นสำหรับการพัฒนาซอฟต์แวร์ ตั้งแต่การจัดการ Container ไปจนถึงการทำ Orchestration เบื้องต้นด้วย Compose

#### 1. Infrastructure Docker Container
<======= vm =======><======= vm =======><======= vm =======>
<========= Network ===========><======== Network ==========>  
<====================== Docker Engine =====================>
<======== Network ==========><======= IPAM Driver =========>  
<================= Network Infrastructure =================>

#### 2. คำสั่งจัดการสถานะ (Container Management)
### คำสั่งจัดการ Container
```bash
docker stop <container_id>                     # สั่งหยุดการทำงาน (Transition to Stopped)
docker start <container_id>                    # สั่งให้กลับมาทำงานใหม่ (Transition to Running)
docker rm -f <container_id>                    # ลบ Container ทิ้งทันที (Force Remove)
docker run ngnix
docker ps                                      # ดู Container ที่กำลังรันอยู่ (Running)
docker ps -a                                   # ดู Container ทั้งหมด (รวมที่หยุดรันหรือแค่สร้างไว้)
docker exec <container_id> ls                  # ดู Container รายการไฟล์ทั้งหมด
docker exec -it <container_id> sh              # sh Container pwd whoami
docker log <container_id>                      # ดู log
docker log -f <container_id>                   # ดู log follow
docker run -p 8800:80 ngnix                    # foreground run host(os):8800 container:80(dockerhub)
docker run -d -p 9900:80 ngnix                 # background
docker run --name mynginx -d -p 7700:80 nginx  # background เพื่อป้องกันการทำงานชนกัน
```
#### 3. tester Container
### คำสั่งจัดการ Container
```bash
docker run --name mynginx -d -p 7700:80 nginx 
docker logs mynginx
docker exec -it mynginx sh
ls
cd usr/share/nginx/html
cat index.html
sudo apt install nano
nano index.html
http://localhost:7700
```
#### 4. WORKSHOP WordPress + MySQL + phpMyAdmin
##### 4.1 เตรียมไฟล์ images ที่ต้องการใช้งาน
- WordPress: `docker pull wordpress`
- MySQL: `docker pull mysql:5.7`
- phpMyAdmin: `docker pull phpmyadmin/phpmyadmin`
```bash
# <========== pull ============>
docker pull wordpress
docker pull mysql:5.7
docker pull phpmyadmin
```
##### 4.2 การสร้าง Network ใน Docker
```bash
# ดูรายการ network ที่มีอยู่
docker network ls
# สร้าง network ชื่อ wordpress
docker network create wordpress
```

##### 4.3 การรัน MySQL Container
```bash
docker run --name mysql -e MYSQL_ROOT_PASSWORD=1234 -e MYSQL_DATABASE=wordpress_db -e MYSQL_USER=wordpress -e MYSQL_PASSWORD=wordpress -d mysql:5.7
```
##### 4.4 การรัน phpMyAdmin Container
```bash
docker run --name pma -e PMA_ARBITRARY=1 -p 8888:80 -d phpmyadmin
```
##### 4.5 เชื่อม Container เข้ากับ Network ที่สร้างไว้
```bash
docker network ls
docker network connect wordpress mysql
docker network connect wordpress pma
```

##### 4.6 เช็คว่าใครอยู่ใน network ชื่อ wordpress บ้าง ?
```bash
docker network inspect wordpress
```
> จะเห็น container mysql และ pma อยู่ใน network นี้

##### 4.7 การรัน WordPress Container
# Mac/Linux
```bash
docker run --name wordpress -p 7777:80 \
--network wordpress \
-e WORDPRESS_DB_HOST=mysql \
-e WORDPRESS_DB_USER=wordpress\
-e WORDPRESS_DB_PASSWORD=wordpress \
-e WORDPRESS_DB_NAME=wordpress_db \
-d wordpress
```
# Windows (PowerShell)
```bash
docker run --name wordpress -p 7777:80 `
--network wordpress `
-e WORDPRESS_DB_HOST=mysql `
-e WORDPRESS_DB_USER=wordpress `
-e WORDPRESS_DB_PASSWORD=wordpress `
-e WORDPRESS_DB_NAME=wordpress_db `
-d wordpress
```
# Windows (Command Prompt)
```bash
docker run --name wordpress -p 7777:80 ^
--network wordpress ^
-e WORDPRESS_DB_HOST=mysql ^
-e WORDPRESS_DB_USER=wordpress ^
-e WORDPRESS_DB_PASSWORD=wordpress ^
-e WORDPRESS_DB_NAME=wordpress_db ^
-d wordpress
```

#### 4.8 การเข้าถึง WordPress และ phpMyAdmin
- WordPress: [http://localhost:7777](http://localhost:7777)
- phpMyAdmin: [http://localhost:8888](http://localhost:8888)
> server: mysql
> Username: root  
> Password: 1234

---

## สิ่งที่เรียนรู้ใน Day 2

✅ พื้นฐานการใช้งาน Docker
✅ การใช้งานคำสั่งพื้นฐานของ Docker
✅ การสร้างและจัดการ Container

---

## หมายเหตุ

- ใช้ Docker Desktop เวอร์ชั่นล่าสุด
- ใช้ Git เวอร์ชั่นล่าสุด
- ใช้ Visual Studio Code เวอร์ชั่นล่าสุด
- ใช้ Git Bash (สำหรับ Windows) หรือ Terminal (สำหรับ Mac/Linux)

## สรุป
ขอให้สนุกกับการเรียนรู้ Basic Docker for Developer 2026 ครับ!
