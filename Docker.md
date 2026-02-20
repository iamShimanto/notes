# 🐳 Docker Commands & Usage Guide

This section explains how to use Docker and Docker Compose for development and production in this project.

---

# 📦 Docker Basics

## 🔹 Check Docker Version

```bash
docker --version
docker compose version
```

---

## 🔹 Check Running Containers

```bash
docker ps
```

Show all containers (including stopped):

```bash
docker ps -a
```

---

## 🔹 Check Docker Images

```bash
docker images
```

---

## 🔹 Remove Unused Images

```bash
docker image prune -a
```

---

# 🚀 Docker Compose – Production Usage

We use:

```bash
docker compose -f docker-compose.prod.yml
```

---

## 🔹 Build & Start Containers (First Run)

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

- `-d` → detached mode (background)
- `--build` → rebuild images

---

## 🔹 Check Container Status

```bash
docker compose -f docker-compose.prod.yml ps
```

---

## 🔹 View Logs (All Services)

```bash
docker compose -f docker-compose.prod.yml logs
```

Live logs:

```bash
docker compose -f docker-compose.prod.yml logs -f
```

Last 200 lines:

```bash
docker compose -f docker-compose.prod.yml logs -f --tail=200
```

---

## 🔹 Restart Services

Restart all:

```bash
docker compose -f docker-compose.prod.yml restart
```

Restart specific service:

```bash
docker compose -f docker-compose.prod.yml restart nginx
```

---

## 🔹 Stop Containers

```bash
docker compose -f docker-compose.prod.yml down
```

Stop & remove volumes:

```bash
docker compose -f docker-compose.prod.yml down -v
```

---

## 🔹 Rebuild After Code Update

```bash
git pull
docker compose -f docker-compose.prod.yml up -d --build
```

---

# 🧪 Run Single Service

Start only server:

```bash
docker compose -f docker-compose.prod.yml up server
```

Start only nginx:

```bash
docker compose -f docker-compose.prod.yml up nginx
```

---

# 🔍 Enter Running Container (Debug)

List containers:

```bash
docker ps
```

Enter container:

```bash
docker exec -it CONTAINER_NAME sh
```

Example:

```bash
docker exec -it joudperfume-server-1 sh
```

---

# 🗑 Remove Specific Container

```bash
docker rm CONTAINER_ID
```

Force remove:

```bash
docker rm -f CONTAINER_ID
```

---

# 🧹 Clean Docker System (Careful ⚠️)

Remove unused containers, images, networks:

```bash
docker system prune -a
```

---

# 🔐 Nginx Container Commands

Test nginx config:

```bash
docker exec -it joudperfume-nginx-1 nginx -t
```

Reload nginx:

```bash
docker exec -it joudperfume-nginx-1 nginx -s reload
```

---

# 🔄 SSL Renew Command

```bash
docker run --rm \
  -v "$(pwd)/infra/letsencrypt:/etc/letsencrypt" \
  -v "$(pwd)/infra/nginx/www:/var/www/certbot" \
  certbot/certbot renew --webroot -w /var/www/certbot
```

Restart nginx after renew:

```bash
docker compose -f docker-compose.prod.yml restart nginx
```

---

# 🧠 Docker Networking

Check networks:

```bash
docker network ls
```

Inspect network:

```bash
docker network inspect joudperfume_appnet
```

---

# 📊 Monitor Resource Usage

```bash
docker stats
```

---

# ⚡ Quick Production Cheat Sheet

```bash
# Deploy
git pull
docker compose -f docker-compose.prod.yml up -d --build

# Logs
docker compose -f docker-compose.prod.yml logs -f --tail=200

# Restart nginx
docker compose -f docker-compose.prod.yml restart nginx

# Stop everything
docker compose -f docker-compose.prod.yml down
```

---

# 🛡 Best Practices

- Always use `--build` after code changes
- Keep `.env.production` secure
- Do not expose internal ports
- Use `docker system prune` carefully
- Monitor logs regularly
- Restart nginx after SSL renew

---

# 🏁 Summary

Docker is used to:

- Run isolated services (server, client, admin)
- Use single Nginx reverse proxy
- Handle SSL certificates
- Simplify deployment
- Ensure consistent production environment

---

Happy Deploying 🚀
