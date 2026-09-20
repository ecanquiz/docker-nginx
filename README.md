# docker-nginx

<p align="center">
  <a href="https://nginx.org/" target="_blank">
    <img src="https://nginx.org/nginx.png" width="180" alt="NGINX Logo" />
  </a>
</p>

<p align="center">
  <strong>NGINX</strong> — Reverse proxy for the Viñedosya e-commerce platform
</p>

<p align="center">
  <a href="https://nginx.org/" target="_blank"><img src="https://img.shields.io/badge/NGINX-1.27-009639?logo=nginx&logoColor=white" alt="NGINX" /></a>
  <a href="https://www.docker.com/" target="_blank"><img src="https://img.shields.io/badge/Docker-✓-2496ED?logo=docker&logoColor=white" alt="Docker" /></a>
  <a href="https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html" target="_blank"><img src="https://img.shields.io/badge/envsubst-✓-4EAA25?logo=gnubash&logoColor=white" alt="envsubst" /></a>
</p>

<p align="center">
  <a href="./LICENSE"><img src="https://img.shields.io/badge/license-UNLICENSED-lightgrey" alt="License" /></a>
</p>

---

## 📖 Overview

**docker-nginx** is the **NGINX reverse proxy** for the Viñedosya e-commerce platform. It serves as the **single entry point** for the application, routing requests to the appropriate backend service.

### 🎯 Purpose

| URL | Routes to | Purpose |
|-----|-----------|---------|
| `http://localhost/` | `nuxt-app:3000` | Nuxt SSR (frontend) |
| `http://localhost/api/*` | `nest-api:3001/api/v1/*` | NestJS REST API |
| `http://localhost/socket.io/*` | `nest-api:3001` | Socket.io WebSockets |
| `http://localhost/health` | NGINX itself | Health check |

### 🔗 Integration

This proxy works alongside:

| Service | Technology | Repository |
|---------|------------|------------|
| **Frontend** | Nuxt 4 + Nitro + Vue 3 | [`e-commerce-nuxt`](https://github.com/your-org/e-commerce-nuxt) |
| **Backend** | NestJS 11 + TypeORM + PostgreSQL | [`invoice-nest`](https://github.com/your-org/invoice-nest) |
| **Database** | PostgreSQL 13 | Dockerized |
| **Cache / Queues** | Redis | Dockerized |
| **Mail** | Mailhog (dev) / SMTP (prod) | Dockerized |
| **Reverse Proxy** | **NGINX** | `docker-nginx` (this repo) |

All services communicate through a shared Docker network (`app-network`).

---

## 🏗️ Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                    USER BROWSER                             │
│                http://localhost/                            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                       NGINX                                 │
│                    (Reverse Proxy)                          │
│                       :80, :443                             │
│                                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐        │
│  │  /api/* → nest-api   │  │  /* → nuxt-app       │        │
│  └──────────────────────┘  └──────────────────────┘        │
└───────────────────────────┬─────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
    ┌──────────────────┐      ┌──────────────────┐
    │   Nuxt (SSR)     │      │   NestJS API     │
    │   :3000          │      │   :3001          │
    └──────────────────┘      └──────────────────┘
              │                           │
              └─────────────┬─────────────┘
                            │
                   ┌────────▼────────┐
                   │  app-network    │
                   └────────┬────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   ┌────▼────┐        ┌────▼────┐        ┌────▼─────┐
   │Postgres │        │  Redis  │        │ Mailhog  │
   └─────────┘        └─────────┘        └──────────┘
```

---

## ✨ Features

- 🚀 **Reverse proxy** for Nuxt (SSR) and NestJS (API)
- 🔀 **API version rewrite** (`/api/` → `/api/v1/`) via `API_VERSION` variable
- 🔌 **WebSocket support** (Socket.io)
- 🗜️ **Gzip compression** (reduces bandwidth by ~70%)
- 🔒 **Security headers** (X-Frame-Options, X-Content-Type-Options, etc.)
- 🩺 **Health check** endpoint (`/health`)
- ⚙️ **Environment variables** via `envsubst` (no hardcoded values)
- 🐳 **Docker-ready** (single `docker compose up -d`)

---

## 🚀 Quick Start

### Prerequisites

- **Docker** + **Docker Compose**
- Shared network `app-network`
- `nuxt-app` running (port 3000)
- `nest-api` running (port 3001)

### 1. Clone the repository

```bash
git clone <repository-url>
cd docker-nginx
```

### 2. Create `.env`

```bash
cp .env.example .env
```

**Edit `.env` with your values:**

```env
NGINX_PORT=80
NGINX_SSL_PORT=443

NUXT_UPSTREAM=nuxt-app:3000
NEST_UPSTREAM=nest-api:3001

API_VERSION=v1

NODE_ENV=production
```

### 3. Create shared network (once)

```bash
docker network create app-network
```

### 4. Start NGINX

```bash
docker compose up -d
docker compose logs -f
```

### 5. Verify

```bash
# Frontend
curl -I http://localhost/

# API
curl http://localhost/api/products/public | head -5

# Health
curl http://localhost/health
```

---

## 📁 Project Structure

```text
nginx/
├── .env                    # Environment variables (NOT committed)
├── .env.example            # Environment variables template (committed)
├── .gitignore              # Git ignore rules
├── docker-compose.yml      # NGINX container orchestration
├── nginx.conf.template     # NGINX configuration template
├── README.md               # This file
├── docs/
│   └── DOCKER_NGINX.md     # Complete NGINX documentation
└── logs/                   # NGINX logs (NOT committed)
```

---

## 📚 Documentation

Complete documentation is available in the [`docs/`](./docs) folder.

| Document | Description |
|----------|-------------|
| [DOCKER_NGINX.md](./docs/DOCKER_NGINX.md) | Complete NGINX reverse proxy guide |

### What's inside `DOCKER_NGINX.md`

- 📖 **What is NGINX?** — Introduction and key features
- 🎯 **Why a Reverse Proxy?** — Benefits over direct access
- 🏗️ **Architecture** — Request routing and network setup
- 📄 **Configuration Files** — `docker-compose.yml`, `nginx.conf.template`, `.env`
- 🧠 **Core Concepts** — Upstreams, location blocks, proxy_pass, rewrite, WebSockets, security headers, gzip
- 🚀 **Setup** — Step-by-step installation
- 🔄 **Environment Variables with envsubst** — How to use variables in NGINX
- 📚 **How We Got Here (Evolution)** — From hardcoded config to envsubst
- ✅ **Verification** — Endpoint and header tests
- 🔴 **Troubleshooting** — 8 common issues and solutions
- 🎯 **Best Practices** — 10 recommendations for NGINX

---

## 🔧 Common Commands

| Command | Description |
|---------|-------------|
| `docker compose up -d` | Start NGINX |
| `docker compose down` | Stop NGINX |
| `docker compose logs -f` | Follow logs |
| `docker compose restart nginx` | Restart NGINX |
| `docker exec nginx nginx -t` | Test configuration |
| `docker exec nginx nginx -s reload` | Reload configuration (without envsubst) |
| `docker exec nginx cat /etc/nginx/nginx.conf` | View generated config |

---

## ⚙️ Environment Variables

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `NGINX_PORT` | HTTP port | `80` | `80` |
| `NGINX_SSL_PORT` | HTTPS port | `443` | `443` |
| `NUXT_UPSTREAM` | Nuxt container name and port | `nuxt-app:3000` | `nuxt-app:3000` |
| `NEST_UPSTREAM` | NestJS container name and port | `nest-api:3001` | `nest-api:3001` |
| `API_VERSION` | API version for rewrite | `v1` | `v1`, `v2` |
| `NODE_ENV` | Environment | `production` | `production`, `development` |

---

## 🔄 Changing the API Version

**Without touching `nginx.conf.template`:**

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

## ⚠️ Important Notes

### 1. Use container names, not `localhost`

Inside a container, `localhost` = the container itself.

| ✅ Correct | ❌ Incorrect |
|-----------|--------------|
| `http://nest-api:3001` | `http://localhost:3001` |

### 2. `envsubst` runs at startup

NGINX generates the config from the template **only at container startup**.

To apply changes to `.env`:
```bash
docker compose down
docker compose up -d
```

**`nginx -s reload` does NOT re-run `envsubst`.**

### 3. Only NGINX is exposed

In production, only NGINX should be exposed to the internet. Nuxt and NestJS should be internal.

### 4. SSL/HTTPS in production

This setup uses HTTP (`:80`). For production, configure SSL certificates:

```nginx
server {
    listen 443 ssl;
    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    ...
}
```

---

## 🔴 Troubleshooting

**Quick reference** (full details in [DOCKER_NGINX.md](./docs/DOCKER_NGINX.md)):

| Issue | Cause | Solution |
|-------|-------|----------|
| `404 Not Found` on `/api/*` | Missing rewrite rule | Add `rewrite ^/api/(.*)$ /api/${API_VERSION}/$1 break;` |
| `502 Bad Gateway` | `nuxt-app` or `nest-api` not running | Start the missing service |
| `504 DNS look up failed` | Corporate firewall (Fortinet) | Run tests from host |
| NGINX can't resolve upstream | Not on `app-network` | `docker network connect app-network nginx` |
| `envsubst` not substituting | Variables not passed | Check `.env` and `docker-compose.yml` |
| Changes to `.env` not reflected | Config generated at startup | `docker compose down && docker compose up -d` |
| WebSocket fails | Missing upgrade headers | Add `Upgrade` and `Connection` headers |
| Port 80 in use | Another service uses it | Stop the conflicting service |

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feat/my-feature`
3. Commit your changes: `git commit -m "feat: add my feature"`
4. Push to the branch: `git push origin feat/my-feature`
5. Open a Pull Request

**Commit convention:** [Conventional Commits](https://www.conventionalcommits.org/)

---

## 📝 License

This project is **UNLICENSED** — proprietary and confidential.

---

<p align="center">
  Made with ❤️ by the Viñedosya team
</p>

<p align="center">
  <a href="https://nginx.org/">NGINX</a> ·
  <a href="https://www.docker.com/">Docker</a> ·
  <a href="https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html">envsubst</a>
</p>

