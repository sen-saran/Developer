## Basic Docker for Developer 2026: ปูพื้นฐานสู่การใช้งานจริง - Day 2

### 📋 สารบัญ
1. [พื้นฐานการใช้งาน Docker](#พื้นฐานการใช้งาน-docker)
   - [1. การติดตั้ง Docker Desktop](#1-การติดตั้ง-docker-desktop)
   - [2. การใช้งานคำสั่งพื้นฐานของ Docker](#2-การใช้งานคำสั่งพื้นฐานของ-docker)
   - [3. การสร้างและจัดการ Container](#3-การสร้างและจัดการ-container)

## พื้นฐานการใช้งาน Docker
- การติดตั้ง Docker Desktop
- การใช้งานคำสั่งพื้นฐานของ Docker
- การสร้างและจัดการ Container
- การสร้าง Dockerfile และ Docker Image
- การใช้งาน Docker Compose

#### 1. การติดตั้ง Docker Desktop
- ดาวน์โหลดและติดตั้ง Docker Desktop จาก [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
- ตรวจสอบการติดตั้งโดยรันคำสั่ง:
```bash
docker --version
docker compose version
docker info
```

#### 2. การใช้งานคำสั่งพื้นฐานของ Docker
##### 2.1 รัน hello-world container
```bash
docker run hello-world
```
> คำสั่งนี้จะดาวน์โหลดและรัน container ที่แสดงข้อความต้อนรับจาก Docker

##### 2.2 ดูรายการ container ที่กำลังรันอยู่
```bash
docker ps
docker ps -a  # ดู container ทั้งหมดรวมถึงที่หยุดแล้ว
```

##### 2.3 หยุด container
```bash
docker stop <container_id>
```

##### 2.4 ลบ container
```bash
docker rm <container_id>
```

##### 2.5 ดูรายการ image ที่มีอยู่
```bash
docker images
docker image ls
```
##### 2.6 ลบ image
```bash
docker rmi <image_id>
```

#### 3. การสร้างและจัดการ Container
##### 3.1 รัน container จาก image
```bash
docker run -d -p 8880:80 --name mynginx nginx
```
> คำสั่งนี้จะรัน Nginx container ในโหมด detached และแมปพอร์ต 80 ของ container ไปยังพอร์ต 8880 ของโฮสต์

##### 3.2 เข้าสู่ shell ของ container
```bash
docker exec -it mynginx /bin/bash
# หรือ
docker exec -it mynginx /bin/sh
```

##### 3.3 ดู log ของ container
```bash
docker logs mynginx
```

#### 4. WORKSHOP WordPress + MySQL + phpMyAdmin
##### 4.1 เตรียมไฟล์ images ที่ต้องการใช้งาน
- WordPress: `docker pull wordpress`
- MySQL: `docker pull mysql:5.7`
- phpMyAdmin: `docker pull phpmyadmin/phpmyadmin`

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
docker network connect wordpress mysql
docker network connect wordpress pma
```

##### 4.6 เช็คว่าใครอยู่ใน network ชื่อ wordpress บ้าง ?
```bash
docker network inspect wordpress
```
> จะเห็น container mysql และ pma อยู่ใน network นี้

##### 4.7 การรัน WordPress Container
```bash
# Mac/Linux
docker run --name wordpress -p 7777:80 \
--network wordpress \
-e WORDPRESS_DB_HOST=mysql \
-e WORDPRESS_DB_USER=wordpress\
-e WORDPRESS_DB_PASSWORD=wordpress \
-e WORDPRESS_DB_NAME=wordpress_db \
-d wordpress

# Windows (PowerShell)
docker run --name wordpress -p 7777:80 `
--network wordpress `
-e WORDPRESS_DB_HOST=mysql `
-e WORDPRESS_DB_USER=wordpress `
-e WORDPRESS_DB_PASSWORD=wordpress `
-e WORDPRESS_DB_NAME=wordpress_db `
-d wordpress

# Windows (Command Prompt)
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