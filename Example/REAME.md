# Docker for Developers 🚀

คู่มือสรุปคำสั่ง Docker ที่จำเป็นสำหรับการพัฒนาซอฟต์แวร์ ตั้งแต่การจัดการ Container ไปจนถึงการทำ Orchestration เบื้องต้นด้วย Compose

#### 1. Infrastructure Docker Container

โครงสร้างสถาปัตยกรรมของ Docker ที่ช่วยให้การรันแอปพลิเคชันมีประสิทธิภาพสูงและกินทรัพยากรน้อยกว่า VM ทั่วไป

| Layer | คำอธิบาย |
| :--- | :--- |
| **Containers** | แอปพลิเคชันที่ถูกแพ็กไว้ (เช่น WordPress, MySQL) |
| **Docker Engine** | ตัวกลางจัดการ Container และทรัพยากรระบบ |
| **Network & IPAM** | ส่วนจัดการหมายเลข IP และการเชื่อมต่อสื่อสาร |
| **Infrastructure** | ทรัพยากรพื้นฐาน (Host OS, Hardware, Cloud) |

<==== vm ====><==== vm ====><==== vm ====>

<====== Network ======>  <====== Network ======>

<=============  DockerEngine =============>

<===== Network =====>  <===== Network =====>

<=========== NetworkInfrastructure ===========>

---
**สถาปัตยกรรมเครือข่ายภายใน:**
1. **Network Infrastructure**: ระบบพื้นฐานด้าน Network
2. **IPAM Driver**: จัดสรร IP Address ให้แต่ละ Container อัตโนมัติ
3. **Docker Engine**: ควบคุมทุกอย่างผ่านพื้นฐาน Network เดียวกัน
4. **Isolated Environments**: แบ่งโซนการทำงานชัดเจนเสมือนมี Server หลายเครื่อง

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
### คำสั่งจัดการ volume
```bash
docker volume ls
docker volume create db_data
docker container inspect wordpress
# MySQL
docker run --name mysql -e C:/Users/admin/Desktop/คู่มือ/Docker for Developer 2026/db_data:/var/lib/mysql-e MYSQL_ROOT_PASSWORD=1234 -e MYSQL_DATABASE=wordpress_db -e MYSQL_USER=wordpress -e MYSQL_PASSWORD=wordpress -d mysql:5.7
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

#### 5. Docker MERN Stack Application
##### 5.1 Dockerfile คืออะไร?
**Dockerfile** คือสคริปต์ที่รวบรวมคำสั่ง (Instructions) เพื่อใช้สร้าง **Docker Image** เปรียบเสมือนพิมพ์เขียวที่บอกว่า Container ต้องติดตั้งอะไรบ้างถึงจะทำงานได้
---
> การสร้างและรันแอปพลิเคชัน MERN Stack (MongoDB, Express, React, Node.js) ด้วย Docker
---
##### 5.2 ตารางคำสั่ง Dockerfile (Core Instructions)

| คำสั่ง | คำอธิบาย | ตัวอย่างการใช้งาน |
| :--- | :--- | :--- |
| **FROM** | ระบุ Image ตั้งต้นที่ใช้ (Base Image) | `FROM node:18-alpine` |
| **WORKDIR** | กำหนดตำแหน่ง Folder เริ่มต้นภายใน Container | `WORKDIR /app` |
| **COPY** | คัดลอกไฟล์จากเครื่อง Host เข้าไปใน Container | `COPY . .` |
| **ADD** | คัดลอกไฟล์ (รองรับไฟล์บีบอัดและการดึงจาก URL) | `ADD data.tar.gz /app` |
| **RUN** | สั่งทำงานคำสั่งในขั้นตอน Build (เช่นการลง Library) | `RUN npm install` |
| **ENV** | กำหนดค่า Environment Variable | `ENV NODE_ENV=production` |
| **EXPOSE** | กำหนด Port ที่ต้องการเปิดใช้งาน | `EXPOSE 3000` |
| **CMD** | คำสั่งหลักที่จะรันเมื่อ Container เริ่มทำงาน | `CMD ["npm", "start"]` |
| **ENTRYPOINT** | กำหนดคำสั่งหลักที่ตายตัวสำหรับ Container | `ENTRYPOINT ["node", "app.js"]` |
---
> **ข้อควรรู้:** `RUN` ทำงานตอน Build (สร้าง Image), ส่วน `CMD` ทำงานตอน Run (สร้าง Container)
---

### ตัวยอย่าง docker-compose.yml สำหรับ WordPress + MySQL + phpMyAdmin
#### 1. การสร้าง Dockerfile และ Docker Image
##### 1.1 สร้างโฟลเดอร์โปรเจ็กต์
```bash
# สร้างโฟลเดอร์โปรเจ็กต์
mkdir docker-wordpress-app
cd docker-wordpress-app

# สร้างไฟล์ docker-compose.yml (ใช้คำสั่ง touch สำหรับ Mac/Linux หรือสร้างไฟล์ใหม่ใน VS Code)
touch docker-compose.yml
```

```yaml
networks:
  wp_network:
    name: wp_network
    driver: bridge   

services:
  db:
    image: mysql:5.7 # เสถียรสำหรับหลายระบบ
    container_name: mysql_db
    restart: always
    networks:
      - wp_network
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - 3306:3306
    environment:
      - MYSQL_ROOT_PASSWORD=1234
      - MYSQL_DATABASE=wordpress_db
      - MYSQL_USER=wordpress
      - MYSQL_PASSWORD=wordpress

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    container_name: wordpress_app
    restart: always
    networks:
      - wp_network
    volumes:
      - ./wordpress_files:/var/www/html
    ports:
      - 8800:80
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=wordpress
      - WORDPRESS_DB_PASSWORD=wordpress
      - WORDPRESS_DB_NAME=wordpress_db

  phpmyadmin:
    depends_on:
      - db
    image: phpmyadmin:latest
    container_name: phpmyadmin_tool
    restart: always
    networks:
      - wp_network
    environment:
      - PMA_ARBITRARY=1
      - PMA_HOST=db
    ports:
      - 7770:80

volumes:
  mysql_data:
    name: mysql_data
```
---
```bash
# ตรวจสอบความถูกต้องของไฟล์ YAML (Syntax Check)
docker compose config

# สั่งรันระบบในพื้นหลัง (Background)
docker compose up -d

# สั่งรันพร้อมบังคับ Build ใหม่ (กรณีมีการแก้ไข Dockerfile หรือ Config)
docker compose up -d --build

# ดูสถานะของ Container ใน Compose
docker compose ps

# สั่งหยุดและลบ Container/Network ทั้งหมดในโปรเจ็กต์
docker compose down
```






