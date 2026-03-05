## Basic Docker for Developer 2026: ปูพื้นฐานสู่การใช้งานจริง - Day 3

### 📋 สารบัญ
1. [พื้นฐาน Dockerfile](#พื้นฐาน-dockerfile)
2. [พื้นฐาน Docker Compose](#พื้นฐาน-docker-compose)

## พื้นฐาน Dockerfile
> Dockerfile คือไฟล์ที่ใช้กำหนดขั้นตอนการสร้าง Docker Image โดยระบุคำสั่งต่างๆ ที่จะถูกดำเนินการเมื่อสร้าง Image

1. การสร้าง Dockerfile และ Docker Image
2. การรัน Docker Container จาก Image ที่สร้างขึ้น

```bash
docker run -d -p 3300:3000 --name mydockerapp docker-node-app
```
## พื้นฐาน Docker Compose
> Docker Compose คือเครื่องมือที่ช่วยในการจัดการหลายๆ Container พร้อมกัน โดยใช้ไฟล์การตั้งค่าเดียว (docker-compose.yml)
1. การสร้างไฟล์ docker-compose.yml
2. การรันและจัดการ Container ด้วย Docker Compose

```bash
docker-compose up -d
```

#### 1. การสร้าง Dockerfile และ Docker Image

##### 1.1 สร้างโฟลเดอร์โปรเจ็กต์
```bash
mkdir basic-docker/docker-node-app
cd basic-docker/docker-node-app
```

##### 1.2 สร้างไฟล์ package.json
```bash
touch package.json
```

##### 1.3 กำหนดสคริปต์ใน package.json
```json
{
  "name": "docker-node-app",
  "version": "1.0.0",
  "description": "A simple Node.js app for Docker",
  "main": "index.js",
  "scripts": {
    "dev": "nodemon index.js",
    "start": "node index.js"
  },
  "author": "Your Name",
  "license": "ISC",
  "dependencies": {
    "express": "^4.18.2",
    "nodemon": "^2.0.22"
  }
}
```
> nodemon เป็นเครื่องมือที่ช่วยในการพัฒนา Node.js โดยจะทำการรีสตาร์ทเซิร์ฟเวอร์อัตโนมัติเมื่อมีการเปลี่ยนแปลงในโค้ด

##### 1.4 สร้างไฟล์ index.js
```javascript
const express = require('express')
const app = express()

// ทำ url ให้สามารถเข้าถึงได้
app.get('/', (req, res) => {
  res.send('Hello World!')
})

// run the server
app.listen(3000, () => {
  console.log('Server is running on http://localhost:3000')
})
```

##### 1.5 สร้างไฟล์ Dockerfile

> ไฟล์ Dockerfile เป็นไฟล์ข้อความธรรมดาที่ใช้กำหนดขั้นตอนการสร้าง Docker Image โดยระบุฐานของ image, การติดตั้ง dependencies, การคัดลอกไฟล์, การตั้งค่าพอร์ต และคำสั่งที่ต้องรันเมื่อ container เริ่มทำงาน

```Dockerfile
# โหลด image ของ node จาก docker hub
FROM node:alpine

# กำหนด directory ที่จะใช้เก็บไฟล์ของโปรเจค
WORKDIR /app

# คัดลอกไฟล์ package.json และ package-lock.json ไปยัง directory ที่กำหนดไว้
COPY package*.json ./

# ติดตั้ง package ที่ระบุในไฟล์ package.json
RUN npm install

# ติดตั้ง nodemon เพื่อใช้ในการรันโปรเจค
RUN npm install -g nodemon

# คัดลอกไฟล์ทั้งหมดไปยัง directory ที่กำหนดไว้
COPY . .

# ระบุ port ที่จะใช้
EXPOSE 3000

# รันคำสั่ง npm run dev เมื่อ container ถูกสร้างขึ้น
CMD ["npm", "run", "dev"]
```

##### 1.6 สร้าง Docker Image
```bash
docker build -t docker-node-app .
```

##### 1.7 รัน Docker Container
```bash
docker run -d -p 3300:3000 --name mydockerapp docker-node-app
```

#### 6. การใช้งาน Docker Compose
##### 6.1 สร้างไฟล์ docker-compose.yml
```yaml
networks:
  nodejs_network:
    name: nodejs_network
    driver: bridge

services:

  # NodeJS App
  nodejs:
    build:
      context: .
      dockerfile: Dockerfile
      tags:
        - "mynodeapp:1.0"
    container_name: mynodeapp
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - CHOKIDAR_USEPOLLING=true # สำหรับ Windows เพื่อให้ nodemon ทำงานได้
    networks:
      - nodejs_network
    restart: always

  # MongoDB
  mongodb:
    image: mongo
    container_name: mongodb
    ports:
      - "28017:27017"
    networks:
      - nodejs_network
    restart: always
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
    name: mongo_data
    driver: local
```

##### 6.2 รัน Docker Compose
```bash
# เช็คความถูกต้องของไฟล์ docker-compose.yml
docker compose config

# รัน Docker Compose
docker compose up -d

# หรือถ้ามีการเปลี่ยนแปลงไฟล์ Dockerfile หรือ docker-compose.yml
docker compose up -d --build
```

##### 6.3 ตรวจสอบสถานะของ Container
```bash
docker compose ps
```

##### 6.4 หยุดและลบ Container
```bash
docker compose down

# หรือถ้าต้องการลบข้อมูลทั้งหมดรวมถึง volume
docker compose down -v

# หรือถ้าต้องการลบ image ด้วย
docker compose down --rmi all -v
```

### ตัวยอย่าง docker-compose.yml สำหรับ WordPress + MySQL + phpMyAdmin

```yaml
networks:
  wp_network:
    name: wp_network
    driver: bridge   

services:
  db:
    image: mysql:5.7 # for windows
    # image: mysql:8.0 # for macos/linux
    container_name: mysql_db
    restart: always
    networks:
      - wp_network
    volumes:
      - mysql:/var/lib/mysql
    ports:
      - 3360:3306
    environment:
      - MYSQL_ROOT_PASSWORD=1234
      - MYSQL_DATABASE=wordpress_db
      - MYSQL_USER=wordpress
      - MYSQL_PASSWORD=wordpress
  wordpress:
    depends_on:
      - db
    image: wordpress
    container_name: wordpress
    restart: always
    networks:
      - wp_network
    volumes:
      - ./wordpress:/var/www/html
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
    image: phpmyadmin
    container_name: phpmyadmin
    restart: always
    networks:
      - wp_network
    environment:
      - PMA_ARBITRARY=1
    ports:
      - 7770:80

# Named volume for MySQL data persistence
volumes:
  mysql:
    name: mysql
    driver: local
```
---
## สิ่งที่เรียนรู้ใน Day 3

✅ พื้นฐานการใช้งาน Dockerfile
✅ การใช้งาน Docker Compose

---
## หมายเหตุ

- ใช้ Docker Desktop เวอร์ชั่นล่าสุด
- ใช้ Git เวอร์ชั่นล่าสุด
- ใช้ Visual Studio Code เวอร์ชั่นล่าสุด
- ใช้ Git Bash (สำหรับ Windows) หรือ Terminal (สำหรับ Mac/Linux)

## สรุป
ขอให้สนุกกับการเรียนรู้ Basic Docker for Developer 2026 ครับ!