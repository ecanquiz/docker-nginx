# NGINX — Reverse Proxy Guide

NGINX serves as the reverse proxy for the Viñedosya e-commerce platform.

## Architecture

- `http://localhost/` → Nuxt (`nuxt-app:3000`)
- `http://localhost/api/` → NestJS (`nest-api:3001`)
- `http://localhost/socket.io/` → NestJS WebSockets
- `http://localhost/health` → NGINX health check

## Setup

```bash
cd ~/devenv/nginx
docker compose up -d
```

### Files

- `docker-compose.yml` — NGINX container
- `nginx.conf` — NGINX configuration
- `.env` — Environment variables



## ❓ Preguntas antes de empezar

1. **¿Estás de acuerdo con la estructura?** (carpeta separada `~/devenv/nginx/`)
2. **¿Quieres SSL/HTTPS local?** (para pruebas con certificados autofirmados)
3. **¿Quieres que NGINX sirva archivos estáticos directamente?** (optimización)
4. **¿Quieres que NGINX haga rate limiting?**

---

**¿Empezamos creando la carpeta y los archivos?** 🚀

Si tienes alguna preferencia específica (por ejemplo, SSL local, rate limiting, etc.), avísame y ajustamos la configuración.