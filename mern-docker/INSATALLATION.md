## Docker MERN Stack Application

## 1. Infrastacture
> การสร้างและรันแอปพลิเคชัน MERN Stack (MongoDB, Express, React, Node.js) ด้วย Docker
- Network mern-docker
- Container (Node,Express) Name:Server  Port 5100:5000
- Container (Nginx,React)  Name:Client  Port 3100:3000
- Container (MongoDB)      Name:MongoDB Port 28017:27017

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
