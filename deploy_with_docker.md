# 🚀 VPS Deployment Guide — Full Stack (Client + Admin + Backend + Redis)

A complete step-by-step guide to deploy a full-stack project on a VPS using **Docker**, **Nginx**, **SSL (Let's Encrypt)**, and **Redis**.

---

## 📁 Project Structure

```
myproject/
├─ client/
│  ├─ Dockerfile
│  └─ .env.production
├─ admin/
│  ├─ Dockerfile
│  └─ .env.production
├─ server/
│  ├─ Dockerfile
│  └─ .env.production
├─ nginx/
│  └─ conf.d/          # Optional global config
├─ redis/
└─ docker-compose.yml
```

| Service  | Internal Port | Exposed Port |
| -------- | ------------- | ------------ |
| Client   | 80            | 8080         |
| Admin    | 80            | 8081         |
| Backend  | 5000          | 5001         |

---

## 🛠️ Step 0: VPS Preparation

### Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Required Packages

```bash
sudo apt install docker.io docker-compose nginx certbot python3-certbot-nginx -y
```

### Enable & Start Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### Open HTTP/HTTPS Ports

```bash
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

---

## 📂 Step 1: Create Project Structure

```bash
mkdir -p ~/myproject/{client,admin,server,nginx,redis}
cd ~/myproject
```

---

## 🐳 Step 2: Dockerfiles

### `client/Dockerfile`

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### `admin/Dockerfile`

> Same as client. Adjust build context if needed.

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### `server/Dockerfile`

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
ENV NODE_ENV=production
EXPOSE 5000
CMD ["node", "index.js"]
```

---

## 🧩 Step 3: `docker-compose.yml`

```yaml
version: '3.9'

services:
  backend:
    build: ./server
    container_name: backend
    ports:
      - "127.0.0.1:5001:5000"
    env_file:
      - ./server/.env.production
    depends_on:
      - redis
    restart: unless-stopped

  admin:
    build: ./admin
    container_name: admin
    ports:
      - "127.0.0.1:8081:80"
    depends_on:
      - backend
    restart: unless-stopped

  client:
    build: ./client
    container_name: client
    ports:
      - "127.0.0.1:8080:80"
    depends_on:
      - backend
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: redis
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  redis_data:
```

---

## 🌐 Step 4: Nginx Reverse Proxy Configs

### Client — `/etc/nginx/sites-available/client.example.com`

```nginx
server {
    listen 80;
    server_name client.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name client.example.com;

    ssl_certificate /etc/letsencrypt/live/client.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/client.example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Admin — `/etc/nginx/sites-available/admin.example.com`

```nginx
server {
    listen 80;
    server_name admin.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name admin.example.com;

    ssl_certificate /etc/letsencrypt/live/admin.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/admin.example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Backend — `/etc/nginx/sites-available/api.example.com`

```nginx
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 🔗 Step 5: Enable Sites

```bash
sudo ln -s /etc/nginx/sites-available/client.example.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/admin.example.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/
```

---

## 🔒 Step 6: Generate SSL Certificates

```bash
sudo systemctl stop nginx

sudo certbot certonly --standalone -d client.example.com
sudo certbot certonly --standalone -d admin.example.com
sudo certbot certonly --standalone -d api.example.com

sudo systemctl start nginx
```

---

## ✅ Step 7: Test Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**Verify in browser:**

- 🌍 [https://client.example.com](https://client.example.com)
- 🔧 [https://admin.example.com](https://admin.example.com)
- ⚡ [https://api.example.com](https://api.example.com)

---

## 🚀 Step 8: Deploy Docker Containers

```bash
cd ~/myproject
docker-compose build
docker-compose up -d
```

**Useful Docker commands:**

```bash
# View running containers
docker-compose ps

# View logs
docker-compose logs -f

# Restart all services
docker-compose restart

# Stop all services
docker-compose down
```

---

## 🔄 Step 9: Auto SSL Renewal

Test renewal:

```bash
sudo certbot renew --dry-run
```

> ✅ Certbot sets up a systemd timer automatically that renews certificates before they expire. No manual cron job needed.

---

## 📝 License

This guide is free to use for personal and commercial projects.
