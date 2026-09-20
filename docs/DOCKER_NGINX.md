# NGINX — Reverse Proxy Guide

Complete reference for the NGINX reverse proxy setup in the Viñedosya e-commerce platform.

---

## 📖 Table of Contents

1. [What is NGINX?](#what-is-nginx)
2. [Why a Reverse Proxy?](#why-a-reverse-proxy)
3. [Architecture](#architecture)
4. [Project Structure](#project-structure)
5. [Configuration Files](#configuration-files)
   - [docker-compose.yml](#docker-composeyml)
   - [nginx.conf.template](#nginxconftemplate)
   - [.env](#env)
   - [.env.example](#envexample)
6. [Core Concepts](#core-concepts)
   - [Upstreams](#upstreams)
   - [Location Blocks](#location-blocks)
   - [Proxy Pass](#proxy-pass)
   - [Rewrite Rules](#rewrite-rules)
   - [WebSocket Support](#websocket-support)
   - [Security Headers](#security-headers)
   - [Gzip Compression](#gzip-compression)
7. [Setup](#setup)
8. [Environment Variables with envsubst](#environment-variables-with-envsubst)
9. [How We Got Here (Evolution)](#how-we-got-here-evolution)
   - [Version 1: Hardcoded Configuration](#version-1-hardcoded-configuration)
   - [Version 2: Environment Variables with envsubst](#version-2-environment-variables-with-envsubst)
10. [Verification](#verification)
11. [Troubleshooting](#troubleshooting)
12. [Best Practices](#best-practices)

---

## 📖 What is NGINX?

**NGINX** (pronounced "engine-x") is a high-performance web server and reverse proxy. It was created in 2004 by Igor Sysoev to solve the C10K problem (handling 10,000+ concurrent connections).

### Key Features

| Feature | Description |
|---------|-------------|
| **Web Server** | Serves static files (HTML, CSS, JS, images) |
| **Reverse Proxy** | Forwards requests to backend servers |
| **Load Balancer** | Distributes traffic across multiple servers |
| **SSL/TLS Termination** | Handles HTTPS certificates |
| **Caching** | Caches responses to reduce backend load |
| **Compression** | Gzip/Brotli compression for responses |

### Why NGINX?

- ✅ **Fast:** Handles thousands of concurrent connections with low memory
- ✅ **Reliable:** Used by 40%+ of the top 10,000 websites
- ✅ **Lightweight:** Small footprint (~50MB Docker image)
- ✅ **Flexible:** Highly configurable
- ✅ **Free:** Open-source (BSD license)

---

## 🎯 Why a Reverse Proxy?

### Without Reverse Proxy

```text
┌─────────────────────────────────────────────────────────────┐
│                    USER BROWSER                             │
│                                                             │
│  ┌─────────────────────────┐  ┌─────────────────────────┐   │
│  │  http://localhost:3000  │  |  http://localhost:3001  │   │
│  │  (Nuxt)                 │  │  (NestJS)               │   │
│  └─────────────────────────┘  └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Problems:**
- ❌ User must know two URLs and ports
- ❌ CORS issues (different origins)
- ❌ No SSL/HTTPS
- ❌ Both ports exposed to the internet
- ❌ No compression, caching, or rate limiting

### With Reverse Proxy

```text
┌─────────────────────────────────────────────────────────────┐
│                    USER BROWSER                             │
│                                                             │
│              ┌──────────────────────────┐                   │
│              │  http://localhost/       │                   │
│              │  (Single entry point)    │                   │
│              └────────────┬─────────────┘                   │
└───────────────────────────┼─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                       NGINX                                 │
│                    (Reverse Proxy)                          │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │  /api/* → nest-api   │  │  /* → nuxt-app       │         │
│  └──────────────────────┘  └──────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
    ┌──────────────────┐      ┌──────────────────┐
    │   Nuxt (SSR)     │      │   NestJS API     │
    │   :3000          │      │   :3001          │
    └──────────────────┘      └──────────────────┘
```

**Benefits:**
- ✅ Single entry point (`http://localhost/`)
- ✅ No CORS issues (same origin)
- ✅ Ready for SSL/HTTPS
- ✅ Only NGINX port exposed
- ✅ Compression, caching, security headers

---

## 🏗️ Architecture

### Request Routing

| URL | Destination | Purpose |
|-----|-------------|---------|
| `http://localhost/` | `nuxt-app:3000` | Nuxt SSR (frontend) |
| `http://localhost/api/*` | `nest-api:3001/api/v1/*` | NestJS REST API |
| `http://localhost/socket.io/*` | `nest-api:3001` | Socket.io WebSockets |
| `http://localhost/health` | NGINX itself | Health check |

### Network

All containers communicate via the **`app-network`** Docker network:

```text
┌─────────────────────────────────────────────────────────────┐
│                    app-network                              │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │  nginx   │  │nuxt-app  │  │nest-api  │  │postgres  │     │
│  │  :80     │  │ :3000    │  │ :3001    │  │ :5432    │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
│                                                             │
│  ┌──────────┐  ┌──────────┐                                 │
│  │  redis   │  │ mailhog  │                                 │
│  │  :6379   │  │ :1025    │                                 │
│  └──────────┘  └──────────┘                                 │
└─────────────────────────────────────────────────────────────┘
```

**Important:** Containers on the same network can reach each other by **container name** (not `localhost`).

- ✅ `http://nest-api:3001` (correct)
- ❌ `http://localhost:3001` (incorrect inside a container)

---

## 📁 Project Structure

```text
nginx/
├── .env                    # Environment variables (NOT committed)
├── .env.example            # Environment variables template (committed)
├── .gitignore              # Git ignore rules
├── docker-compose.yml      # NGINX container orchestration
├── nginx.conf.template     # NGINX configuration template
├── docs/                   # Documentation
│   └── DOCKER_NGINX.md     # This file
└── logs/                   # NGINX logs (NOT committed)
```

---

## 📄 Configuration Files

### `docker-compose.yml`

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf.template:/etc/nginx/nginx.conf.template:ro
      - ./logs:/var/log/nginx
    environment:
      - NUXT_UPSTREAM=${NUXT_UPSTREAM:-nuxt-app:3000}
      - NEST_UPSTREAM=${NEST_UPSTREAM:-nest-api:3001}
      - API_VERSION=${API_VERSION:-v1}
    command: >
      sh -c "envsubst '$${NUXT_UPSTREAM} $${NEST_UPSTREAM} $${API_VERSION}' 
      < /etc/nginx/nginx.conf.template 
      > /etc/nginx/nginx.conf 
      && nginx -g 'daemon off;'"
    env_file:
      - .env
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    external: true
```

**Explanation:**

| Setting | Purpose |
|---------|---------|
| `image: nginx:alpine` | Lightweight NGINX image (~50MB) |
| `container_name: nginx` | Fixed name for DNS resolution |
| `ports: 80:80, 443:443` | Expose HTTP and HTTPS |
| `volumes` | Mount config template and logs |
| `environment` | Variables for `envsubst` |
| `command` | Substitute variables, then start NGINX |
| `networks` | Connect to `app-network` |

---

### `nginx.conf.template`

**Full file:**

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # ============================================
    # Logging
    # ============================================
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    # ============================================
    # Performance
    # ============================================
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 20M;

    # ============================================
    # Gzip Compression
    # ============================================
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/rss+xml
        application/atom+xml
        image/svg+xml;

    # ============================================
    # Upstreams
    # ============================================
    upstream nuxt_app {
        server ${NUXT_UPSTREAM};
    }

    upstream nest_api {
        server ${NEST_UPSTREAM};
    }

    # ============================================
    # Server
    # ============================================
    server {
        listen 80;
        server_name localhost;

        # ============================================
        # Security Headers
        # ============================================
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;

        # ============================================
        # API (NestJS) - /api/* → /api/${API_VERSION}/*
        # ============================================
        location /api/ {
            rewrite ^/api/(.*)$ /api/${API_VERSION}/$1 break;
            
            proxy_pass http://nest_api;
            proxy_http_version 1.1;
            
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
            
            proxy_buffering off;
            proxy_request_buffering off;
        }

        # ============================================
        # WebSocket (Socket.io) - /socket.io/*
        # ============================================
        location /socket.io/ {
            proxy_pass http://nest_api;
            proxy_http_version 1.1;
            
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            proxy_connect_timeout 7d;
            proxy_send_timeout 7d;
            proxy_read_timeout 7d;
        }

        # ============================================
        # Nuxt (SSR) - Todo lo demás
        # ============================================
        location / {
            proxy_pass http://nuxt_app;
            proxy_http_version 1.1;
            
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # ============================================
        # Health check
        # ============================================
        location /health {
            access_log off;
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }
    }
}
```

---

### `.env`

```env
# ============================================
# NGINX Configuration
# ============================================
NGINX_PORT=80
NGINX_SSL_PORT=443

# ============================================
# Upstreams (container names)
# ============================================
NUXT_UPSTREAM=nuxt-app:3000
NEST_UPSTREAM=nest-api:3001

# ============================================
# API Version
# ============================================
API_VERSION=v1

# ============================================
# Environment
# ============================================
NODE_ENV=production
```

**⚠️ Nota:** El archivo `.env` **NO debe commitearse** (contiene valores específicos del entorno).

---

### `.env.example`

```env
# ============================================
# NGINX Configuration
# ============================================
NGINX_PORT=80
NGINX_SSL_PORT=443

# ============================================
# Upstreams (container names)
# ============================================
NUXT_UPSTREAM=nuxt-app:3000
NEST_UPSTREAM=nest-api:3001

# ============================================
# API Version
# ============================================
API_VERSION=v1

# ============================================
# Environment
# ============================================
NODE_ENV=production
```

**✅ Nota:** El archivo `.env.example` **SÍ debe commitearse** (sirve de documentación).

---

## 🧠 Core Concepts

### Upstreams

An **upstream** is a group of backend servers that NGINX can forward requests to.

```nginx
upstream nuxt_app {
    server nuxt-app:3000;
}
```

**Benefits:**
- ✅ Load balancing (if multiple servers)
- ✅ Failover (if one server goes down)
- ✅ Named references (use `nuxt_app` instead of `nuxt-app:3000`)

**Example with multiple servers:**
```nginx
upstream nuxt_app {
    server nuxt-app-1:3000;
    server nuxt-app-2:3000;
    server nuxt-app-3:3000;
}
```

---

### Location Blocks

**Location blocks** define how NGINX handles different URL paths.

```nginx
location /api/ {
    # Handle /api/* requests
}

location /socket.io/ {
    # Handle /socket.io/* requests
}

location / {
    # Handle everything else
}
```

**Matching rules (in order of priority):**
1. **Exact match:** `location = /path`
2. **Prefix match (longest first):** `location /api/`
3. **Regex match:** `location ~ ^/api/`
4. **Default:** `location /`

---

### Proxy Pass

**`proxy_pass`** forwards requests to a backend server.

```nginx
proxy_pass http://nest_api;
```

**Important:** The URL in `proxy_pass` affects how the path is forwarded:

| Config | Request | Forwarded to |
|--------|---------|--------------|
| `proxy_pass http://nest_api;` | `/api/auth` | `http://nest-api:3001/api/auth` |
| `proxy_pass http://nest_api/;` | `/api/auth` | `http://nest-api:3001/auth` (strips `/api/`) |
| `proxy_pass http://nest_api/v2/;` | `/api/auth` | `http://nest-api:3001/v2/auth` |

---

### Rewrite Rules

**`rewrite`** changes the URL before forwarding.

```nginx
rewrite ^/api/(.*)$ /api/${API_VERSION}/$1 break;
```

**Breakdown:**

| Part | Meaning |
|------|---------|
| `^/api/(.*)$` | Match URLs starting with `/api/` and capture the rest |
| `/api/${API_VERSION}/$1` | Replace with `/api/v1/` + captured part |
| `break` | Stop processing more rewrite rules |

**Example:**
- Request: `/api/auth/login`
- Captured: `auth/login`
- Rewritten: `/api/v1/auth/login`
- Forwarded: `http://nest-api:3001/api/v1/auth/login`

---

### WebSocket Support

**WebSockets** require special headers to keep the connection alive.

```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

**Why?**
- HTTP is request-response (connection closes after each request)
- WebSockets are persistent (connection stays open)
- Without these headers, NGINX closes the WebSocket connection

**Timeouts for WebSockets:**
```nginx
proxy_connect_timeout 7d;
proxy_send_timeout 7d;
proxy_read_timeout 7d;
```

**Why 7 days?** WebSocket connections can stay open for a long time (e.g., chat, real-time updates).

---

### Security Headers

**Security headers** protect against common web vulnerabilities.

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

| Header | Purpose |
|--------|---------|
| `X-Frame-Options: SAMEORIGIN` | Prevent clickjacking (only same-origin iframes) |
| `X-Content-Type-Options: nosniff` | Prevent MIME-type sniffing |
| `X-XSS-Protection: 1; mode=block` | Enable browser XSS protection |
| `Referrer-Policy: strict-origin-when-cross-origin` | Control referrer information |

**`always`** means the header is added even for error responses.

---

### Gzip Compression

**Gzip** compresses responses to reduce bandwidth.

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css application/json ...;
```

| Setting | Purpose |
|---------|---------|
| `gzip on` | Enable compression |
| `gzip_vary on` | Add `Vary: Accept-Encoding` header |
| `gzip_proxied any` | Compress proxied responses |
| `gzip_comp_level 6` | Compression level (1-9, higher = better but slower) |
| `gzip_types` | MIME types to compress |

**Result:** Responses are compressed by ~70%, reducing bandwidth.

---

## 🚀 Setup

### Prerequisites

- Docker + Docker Compose
- `app-network` created
- `nuxt-app` running
- `nest-api` running

### Steps

```bash
# 1. Navigate to NGINX folder
cd ~/devenv/nginx

# 2. Ensure app-network exists
docker network create app-network 2>/dev/null || true

# 3. Ensure Nuxt and NestJS are running
docker ps | grep -E "nuxt-app|nest-api"

# 4. Start NGINX
docker compose up -d

# 5. Follow logs
docker compose logs -f
```

### Verification

```bash
# 1. NGINX is running
docker ps | grep nginx

# 2. Frontend responds
curl -I http://localhost/

# 3. API responds
curl http://localhost/api/products/public | head -5

# 4. Health check
curl http://localhost/health

# 5. Login works
curl -X POST http://localhost/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"customer@example.com","password":"Pass123!"}'
```

---

## 🔄 Environment Variables with envsubst

### The Problem

**NGINX does NOT natively support environment variables** in its configuration files.

If you try:
```nginx
server nest-api:${API_PORT};
```

NGINX will **not substitute** `${API_PORT}`. It treats it as a literal string.

### The Solution: `envsubst`

**`envsubst`** is a tool that substitutes environment variables in text files.

**How it works:**

```bash
envsubst '${API_VERSION}' < nginx.conf.template > nginx.conf
```

**Breakdown:**
- `envsubst '${API_VERSION}'` → Substitute only `${API_VERSION}`
- `< nginx.conf.template` → Read from template
- `> nginx.conf` → Write to output file

### Why Limit Variables?

If you run `envsubst` without arguments, it substitutes **all** `${...}` patterns. This breaks NGINX variables like `${remote_addr}`, `${host}`, etc.

**Solution:** Pass only the variables you want to substitute:

```bash
envsubst '${NUXT_UPSTREAM} ${NEST_UPSTREAM} ${API_VERSION}'
```

### Docker Compose Integration

```yaml
command: >
  sh -c "envsubst '$${NUXT_UPSTREAM} $${NEST_UPSTREAM} $${API_VERSION}' 
  < /etc/nginx/nginx.conf.template 
  > /etc/nginx/nginx.conf 
  && nginx -g 'daemon off;'"
```

**Breakdown:**
- `$${VAR}` → Escaped `$` for Docker Compose (becomes `${VAR}` in shell)
- `< /etc/nginx/nginx.conf.template` → Read template
- `> /etc/nginx/nginx.conf` → Write final config
- `&& nginx -g 'daemon off;'` → Start NGINX

### Changing API Version

```bash
# 1. Edit .env
nano .env
# Change: API_VERSION=v2

# 2. Restart container
docker compose down
docker compose up -d

# 3. Verify
docker exec nginx cat /etc/nginx/nginx.conf | grep rewrite
# Should show: rewrite ^/api/(.*)$ /api/v2/$1 break;
```

---

## 📚 How We Got Here (Evolution)

### Version 1: Hardcoded Configuration

**Initial approach:** Hardcoded values in `nginx.conf`.

```nginx
upstream nuxt_app {
    server nuxt-app:3000;  # Hardcoded
}

upstream nest_api {
    server nest-api:3001;  # Hardcoded
}

location /api/ {
    rewrite ^/api/(.*)$ /api/v1/$1 break;  # Hardcoded version
    proxy_pass http://nest_api;
}
```

**Problems:**
- ❌ Changing API version requires editing `nginx.conf`
- ❌ Changing upstreams requires editing `nginx.conf`
- ❌ No flexibility for different environments

**But it worked!** We verified login and API endpoints worked correctly.

---

### Version 2: Environment Variables with envsubst

**Improved approach:** Use `envsubst` to inject variables at container startup.

**Changes:**

1. **Renamed `nginx.conf` → `nginx.conf.template`**

2. **Replaced hardcoded values with variables:**

```nginx
upstream nuxt_app {
    server ${NUXT_UPSTREAM};  # Variable
}

upstream nest_api {
    server ${NEST_UPSTREAM};  # Variable
}

location /api/ {
    rewrite ^/api/(.*)$ /api/${API_VERSION}/$1 break;  # Variable
    proxy_pass http://nest_api;
}
```

3. **Modified `docker-compose.yml` to use `envsubst`:**

```yaml
command: >
  sh -c "envsubst '$${NUXT_UPSTREAM} $${NEST_UPSTREAM} $${API_VERSION}' 
  < /etc/nginx/nginx.conf.template 
  > /etc/nginx/nginx.conf 
  && nginx -g 'daemon off;'"
```

**Benefits:**
- ✅ Change API version via `.env` (no `nginx.conf` edits)
- ✅ Change upstreams via `.env`
- ✅ Different configs for different environments (dev, staging, prod)

**Trade-off:**
- ⚠️ Requires container restart (not just `nginx -s reload`)

---

## ✅ Verification

### Basic Checks

```bash
# 1. NGINX running
docker ps | grep nginx

# 2. NGINX config is valid
docker exec nginx nginx -t

# 3. NGINX config generated correctly
docker exec nginx cat /etc/nginx/nginx.conf | grep -E "upstream|rewrite"
```

### Endpoint Tests

```bash
# Frontend (Nuxt)
curl -I http://localhost/
# Expected: HTTP/1.1 200 OK

# API (NestJS)
curl http://localhost/api/products/public | head -5
# Expected: JSON data

# WebSocket (Socket.io)
curl -I http://localhost/socket.io/
# Expected: HTTP/1.1 400 Bad Request (WebSocket upgrade required)

# Health check
curl http://localhost/health
# Expected: OK

# Login
curl -X POST http://localhost/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"customer@example.com","password":"Pass123!"}' \
  -v
# Expected: 200 OK + JWT token
```

### Header Checks

```bash
# Security headers
curl -I http://localhost/
# Expected:
# X-Frame-Options: SAMEORIGIN
# X-Content-Type-Options: nosniff
# X-XSS-Protection: 1; mode=block
# Referrer-Policy: strict-origin-when-cross-origin

# Gzip compression
curl -I -H "Accept-Encoding: gzip" http://localhost/
# Expected: Content-Encoding: gzip
```

---

## 🔴 Troubleshooting

### 1. `404 Not Found` on `/api/*`

**Symptom:**
```text
POST http://localhost/api/auth/login → 404 Not Found
{"message":"Cannot POST /api/auth/login","error":"Not Found","statusCode":404}
```

**Cause:** NGINX forwards `/api/*` to NestJS, but NestJS expects `/api/v1/*`.

**Solution:** Add `rewrite` rule in `nginx.conf.template`:

```nginx
location /api/ {
    rewrite ^/api/(.*)$ /api/${API_VERSION}/$1 break;
    proxy_pass http://nest_api;
}
```

---

### 2. `502 Bad Gateway`

**Symptom:**
```text
502 Bad Gateway
```

**Cause:** `nuxt-app` or `nest-api` is not running.

**Diagnosis:**
```bash
docker ps | grep -E "nuxt-app|nest-api"
```

**Solution:**
```bash
# Start the missing service
cd ~/devenv/vinedosya/invoice-nest && docker compose up -d
cd ~/devenv/vinedosya/e-commerce-nuxt && docker compose up -d
```

---

### 3. `504 DNS look up failed` (Fortinet/corporate proxy)

**Symptom:**
```text
504 DNS look up failed
URL: http://nuxt-app:3000/auth/login
```

**Cause:** Corporate firewall/proxy (e.g., Fortinet) blocks DNS resolution of Docker container names.

**Solution:** Run tests from host, or use `host.docker.internal` (not applicable for NGINX).

**Note:** This affects E2E tests, not NGINX itself.

---

### 4. NGINX can't resolve `nuxt-app` or `nest-api`

**Symptom:**
```text
nginx: [emerg] host not found in upstream "nuxt-app"
```

**Cause:** NGINX is not on `app-network`.

**Solution:**
```bash
docker network inspect app-network | grep nginx
# If not present:
docker network connect app-network nginx
docker compose restart nginx
```

---

### 5. `envsubst` not substituting variables

**Symptom:** Config still has `${API_VERSION}` literally.

**Cause:** Variables not passed to `envsubst`.

**Diagnosis:**
```bash
docker exec nginx env | grep -E "API_VERSION|NUXT_UPSTREAM"
docker exec nginx cat /etc/nginx/nginx.conf | grep rewrite
```

**Solution:**
- Ensure `.env` has the variables
- Ensure `docker-compose.yml` passes them to `envsubst`
- Restart container: `docker compose down && docker compose up -d`

---

### 6. Changes to `.env` not reflected

**Symptom:** Changed `API_VERSION=v2` but NGINX still uses `v1`.

**Cause:** NGINX generates the config at startup, not on the fly.

**Solution:**
```bash
docker compose down
docker compose up -d
```

**Note:** `nginx -s reload` does NOT re-run `envsubst`.

---

### 7. WebSocket connection fails

**Symptom:** Socket.io can't connect.

**Diagnosis:**
```bash
docker logs nginx | grep -i "socket\|upgrade"
docker logs nest-api | grep -i "socket"
```

**Solution:** Ensure `/socket.io/` location has:
```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

---

### 8. Port 80 already in use

**Symptom:**
```text
Error: Bind for 0.0.0.0:80 failed: port is already allocated
```

**Diagnosis:**
```bash
sudo lsof -i :80
```

**Solution:**
```bash
# Stop the conflicting service
sudo systemctl stop apache2  # or nginx
# Or change NGINX port in .env
```

---

## 🎯 Best Practices

### 1. Always Use `app-network`

All containers should be on the same network for DNS resolution.

```bash
docker network create app-network 2>/dev/null || true
```

### 2. Use Container Names, Not `localhost`

Inside a container, `localhost` = the container itself.

| ✅ Correct | ❌ Incorrect |
|-----------|--------------|
| `http://nest-api:3001` | `http://localhost:3001` |

### 3. Version Your API in NGINX

Keep the frontend simple (`/api/`), let NGINX handle versioning (`/api/v1/`).

```nginx
rewrite ^/api/(.*)$ /api/${API_VERSION}/$1 break;
```

### 4. Use `envsubst` for Environment Variables

Don't hardcode values in `nginx.conf.template`. Use variables.

```nginx
server ${NUXT_UPSTREAM};  # ✅ Good
server nuxt-app:3000;      # ❌ Bad
```

### 5. Add Security Headers

Protect against common web vulnerabilities.

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
```

### 6. Enable Gzip Compression

Reduce bandwidth by ~70%.

```nginx
gzip on;
gzip_types text/plain text/css application/json ...;
```

### 7. Use Health Checks

Add a `/health` endpoint for monitoring.

```nginx
location /health {
    access_log off;
    return 200 "OK\n";
}
```

### 8. Log Everything

Use structured logging for debugging.

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
```

### 9. Test Configuration Before Reloading

Always test config before applying.

```bash
docker exec nginx nginx -t
```

### 10. Document Everything

Keep documentation up to date with configuration changes.

---

## 📎 Related Documentation

- [Docker Commands](./DOCKER_COMMANDS.md) — General Docker commands
- [Docker Nuxt Production](./DOCKER_NUXT_PROD.md) — Nuxt production setup
- [Docker Nest API](./DOCKER_NEST_API.md) — NestJS API setup (in NestJS repo)
- [Docker Troubleshooting](./DOCKER_TROUBLESHOOTING.md) — Common issues

---

## 🔗 External Resources

- [NGINX Official Documentation](https://nginx.org/en/docs/)
- [NGINX Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [NGINX WebSocket Proxying](https://nginx.org/en/docs/http/websocket.html)
- [NGINX Security Headers](https://owasp.org/www-project-secure-headers/)
- [envsubst Documentation](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html)

---


## 🔮 Future Enhancements

The following features are **not currently configured** but are documented for future reference.

### 1. SSL/HTTPS

**Status:** ❌ Not configured (development uses HTTP only)

**When to add:** Before production deployment.

**How:**
- Use Let's Encrypt with Certbot (production)
- Use mkcert for local development (if needed)

### 2. Static Files

**Status:** ❌ Not configured (S3 will serve static files)

**When to add:** Never (S3 is preferred for static assets).

**Why:** S3 is faster, cheaper, and more scalable than serving files from NGINX.

### 3. Rate Limiting

**Status:** ❌ Not configured

**When to add:** Before production deployment.

**How:**
```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

location /api/ {
    limit_req zone=api_limit burst=20 nodelay;
}
```

### Considerations:

- Shared IPs (NAT) can cause false positives
- Legitimate APIs (webhooks, bots) may be blocked
- Adjust limits based on real metrics

---

**Last updated:** 2026-09-20
