# 🐍 Flask + Tailwind CSS Docker Guide

> การสร้างและรันแอปพลิเคชัน Python Flask + Tailwind CSS ด้วย Docker  
> พร้อม PostgreSQL, Nginx และ Production-ready Setup

---

## 🏗️ โครงสร้างโปรเจ็กต์

```
flask-tailwind-docker/
├── Dockerfile                  # Dockerfile สำหรับ Development
├── Dockerfile.prod             # Dockerfile สำหรับ Production (multi-stage)
├── docker-compose.yml          # สภาพแวดล้อม Development
├── docker-compose.prod.yml     # สภาพแวดล้อม Production
├── .dockerignore               # ไฟล์ Docker ignore
├── .env                        # ตัวแปรสภาพแวดล้อม
├── pyproject.toml              # Python dependencies (UV)
├── app/
│   ├── __init__.py             # Flask App Factory
│   ├── routes.py               # Routes หลัก
│   ├── models.py               # Database Models
│   ├── static/
│   │   ├── src/
│   │   │   └── input.css       # Tailwind CSS Input
│   │   └── dist/
│   │       └── output.css      # Tailwind CSS Output (compiled)
│   └── templates/
│       ├── base.html           # Base Template
│       └── index.html          # หน้าแรก
├── tailwind.config.js          # Tailwind Configuration
├── package.json                # Node.js dependencies (Tailwind)
└── docker/
    ├── nginx/
    │   └── default.conf        # Nginx Config
    └── postgres/
        └── init.sql            # PostgreSQL Init Script
```

---

## ขั้นตอนที่ 1: สร้างโฟลเดอร์โปรเจ็กต์

```bash
mkdir flask-tailwind-docker && cd flask-tailwind-docker

# สร้างโครงสร้างโฟลเดอร์
mkdir -p app/static/src
mkdir -p app/static/dist
mkdir -p app/templates
mkdir -p docker/nginx
mkdir -p docker/postgres
```

---

## ขั้นตอนที่ 2: สร้างโปรเจ็กต์ Flask ด้วย UV กับ Docker

```bash
# สำหรับ macOS / Linux
docker run --rm -it \
  -v ${PWD}:/app \
  -p 5050:5000 \
  ghcr.io/astral-sh/uv:python3.13-alpine \
  sh -c "
    cd /app &&
    uv init --python 3.13 &&
    uv add flask flask-sqlalchemy psycopg2-binary python-decouple flask-migrate &&
    uv add --dev gunicorn &&
    echo 'Flask project initialized!'
  "

# สำหรับ Windows (PowerShell)
docker run --rm -it `
  -v ${PWD}:/app `
  -p 5050:5000 `
  ghcr.io/astral-sh/uv:python3.13-alpine `
  sh -c "
    cd /app &&
    uv init --python 3.13 &&
    uv add flask flask-sqlalchemy psycopg2-binary python-decouple flask-migrate &&
    uv add --dev gunicorn &&
    echo 'Flask project initialized!'
  "

# สำหรับ Windows (CMD)
docker run --rm -it ^
  -v %cd%:/app ^
  -p 5050:5000 ^
  ghcr.io/astral-sh/uv:python3.13-alpine ^
  sh -c "cd /app && uv init --python 3.13 && uv add flask flask-sqlalchemy psycopg2-binary python-decouple flask-migrate && uv add --dev gunicorn && echo done"
```

ผลลัพธ์ใน `pyproject.toml`:

```toml
[project]
name = "flask-tailwind-docker"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "flask>=3.1.0",
    "flask-sqlalchemy>=3.1.0",
    "flask-migrate>=4.0.0",
    "psycopg2-binary>=2.9.0",
    "python-decouple>=3.8",
]

[dependency-groups]
dev = [
    "gunicorn>=23.0.0",
]
```

---

## ขั้นตอนที่ 3: ติดตั้ง Tailwind CSS

```bash
# สร้างไฟล์ package.json
cat > package.json << 'EOF'
{
  "name": "flask-tailwind-docker",
  "version": "1.0.0",
  "scripts": {
    "dev": "tailwindcss -i ./app/static/src/input.css -o ./app/static/dist/output.css --watch",
    "build": "tailwindcss -i ./app/static/src/input.css -o ./app/static/dist/output.css --minify"
  },
  "devDependencies": {
    "tailwindcss": "^3.4.0"
  }
}
EOF

# สร้าง tailwind.config.js
cat > tailwind.config.js << 'EOF'
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./app/templates/**/*.html",
    "./app/static/src/**/*.js",
  ],
  theme: {
    extend: {
      fontFamily: {
        sans: ['Sarabun', 'sans-serif'],
        display: ['Kanit', 'sans-serif'],
      },
      colors: {
        primary: {
          50:  '#eff6ff',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          900: '#1e3a8a',
        }
      }
    },
  },
  plugins: [],
}
EOF

# สร้าง input.css
cat > app/static/src/input.css << 'EOF'
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  body {
    @apply font-sans text-gray-800;
  }
}

@layer components {
  .btn-primary {
    @apply bg-primary-600 hover:bg-primary-700 text-white font-medium
           py-2 px-4 rounded-lg transition-colors duration-200;
  }
  .card {
    @apply bg-white rounded-xl shadow-md p-6 border border-gray-100;
  }
  .input-field {
    @apply w-full border border-gray-300 rounded-lg px-4 py-2
           focus:outline-none focus:ring-2 focus:ring-primary-500
           focus:border-transparent transition-all duration-200;
  }
}
EOF
```

---

## ขั้นตอนที่ 4: สร้างไฟล์ .env

สร้างไฟล์ `.env` ในโฟลเดอร์หลักของโปรเจ็กต์:

```env
# Flask Settings
SECRET_KEY=your-secret-key-change-this-in-production
DEBUG=True
FLASK_ENV=development

# PostgreSQL Database
DB_ENGINE=postgresql
DB_NAME=flaskdb
DB_USER=postgres
DB_PASSWORD=123456
DB_HOST=pgdb
DB_PORT=5432

# Full Database URL
DATABASE_URL=postgresql://postgres:123456@pgdb:5432/flaskdb
```

---

## ขั้นตอนที่ 5: สร้าง Flask Application

### `app/__init__.py` — App Factory

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from decouple import config

db = SQLAlchemy()
migrate = Migrate()


def create_app():
    app = Flask(__name__)

    # Configuration
    app.config['SECRET_KEY'] = config('SECRET_KEY', default='dev-secret-key')
    app.config['DEBUG'] = config('DEBUG', default=True, cast=bool)
    app.config['SQLALCHEMY_DATABASE_URI'] = config(
        'DATABASE_URL',
        default='sqlite:///dev.db'
    )
    app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

    # Extensions
    db.init_app(app)
    migrate.init_app(app, db)

    # Blueprints
    from app.routes import main
    app.register_blueprint(main)

    return app
```

### `app/models.py` — Database Models

```python
from app import db
from datetime import datetime


class User(db.Model):
    __tablename__ = 'users'

    id         = db.Column(db.Integer, primary_key=True)
    username   = db.Column(db.String(80), unique=True, nullable=False)
    email      = db.Column(db.String(120), unique=True, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    def __repr__(self):
        return f'<User {self.username}>'


class Post(db.Model):
    __tablename__ = 'posts'

    id         = db.Column(db.Integer, primary_key=True)
    title      = db.Column(db.String(200), nullable=False)
    content    = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    user_id    = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    author     = db.relationship('User', backref=db.backref('posts', lazy=True))

    def __repr__(self):
        return f'<Post {self.title}>'
```

### `app/routes.py` — Routes

```python
from flask import Blueprint, render_template, jsonify
from app.models import User, Post
from app import db

main = Blueprint('main', __name__)


@main.route('/')
def index():
    posts = Post.query.order_by(Post.created_at.desc()).limit(10).all()
    return render_template('index.html', posts=posts)


@main.route('/health')
def health():
    return jsonify({'status': 'ok', 'message': 'Flask is running!'})


@main.route('/users')
def users():
    all_users = User.query.all()
    return render_template('users.html', users=all_users)
```

### `app/templates/base.html` — Base Template

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{% block title %}Flask + Tailwind{% endblock %}</title>

  <!-- Google Fonts: Kanit + Sarabun (รองรับภาษาไทย) -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=Kanit:wght@400;600;700&family=Sarabun:wght@300;400;500&display=swap" rel="stylesheet">

  <!-- Tailwind CSS (compiled) -->
  <link rel="stylesheet" href="{{ url_for('static', filename='dist/output.css') }}">
</head>
<body class="bg-gray-50 min-h-screen">

  <!-- Navbar -->
  <nav class="bg-white shadow-sm border-b border-gray-200">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
      <div class="flex justify-between items-center h-16">
        <div class="flex items-center gap-2">
          <span class="text-2xl">🐍</span>
          <span class="font-display font-bold text-xl text-primary-700">
            Flask + Tailwind
          </span>
        </div>
        <div class="flex items-center gap-6">
          <a href="/" class="text-gray-600 hover:text-primary-600 font-medium transition-colors">
            หน้าแรก
          </a>
          <a href="/users" class="text-gray-600 hover:text-primary-600 font-medium transition-colors">
            Users
          </a>
          <a href="/health" class="btn-primary text-sm">
            Health Check
          </a>
        </div>
      </div>
    </div>
  </nav>

  <!-- Main Content -->
  <main class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% if messages %}
        {% for category, message in messages %}
          <div class="mb-4 p-4 rounded-lg
            {% if category == 'error' %}bg-red-50 text-red-700 border border-red-200
            {% else %}bg-green-50 text-green-700 border border-green-200{% endif %}">
            {{ message }}
          </div>
        {% endfor %}
      {% endif %}
    {% endwith %}

    {% block content %}{% endblock %}
  </main>

  <!-- Footer -->
  <footer class="mt-16 border-t border-gray-200 py-8 text-center text-gray-400 text-sm">
    Flask + Tailwind CSS + Docker &copy; {{ 2024 }}
  </footer>

</body>
</html>
```

### `app/templates/index.html` — หน้าแรก

```html
{% extends "base.html" %}

{% block title %}หน้าแรก — Flask + Tailwind{% endblock %}

{% block content %}

<!-- Hero Section -->
<div class="text-center py-16">
  <h1 class="font-display text-5xl font-bold text-gray-900 mb-4">
    Flask + <span class="text-primary-600">Tailwind CSS</span>
  </h1>
  <p class="text-xl text-gray-500 mb-8 max-w-2xl mx-auto">
    แอปพลิเคชัน Python Flask พร้อม Tailwind CSS และ PostgreSQL บน Docker
  </p>
  <div class="flex justify-center gap-4">
    <a href="/users" class="btn-primary text-base px-6 py-3">
      จัดการ Users
    </a>
    <a href="/health" class="border border-gray-300 text-gray-700 hover:bg-gray-50
       font-medium py-3 px-6 rounded-lg transition-colors duration-200">
      Health Check
    </a>
  </div>
</div>

<!-- Stats Cards -->
<div class="grid grid-cols-1 md:grid-cols-3 gap-6 mb-12">
  <div class="card text-center">
    <div class="text-4xl font-display font-bold text-primary-600 mb-2">🐍</div>
    <div class="font-semibold text-gray-900">Python Flask</div>
    <div class="text-sm text-gray-500 mt-1">Web Framework</div>
  </div>
  <div class="card text-center">
    <div class="text-4xl font-display font-bold text-cyan-500 mb-2">🎨</div>
    <div class="font-semibold text-gray-900">Tailwind CSS</div>
    <div class="text-sm text-gray-500 mt-1">Utility-first CSS</div>
  </div>
  <div class="card text-center">
    <div class="text-4xl font-display font-bold text-blue-500 mb-2">🐳</div>
    <div class="font-semibold text-gray-900">Docker</div>
    <div class="text-sm text-gray-500 mt-1">Containerized</div>
  </div>
</div>

<!-- Posts -->
{% if posts %}
<div>
  <h2 class="font-display text-2xl font-bold text-gray-900 mb-6">โพสต์ล่าสุด</h2>
  <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
    {% for post in posts %}
    <div class="card hover:shadow-lg transition-shadow duration-200">
      <h3 class="font-semibold text-gray-900 text-lg mb-2">{{ post.title }}</h3>
      <p class="text-gray-500 text-sm mb-3 line-clamp-2">{{ post.content }}</p>
      <div class="flex justify-between items-center text-xs text-gray-400">
        <span>โดย {{ post.author.username }}</span>
        <span>{{ post.created_at.strftime('%d/%m/%Y') }}</span>
      </div>
    </div>
    {% endfor %}
  </div>
</div>
{% else %}
<div class="card text-center py-16 text-gray-400">
  <div class="text-5xl mb-4">📝</div>
  <p class="text-lg">ยังไม่มีโพสต์ เริ่มสร้างได้เลย!</p>
</div>
{% endif %}

{% endblock %}
```

---

## ขั้นตอนที่ 6: สร้าง Dockerfile (Development)

```dockerfile
# ─── Base Image ───────────────────────────────────────
FROM python:3.13-slim-bookworm

# ─── System Dependencies ──────────────────────────────
RUN apt-get update && apt-get install -y \
    curl \
    ca-certificates \
    libpq-dev \
    gcc \
    nodejs \
    npm \
    && rm -rf /var/lib/apt/lists/*

# ─── Install UV ───────────────────────────────────────
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy

# ─── Create Non-root User ─────────────────────────────
RUN groupadd --system app && \
    useradd --system --gid app --create-home app

WORKDIR /app
RUN chown -R app:app /app
USER app

# ─── Python Dependencies ──────────────────────────────
COPY --chown=app:app pyproject.toml uv.lock* ./
RUN uv sync --locked --no-install-project

# ─── Node.js Dependencies (Tailwind) ─────────────────
COPY --chown=app:app package.json ./
RUN npm install

# ─── Copy Application ─────────────────────────────────
COPY --chown=app:app . .

# ─── Final Python Sync ────────────────────────────────
RUN uv sync --locked

# ─── Build Tailwind CSS ───────────────────────────────
RUN npm run build

ENV PATH="/app/.venv/bin:$PATH"
ENV FLASK_APP=run.py

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

CMD ["python", "-m", "flask", "run", "--host=0.0.0.0", "--port=5000"]
```

---

## ขั้นตอนที่ 7: สร้าง Dockerfile.prod (Production — Multi-stage)

```dockerfile
# ══════════════════════════════════════════════════════
# Stage 1: Build Tailwind CSS
# ══════════════════════════════════════════════════════
FROM node:20-alpine AS tailwind-builder

WORKDIR /build
COPY package.json tailwind.config.js ./
RUN npm install

COPY app/static/src ./app/static/src
COPY app/templates   ./app/templates
RUN npm run build

# ══════════════════════════════════════════════════════
# Stage 2: Build Python App
# ══════════════════════════════════════════════════════
FROM python:3.13-slim-bookworm AS python-builder

RUN apt-get update && apt-get install -y \
    libpq-dev gcc \
    && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy

WORKDIR /app
COPY pyproject.toml uv.lock* ./
RUN uv sync --locked --no-install-project --no-dev

# ══════════════════════════════════════════════════════
# Stage 3: Production Image (เล็กที่สุด)
# ══════════════════════════════════════════════════════
FROM python:3.13-slim-bookworm AS production

RUN apt-get update && apt-get install -y \
    libpq5 curl \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd --system app && \
    useradd --system --gid app --create-home app

WORKDIR /app
RUN chown -R app:app /app
USER app

# Copy Python venv จาก Stage 2
COPY --from=python-builder --chown=app:app /app/.venv /app/.venv

# Copy Application
COPY --chown=app:app app/ ./app/
COPY --chown=app:app run.py ./

# Copy Tailwind Output จาก Stage 1
COPY --from=tailwind-builder --chown=app:app \
    /build/app/static/dist/output.css \
    ./app/static/dist/output.css

ENV PATH="/app/.venv/bin:$PATH"
ENV FLASK_ENV=production
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Production: ใช้ Gunicorn แทน Flask dev server
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", \
     "--timeout", "120", "--access-logfile", "-", "run:app"]
```

---

## ขั้นตอนที่ 8: สร้าง run.py

```python
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

---

## ขั้นตอนที่ 9: สร้าง docker-compose.yml (Development)

```yaml
networks:
  flask_network:
    name: flask_network
    driver: bridge

services:

  # ─── PostgreSQL ───────────────────────────────────────
  pgdb:
    image: postgres:16-alpine
    container_name: flask_pgdb
    restart: unless-stopped
    networks:
      - flask_network
    environment:
      POSTGRES_DB:       flaskdb
      POSTGRES_USER:     postgres
      POSTGRES_PASSWORD: 123456
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8"
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5532:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d flaskdb"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ─── Flask Application ────────────────────────────────
  flaskweb:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: flask_app
    restart: unless-stopped
    networks:
      - flask_network
    ports:
      - "5050:5000"
    environment:
      SECRET_KEY:    "your-secret-key-here"
      DEBUG:         "True"
      FLASK_ENV:     development
      DATABASE_URL:  postgresql://postgres:123456@pgdb:5432/flaskdb
    volumes:
      # Dev: Hot reload — mount source code
      - ./app:/app/app
      - ./run.py:/app/run.py
      # Mount Tailwind output แยก (ไม่ให้ overwrite)
      - /app/app/static/dist
      - /app/.venv
    depends_on:
      pgdb:
        condition: service_healthy
    command: >
      sh -c "
        echo 'Running migrations...' &&
        flask db upgrade &&
        echo 'Starting Flask dev server...' &&
        python -m flask run --host=0.0.0.0 --port=5000 --reload
      "

  # ─── Tailwind CSS Watcher (Dev Only) ─────────────────
  tailwind:
    image: node:20-alpine
    container_name: flask_tailwind
    working_dir: /app
    volumes:
      - .:/app
    command: sh -c "npm install && npm run dev"
    # Hot-reload Tailwind เมื่อแก้ไข HTML/CSS
```

---

## ขั้นตอนที่ 10: สร้าง docker-compose.prod.yml (Production)

```yaml
networks:
  flask_prod_network:
    name: flask_prod_network
    driver: bridge

services:

  pgdb:
    image: postgres:16-alpine
    container_name: flask_prod_pgdb
    restart: always
    networks:
      - flask_prod_network
    environment:
      POSTGRES_DB:       ${DB_NAME}
      POSTGRES_USER:     ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_prod_data:/var/lib/postgresql/data
    # ไม่ expose Port ออกนอกใน Production
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 30s
      timeout: 10s
      retries: 5

  flaskweb:
    build:
      context: .
      dockerfile: Dockerfile.prod    # ← Multi-stage Production Build
    container_name: flask_prod_app
    restart: always
    networks:
      - flask_prod_network
    environment:
      SECRET_KEY:   ${SECRET_KEY}
      DEBUG:        "False"
      FLASK_ENV:    production
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@pgdb:5432/${DB_NAME}
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 1G
    depends_on:
      pgdb:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:1.27-alpine
    container_name: flask_prod_nginx
    restart: always
    networks:
      - flask_prod_network
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro
    security_opt:
      - no-new-privileges:true
    depends_on:
      - flaskweb

volumes:
  postgres_prod_data:
    name: flask_postgres_prod_data
```

---

## ขั้นตอนที่ 11: สร้าง Nginx Config

สร้างไฟล์ `docker/nginx/default.conf`:

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    server_tokens off;

    # Static Files — เสิร์ฟโดย Nginx โดยตรง (เร็วกว่า)
    location /static/ {
        alias /app/app/static/;
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    # Proxy ไปที่ Flask/Gunicorn
    location / {
        proxy_pass         http://flaskweb:5000;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_read_timeout    60s;
        client_max_body_size  20M;
    }
}
```

---

## ขั้นตอนที่ 12: สร้างไฟล์ .dockerignore

```
# Python
__pycache__/
*.py[cod]
*.so
.Python
*.egg-info/
dist/
build/

# Virtual Environments
.venv/
venv/
env/

# Flask
*.log
instance/
.webassets-cache

# Node.js
node_modules/
npm-debug.log

# Tailwind Source (ไม่ต้องใส่ใน Production Image)
app/static/src/

# Database
postgres_data/
*.sql
*.db

# Environment
.env
.env.*

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Git
.git/
.gitignore

# Docker
Dockerfile*
docker-compose*.yml

# Docs
README.md
*.md
```

---

## ขั้นตอนที่ 13: สร้างไฟล์ init.sql

สร้างไฟล์ `docker/postgres/init.sql`:

```sql
-- PostgreSQL Initialization Script for Flask App

-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Timezone
SET timezone = 'Asia/Bangkok';

-- Index สำหรับ Full-text Search (ภาษาไทย)
-- จะสร้างหลัง Flask migration รัน
```

---

## ขั้นตอนที่ 14: รันแอปพลิเคชัน (Development)

```bash
# Build และรัน
docker compose up -d --build

# ตรวจสอบสถานะ
docker compose ps

# ดู Log
docker compose logs -f

# ดู Log เฉพาะ Flask
docker compose logs -f flaskweb

# ดู Log Tailwind Watcher
docker compose logs -f tailwind
```

---

## ขั้นตอนที่ 15: ทดสอบแอปพลิเคชัน

เปิดเว็บเบราว์เซอร์และไปที่ `http://localhost:5050`

```bash
# ทดสอบ Health Check
curl http://localhost:5050/health
# ควรได้: {"message": "Flask is running!", "status": "ok"}

# ทดสอบ API
curl http://localhost:5050/users
```

---

## ขั้นตอนที่ 16: รัน Migrations และสร้าง Superuser

```bash
# Initialize Flask-Migrate (ครั้งแรก)
docker exec -it flask_app sh -c "flask db init"

# สร้าง Migration
docker exec -it flask_app sh -c "flask db migrate -m 'initial migration'"

# รัน Migration
docker exec -it flask_app sh -c "flask db upgrade"

# สร้างข้อมูลทดสอบ (seed data)
docker exec -it flask_app sh -c "
python -c \"
from app import create_app, db
from app.models import User, Post
app = create_app()
with app.app_context():
    user = User(username='admin', email='admin@example.com')
    db.session.add(user)
    db.session.commit()
    print('Admin user created!')
\"
"
```

---

## ขั้นตอนที่ 17: Deploy Production

```bash
# Build Production Image (Multi-stage)
docker compose -f docker-compose.prod.yml up -d --build

# ตรวจสอบ
docker compose -f docker-compose.prod.yml ps

# ดู Log
docker compose -f docker-compose.prod.yml logs -f
```

---

## คำสั่งที่ใช้บ่อย

```bash
# ─── Development ─────────────────────────────────────
docker compose up -d --build          # Build & Start
docker compose down                   # Stop
docker compose restart flaskweb       # Restart Flask only
docker compose logs -f flaskweb       # Logs Flask

# ─── Flask Management ─────────────────────────────────
docker exec -it flask_app flask db migrate -m "add column"
docker exec -it flask_app flask db upgrade
docker exec -it flask_app flask shell  # Python shell

# ─── Tailwind CSS ─────────────────────────────────────
# Dev: Auto-watch (รันอัตโนมัติใน tailwind container)
# Production: Build ครั้งเดียวตอน docker build

# ─── Database ─────────────────────────────────────────
docker exec -it flask_pgdb psql -U postgres -d flaskdb

# ─── Monitor ─────────────────────────────────────────
docker stats --no-stream
docker system df
```

---

## Architecture สรุป

```
Browser
  ↓ HTTP/HTTPS
Nginx (Port 80/443)
  ├── /static/ → เสิร์ฟไฟล์โดยตรง (CSS, JS, Images)
  └── /        → Proxy → Gunicorn → Flask App
                              ↓
                         PostgreSQL
```

```
Development Stack          Production Stack
──────────────────         ──────────────────
Flask dev server           Gunicorn (4 workers)
Tailwind --watch           Tailwind --minify (build once)
Hot reload                 Multi-stage Docker Image
Port 5050                  Nginx :80/:443 + SSL
```

---

> 📝 **หมายเหตุ:**
> - แทนที่ `yourdomain.com` ด้วยโดเมนจริงของคุณ
> - เปลี่ยน `SECRET_KEY` และ `DB_PASSWORD` ก่อน Deploy จริงทุกครั้ง
> - ไฟล์ `.env` ต้องไม่ commit เข้า Git เด็ดขาด — เพิ่มใน `.gitignore`
