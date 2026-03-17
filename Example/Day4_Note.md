## Basic Docker for Developer 2026: ปูพื้นฐานสู่การใช้งานจริง - Day 4

### 📋 สารบัญ
1. [Docker MERN Stack Application](#docker-mern-stack-application)
2. [Docker Python Django Application](#docker-python-django-application)
3. [Docker Hub และการจัดการ Image](#docker-hub-และการจัดการ-image)
4. [การใช้งาน Portainer สำหรับจัดการ Docker](#การใช้งาน-portainer-สำหรับจัดการ-docker)

## 1. Docker MERN Stack Application
> การสร้างและรันแอปพลิเคชัน MERN Stack (MongoDB, Express, React, Node.js) ด้วย Docker

### ขั้นตอนที่ 1: สร้างโฟลเดอร์โปรเจกต์
```bash
mkdir mern-docker
cd mern-docker
```

### ขั้นตอนที่ 2: สร้างโฟลเดอร์สำหรับเซิร์ฟเวอร์
```bash
# สร้างโปรเจ็กต์ Express ด้วย Docker

# for MacOS / Linux
docker run --rm -it \
  -v ${PWD}/server:/app \
  node:22-alpine \
  sh -c "cd /app && npm init -y && npm install express mongoose cors dotenv nodemon"

# for Windows (PowerShell)
docker run --rm -it `
  -v ${PWD}/server:/app `
  node:22-alpine `
  sh -c "cd /app && npm init -y && npm install express mongoose cors dotenv nodemon"

# for Windows (CMD)
docker run --rm -it ^
  -v %cd%/server:/app ^
  node:22-alpine ^
  sh -c "cd /app && npm init -y && npm install express mongoose cors dotenv nodemon"

```

### ขั้นตอนที่ 3: สร้างไฟล์ server.js สำหรับเซิร์ฟเวอร์
```javascript
// server/server.js

const express = require('express')
const mongoose = require('mongoose')
const cors = require('cors')
require('dotenv').config()

const app = express()

app.use(cors())
app.use(express.json())

// Connect to MongoDB
mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('MongoDB Connected'))
  .catch(err => console.log(err));

// Basic route
app.get('/', (req, res) => {
  res.json({ message: 'Welcome to MERN application.' })
});

app.get('/about', (req, res) => {
  res.json({ message: 'About Page' })
});

const PORT = process.env.PORT || 5000
app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`)
})
```

### ขั้นตอนที่ 4: สร้างไฟล์ .env สำหรับเซิร์ฟเวอร์
```env
# server/.env
MONGODB_URI=mongodb://mongodb:27017/mern-docker
PORT=5000
```

### ขั้นตอนที่ 5: แก้ไขไฟล์ package.json เพื่อเพิ่มสคริปต์ start
```json
{
  "name": "app",
  "version": "1.0.0",
  "description": "",
  "main": "server.js",
  "scripts": {
    "dev": "nodemon --watch './**/*.js' server.js",
    "start": "node server.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^17.2.3",
    "express": "^5.1.0",
    "mongoose": "^8.19.3",
    "nodemon": "^3.1.10"
  }
}
```

### ขั้นตอนที่ 6: สร้างไฟล์ Dockerfile
```Dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 5000

CMD ["npm", "run", "dev"]
```

### ขั้นตอนที่ 7: สร้างโฟลเดอร์สำหรับไคลเอนต์

```bash
# สร้างโปรเจ็กต์ React ด้วย Docker

# for MacOS / Linux
docker run --rm -it \
  -v ${PWD}/client:/app \
  node:22-alpine \
  sh -c "cd /app && \
    npm create vite@latest . -- --template react && \
    npm install && \
    npm install react@19.0.0 react-dom@19.0.0"

# for Windows (PowerShell)
docker run --rm -it `
  -v ${PWD}/client:/app `
  node:22-alpine `
  sh -c "cd /app && `
    npm create vite@latest . -- --template react && `
    npm install && `
    npm install react@19.0.0 react-dom@19.0.0"

# for Windows (CMD)
docker run --rm -it ^
  -v %cd%/client:/app ^
  node:22-alpine ^
  sh -c "cd /app && ^
         npm create vite@latest . -- --template react && ^
         npm install && ^
         npm install react@19.0.0 react-dom@19.0.0"
```

### ขั้นตอนที่ 8: แก้ไขไฟล์ vite.config.js ของไคลเอนต์
แก้ไขไฟล์ `client/vite.config.js` ให้เป็นดังนี้:
```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

// https://vite.dev/config/
export default defineConfig({
  plugins: [react()],
  server: {
    host: true,
    port: 3000, // dev
    // port: 80, // prod
    watch: {
      usePolling: true,
    },
    hmr: {
      clientPort: 3000 // dev
      // clientPort: 80 // prod
    }
  }
})
```

### ขั้นตอนที่ 9: แก้ไขไฟล์ App.jsx ของไคลเอนต์
แก้ไขไฟล์ `client/src/App.jsx` ให้เป็นดังนี้:
```javascript
import { useState, useEffect } from 'react'
import './App.css'

// Example using React 19 features with useEffect
function App() {
  const [count, setCount] = useState(0)
  const [data, setData] = useState(null)

  // Example using React 19 features with useEffect
  useEffect(() => {
    // Automatic batching demo
    setTimeout(() => {
      setCount(c => c + 1);
      setData({ message: 'Hello React 19!' });
    }, 1000);
  }, []);

  return (
    <>
      <h1>Hello Vite + React 19</h1>
      <div className="card">
        <button onClick={() => setCount((count) => count + 1)}>
          count is {count}
        </button>
        {data && <p>{data.message}</p>}
        <p>
          Edit <code>src/App.jsx</code> and save to test HMR
        </p>
      </div>
    </>
  )
}

export default App
```

### ขั้นตอนที่ 10: สร้างไฟล์ Dockerfile สำหรับไคลเอนต์
สร้างไฟล์ `client/Dockerfile` ดังนี้:
```Dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

### ขั้นตอนที่ 11: สร้างไฟล์ docker-compose.yml
สร้างไฟล์ `docker-compose.yml` ในโฟลเดอร์หลักของโปรเจ็กต์:
```yaml
networks:
  mern-network:
    name: mern-network
    driver: bridge

services:

  # mongo service
  mongodb:
    image: mongo:latest
    container_name: mongodb
    ports:
      - "28017:27017"
    volumes:
      - ./mongodb_data:/data/db
    networks:
      - mern-network
    restart: always

  # server service
  server:
    build:
      context: ./server
      dockerfile: Dockerfile
    image: mern-server:1.0
    container_name: server
    ports:
      - "5100:5000"
    volumes:
      - ./server:/app
      - /app/node_modules # bookmark คือการให้ docker ไม่เอา node_modules มาใช้งาน
    environment:
      - MONGODB_URI=${MONGODB_URI}
      - NODE_ENV=development
    command: npm run dev
    networks:
      - mern-network
    restart: always
    depends_on:
      - mongodb

  # client service
  client:
    build:
      context: ./client
      dockerfile: Dockerfile
    image: mern-client:1.0
    container_name: client
    ports:
      - "3100:3000"
    volumes:
      - ./client:/app
      - /app/node_modules # bookmark คือการให้ docker ไม่เอา node_modules มาใช้งาน
    environment:
      - VITE_API_URL=http://localhost:5000
    command: npm run dev
    networks:
      - mern-network
    restart: always
    depends_on:
      - server
```

### ขั้นตอนที่ 12: รันแอปพลิเคชัน
```bash
docker compose up -d
```

### ขั้นตอนที่ 13: ทดสอบแอปพลิเคชัน
- เปิดเว็บเบราว์เซอร์และไปที่ `http://localhost:3100` เพื่อดูแอปพลิเคชัน React
- ไปที่ `http://localhost:5100` เพื่อดูแอปพลิเคชัน Express

### ขั้นตอนที่ 14: หยุดและลบคอนเทนเนอร์
```bash
docker compose down --rmi all
```

### ขั้นตอนที่ 15: สร้าง dockerfile สำหรับ production (สำหรับไคลเอนต์)
สร้างไฟล์ `client/Dockerfile.prod` ดังนี้:
```Dockerfile
FROM node:22-alpine AS build

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
# daemon off is used to run the nginx in the foreground
# so that the container does not exit immediately after starting
# and the container will be running as long as the nginx is running
```

สร้างไฟล์ `client/nginx.conf` ดังนี้:
```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

### ขั้นตอนที่ 16: สร้าง dockerfile สำหรับ production (สำหรับเซิร์ฟเวอร์)
สร้างไฟล์ `server/Dockerfile.prod` ดังนี้:
```Dockerfile
FROM node:22-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

EXPOSE 5000
CMD ["npm", "start"]
```

### ขั้นตอนที่ 17: สร้างไฟล์ docker-compose.prod.yml สำหรับ production
สร้างไฟล์ `docker-compose.prod.yml` ในโฟลเดอร์หลักของโปรเจ็กต์:
```yaml
networks:
  mern-network:
    name: mern-network
    driver: bridge

services:

  # mongo service
  mongodb:
    image: mongo:latest
    container_name: mongodb
    ports:
      - "29017:27017"
    volumes:
      - mongodb_data:/data/db # ใช้ volume named mongodb_data
    networks:
      - mern-network
    restart: always

  # server service
  server:
    build:
      context: ./server
      dockerfile: Dockerfile.prod
    image: mern-server:1.0
    container_name: server
    ports:
      - "5200:5000"
    environment:
      - MONGODB_URI=${MONGODB_URI}
      - NODE_ENV=production
    depends_on:
      - mongodb
    networks:
      - mern-network

  # client service
  client:
    build:
      context: ./client
      dockerfile: Dockerfile.prod
    image: mern-client:1.0
    container_name: client
    ports:
      - "8088:80"
    environment:
      - VITE_API_URL=http://localhost:5200
    depends_on:
      - server
    networks:
      - mern-network

volumes:
  mongodb_data:
    driver: local
```

### ขั้นตอนที่ 18: รันแอปพลิเคชันในโหมด production
```bash
# อย่าลืมรันลบ container เดิมก่อน
docker compose down --rmi all

# คำสั่งรันตรวจสอบความถูกต้องของไฟล์ docker-compose.prod.yml
docker compose -f docker-compose.prod.yml config

# คำสั่งรันแอปพลิเคชันในโหมด production
docker compose -f docker-compose.prod.yml up -d --build
```

## 2. Docker Python Django Application และ UV
> การสร้างและรันแอปพลิเคชัน Python Django ด้วย Docker 

## 🏗️ โครงสร้างโปรเจ็กต์

```
pythondjango-docker/
├── Dockerfile                  # Dockerfile สำหรับ Development
├── Dockerfile.prod             # Dockerfile สำหรับ Production (multi-stage)
├── docker-compose.yml          # สภาพแวดล้อม Development
├── docker-compose.prod.yml     # สภาพแวดล้อม Production
├── .dockerignore               # ไฟล์ Docker ignore
├── .env                        # Template สำหรับสภาพแวดล้อม Production
└── docker/
    ├── nginx/
    │   ├── nginx.conf          # การตั้งค่าหลัก Nginx
    │   └── default.conf        # การตั้งค่าไซต์ Nginx
    └── postgres/
        └── init.sql            # การเริ่มต้น PostgreSQL
```

### ขั้นตอนที่ 1: สร้างโฟลเดอร์โปรเจกต์
```bash
mkdir pythondjango-docker && cd pythondjango-docker
```

### ขั้นตอนที่ 2: สร้างโปรเจ็กต์ Django ด้วย UV กับ Docker
```bash
# for MacOS / Linux
docker run --rm -it \
  -v ${PWD}:/app \
  -p 8110:8000 \
  ghcr.io/astral-sh/uv:python3.13-alpine \
  sh -c "cd /app && uv init --python 3.13 && uv add django && uv sync && uv run django-admin startproject bookstore_project . && uv run manage.py startapp books_app && uv add psycopg python-decouple gunicorn && uv run manage.py runserver 0.0.0.0:8000"

# for Windows (PowerShell)
docker run --rm -it `
  -v ${PWD}:/app `
  -p 8110:8000 `
  ghcr.io/astral-sh/uv:python3.13-alpine `
  sh -c "cd /app && uv init --python 3.13 && uv add django && uv sync && uv run django-admin startproject bookstore_project . && uv run manage.py startapp books_app && uv add psycopg python-decouple gunicorn && uv run manage.py runserver 0.0.0.0:8000"

# for Windows (CMD)
docker run --rm -it ^
  -v %cd%:/app ^
  -p 8110:8000 ^
  ghcr.io/astral-sh/uv:python3.13-alpine ^
  sh -c "cd /app && uv init --python 3.13 && uv add django && uv sync && uv run django-admin startproject bookstore_project . && uv run manage.py startapp books_app && uv add psycopg python-decouple gunicorn && uv run manage.py runserver 0.0.0.0:8000"
```
ผลลัพธ์ใน `pyproject.toml`:
```toml
[project]
name = "django-uv"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "django>=5.2.7",
    "psycopg>=3.2.12",
    "python-decouple>=3.8",
]

[dependency-groups]
dev = [
    "django-debug-toolbar>=6.0.0",
]
prod = [
    "gunicorn>=23.0.0",
]
```

เปิดเบราว์เซอร์และไปที่ `http://127.0.0.1:8110` เพื่อตรวจสอบว่าโปรเจ็กต์ Django ถูกสร้างขึ้นเรียบร้อยแล้ว

### ขั้นตอนที่ 3: สร้างไฟล์ .env
สร้างไฟล์ `.env` ในโฟลเดอร์หลักของโปรเจ็กต์

```env
# Django Settings
SECRET_KEY=your_secret_key_here
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1

# PostgreSQL Database Configuration
DB_ENGINE=django.db.backends.postgresql
DB_NAME=bookstoredb
DB_USER=postgres
DB_PASSWORD=123456
DB_HOST=pgdb
DB_PORT=5432
```

### ขั้นตอนที่ 4: สร้างไฟล์ Dockerfile
สร้างไฟล์ `Dockerfile` ในโฟลเดอร์หลักของโปรเจ็กต์:
```Dockerfile
# Base image with Python 3.13
FROM python:3.13-slim-bookworm

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    ca-certificates \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Install uv - pin to specific version for reproducibility
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Set environment variables for uv
ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy

# Create app user
RUN groupadd --system app && \
    useradd --system --gid app --create-home app

# Set working directory
WORKDIR /app

# Change ownership of the app directory
RUN chown -R app:app /app

# Switch to non-root user
USER app

# Copy dependency files first (for better caching)
COPY --chown=app:app pyproject.toml uv.lock ./

# Install dependencies
RUN uv sync --locked --no-install-project

# Copy the rest of the application
COPY --chown=app:app . .

# Final sync to install the project itself
RUN uv sync --locked

# Expose port 8000
EXPOSE 8000

# Set the PATH to include the virtual environment
ENV PATH="/app/.venv/bin:$PATH"

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/ || exit 1

# Default command - can be overridden in docker-compose
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

### ขั้นตอนที่ 5: สร้างไฟล์ docker-compose.yml
สร้างไฟล์ `docker-compose.yml` ในโฟลเดอร์หลักของโปรเจ็กต์:
```yaml
networks:
  bookstore_network:
    name: bookstore_network
    driver: bridge

services:
  # PostgreSQL Database
  pgdb:
    image: postgres:16-alpine
    container_name: pgdb
    restart: unless-stopped
    environment:
      POSTGRES_DB: bookstoredb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 123456
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8"
    volumes:
      # Use named volume for database data (recommended for safety and performance)
      - ./postgres_data:/var/lib/postgresql/data
      # Bind mount for initialization scripts (easy to edit)
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      # Optional: bind mount for easy database dumps during development
      # - ./docker/postgres/dumps:/dumps
    ports:
      - "5532:5432"
    networks:
      - bookstore_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d bookstoredb"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Django Web Application
  djangoweb:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: djangoweb
    restart: unless-stopped
    ports:
      - "8110:8000"
    environment:
      # Django Settings
      SECRET_KEY: "your_secret_key_here"
      DEBUG: "True"
      ALLOWED_HOSTS: localhost,127.0.0.1,0.0.0.0
      STATIC_ROOT: /app/staticfiles
      
      # Database Configuration
      DB_ENGINE: django.db.backends.postgresql
      DB_NAME: bookstoredb
      DB_USER: postgres
      DB_PASSWORD: 123456
      DB_HOST: pgdb
      DB_PORT: 5432
    volumes:
      # Development volume mounts for live code changes
      - ./:/app
      # Preserve .venv directory in container
      - /app/.venv
    depends_on:
      pgdb:
        condition: service_healthy
    networks:
      - bookstore_network
    command: >
      sh -c "
        echo 'Waiting for database...' &&
        sleep 5 &&
        python manage.py migrate &&
        python manage.py runserver 0.0.0.0:8000
      "
```

### ขั้นตอนที่ 6: สร้างไฟล์ .dockerignore
สร้างไฟล์ `.dockerignore` ในโฟลเดอร์หลักของโปรเจ็กต์:

```
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
*.egg-info/
dist/
build/

# Virtual Environment
.venv/
venv/
env/
ENV/

# Django
*.log
db.sqlite3
db.sqlite3-journal
media/
staticfiles/

# Environment
.env
.env.local

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Git
.git/
.gitignore

# Docker
Dockerfile*
docker-compose*.yml

# Database
postgres_data/
*.sql

# Documentation
README.md
*.md
```

### ขั้นตอนที่ 7: สร้างไฟล์ init.sql สำหรับการเริ่มต้นฐานข้อมูล
สร้างไฟล์ `docker/postgres/init.sql` ดังนี้:
```sql
-- PostgreSQL initialization script
-- This script runs when the database container starts for the first time

-- Create the database if it doesn't exist (usually handled by POSTGRES_DB env var)
-- CREATE DATABASE bookstoredjuv;

-- You can add additional initialization queries here
-- For example, creating additional users, extensions, etc.

-- Enable some useful extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Set timezone
SET timezone = 'UTC';
```

### ขั้นตอนที่ 8: แก้ไขการตั้งค่า Django สำหรับฐานข้อมูล
เปิดไฟล์ `bookstore_project/settings.py` และแก้ไขการตั้งค่าฐานข้อมูลเป็นดังนี้:
```python
import os
from decouple import config

# SECURITY WARNING: keep the secret key used in production secret!
# ดึงค่าจาก .env
SECRET_KEY = config('SECRET_KEY', default='your_secret_key_here')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = config('DEBUG', default=True, cast=bool)

ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='localhost,127.0.0.1').split(',')


INSTALLED_APPS = [
    ...
    'books_app',
    ...
]
...


DATABASES = {
    'default': {
        'ENGINE': config('DB_ENGINE', default='django.db.backends.sqlite3'),
        'NAME': config('DB_NAME', default=str(BASE_DIR / 'db.sqlite3')),
        'USER': config('DB_USER', default=''),
        'PASSWORD': config('DB_PASSWORD', default=''),
        'HOST': config('DB_HOST', default=''),
        'PORT': config('DB_PORT', default=''),
    }
}
```

### ขั้นตอนที่ 9: รันแอปพลิเคชัน
```bash
docker compose up -d --build
```

### ขั้นตอนที่ 10: ทดสอบแอปพลิเคชัน
- เปิดเว็บเบราว์เซอร์และไปที่ `http://localhost:8110


### ขั้นตอนที่ 11: run migrations และสร้าง superuser
```bash
docker exec -it djangoweb sh -c "python manage.py migrate"
docker exec -it djangoweb sh -c "python manage.py createsuperuser"
```

- เปิดเว็บเบราว์เซอร์และไปที่ `http://localhost:8110` เพื่อตรวจสอบว่าโปรเจ็กต์ Django รันได้ถูกต้อง
- ไปที่ `http://localhost:8110/admin` เพื่อล็อกอินเข้าสู่แผงผู้ดูแลระบบ Django

## 3. Docker Hub และการจัดการ Image
> การใช้งาน Docker Hub สำหรับการจัดการ Docker Images 

### ขั้นตอนที่ 1: สร้างบัญชี Docker Hub
- ไปที่ [Docker Hub](https://hub.docker.com/) และสมัครบัญชีผู้ใช้ใหม่หากยังไม่มี

### ขั้นตอนที่ 2: เข้าสู่ระบบ Docker Hub ผ่าน CLI
```bash
docker login
```
- ป้อนชื่อผู้ใช้และรหัสผ่านของ Docker Hub

### ขั้นตอนที่ 3: สร้าง Docker Image
```bash
docker build -t your_dockerhub_username/your_image_name:tag .
```
### ขั้นตอนที่ 4: push Docker Image ไปยัง Docker Hub
```bash
docker push your_dockerhub_username/your_image_name:tag
```
- ตัวอย่าง:
```bash
docker push johndoe/my-django-app:latest
```

## 4. การใช้งาน Portainer สำหรับจัดการ Docker
> การติดตั้งและใช้งาน Portainer เพื่อจัดการ Docker ผ่านเว็บอินเทอร์เฟซ
### ขั้นตอนที่ 1: สร้าง docker-compose.yml สำหรับ Portainer
```bash
networks:
  pt_network:
    name: pt_network
    driver: bridge

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer_container
    ports:
      - "8989:8000"
      - "9553:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - pt_data:/data
    networks:
      - pt_network
    restart: always

volumes:
  pt_data:
```

### ขั้นตอนที่ 2: รัน Portainer
```bash
docker compose up -d
```

### ขั้นตอนที่ 3: เข้าสู่ระบบ Portainer
- เปิดเว็บเบราว์เซอร์และไปที่ `https://localhost:9553`
- ตั้งค่าบัญชีผู้ดูแลระบบครั้งแรก
- เชื่อมต่อกับ Docker environment ของคุณ
- คุณสามารถจัดการคอนเทนเนอร์, อิมเมจ, วอลุ่ม และเครือข่ายผ่านเว็บอินเทอร์เฟซของ Portainer ได้อย่างง่ายดาย 
