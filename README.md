# Production Deployment Guide

Fullstack MERN-based project deployed using Docker, Nginx, and Ubuntu VPS.

---

## 🌍 Architecture Overview

- 🌐 Frontend (Next.js) → https://shimanto.dev  
- ⚙️ Backend API (Express) → https://api.shimanto.dev  
- 🛠 Admin Panel (Vite) → https://admin.shimanto.dev  
- 🐳 Dockerized services  
- 🔐 Nginx reverse proxy + Let's Encrypt SSL  
- 🖥 Ubuntu VPS hosting  

---

# 1️⃣ DNS Setup (Domain Provider)

Add A Records:

```
@      → YOUR_VPS_IP
www    → YOUR_VPS_IP
api    → YOUR_VPS_IP
admin  → YOUR_VPS_IP
```

After DNS propagation:

```bash
ping shimanto.dev
ping api.shimanto.dev
ping admin.shimanto.dev
```

---

# 2️⃣ VPS Base Setup (Ubuntu)

## Update System + Firewall

```bash
sudo apt update && sudo apt -y upgrade

sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443

sudo ufw enable
sudo ufw status
```

---

## Install Docker + Compose Plugin

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo $VERSION_CODENAME) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install -y docker-ce docker-ce-cli \
containerd.io docker-buildx-plugin docker-compose-plugin
```

Add deploy user to docker group:

```bash
sudo usermod -aG docker $USER
```

Logout & login again.

---

# 3️⃣ Project Setup

```bash
sudo mkdir -p /var/www/shimanto
sudo chown -R $USER:$USER /var/www/shimanto

cd /var/www/shimanto
git clone YOUR_REPO_URL .
```

---

# 4️⃣ Production Folder Structure

```
/var/www/shimanto
│
├── server/
├── client/
├── admin/
├── docker-compose.prod.yml
│
└── infra/
    ├── nginx/
    │   ├── conf.d/
    │   └── www/
    └── letsencrypt/
```

---

# 5️⃣ Production Docker Compose

Create `docker-compose.prod.yml`:

```yaml
services:
  server:
    build: ./server
    env_file: ./server/.env.production
    restart: unless-stopped
    networks: [appnet]
    dns:
      - 1.1.1.1
      - 8.8.8.8

  client:
    build: ./client
    env_file: ./client/.env.production
    restart: unless-stopped
    networks: [appnet]
    depends_on: [server]

  admin:
    build: ./admin
    env_file: ./admin/.env.production
    restart: unless-stopped
    networks: [appnet]
    depends_on: [server]

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./infra/nginx/conf.d:/etc/nginx/conf.d:ro
      - ./infra/nginx/www:/var/www/certbot:ro
      - ./infra/letsencrypt:/etc/letsencrypt:ro
    depends_on: [client, admin, server]
    networks: [appnet]

networks:
  appnet:
    driver: bridge

```

⚠️ No internal service exposes ports publicly. All traffic goes through Nginx.

---

# 6️⃣ Nginx HTTP Config (Before SSL)

Create folders:

```bash
mkdir -p infra/nginx/conf.d
mkdir -p infra/nginx/www
mkdir -p infra/letsencrypt
```

Create `infra/nginx/conf.d/http.conf`:

```nginx
# Main Site
server {
  listen 80;
  server_name shimanto.dev www.shimanto.dev;

  location /.well-known/acme-challenge/ { root /var/www/certbot; }

  location / {
    proxy_pass http://client:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}

# API
server {
  listen 80;
  server_name api.shimanto.dev;

  location /.well-known/acme-challenge/ { root /var/www/certbot; }

  location / {
    proxy_pass http://server:5000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}

# Admin
server {
  listen 80;
  server_name admin.shimanto.dev;

  location /.well-known/acme-challenge/ { root /var/www/certbot; }

  location / {
    proxy_pass http://admin:80;
    proxy_set_header Host $host;
  }
}
```

---

# 7️⃣ Production Environment Variables

## client/.env.production

```
NEXT_PUBLIC_API_URL=https://api.shimanto.dev
```

## admin/.env.production

```
VITE_API_URL=https://api.shimanto.dev
```

## server/.env.production

Include:

- Database URI
- JWT Secret
- Cloudinary Config
- CORS Allowed Origins:

```
https://shimanto.dev
https://admin.shimanto.dev
```

---

# 8️⃣ First Run (HTTP Stage)

```bash
docker compose -f docker-compose.prod.yml up -d --build
docker compose -f docker-compose.prod.yml ps
```

Test:

- http://shimanto.dev
- http://api.shimanto.dev
- http://admin.shimanto.dev

---

# 9️⃣ SSL Setup (Let's Encrypt – Webroot)

Run separately:

```bash
docker run --rm \
  -v "$(pwd)/infra/letsencrypt:/etc/letsencrypt" \
  -v "$(pwd)/infra/nginx/www:/var/www/certbot" \
  certbot/certbot certonly \
  --webroot -w /var/www/certbot \
  -d shimanto.dev -d www.shimanto.dev \
  --email your@email.com --agree-tos --no-eff-email
```

Repeat for:

- api.shimanto.dev  
- admin.shimanto.dev  

---

# 🔐 HTTPS Configuration

Replace HTTP config with HTTPS config including:

- HTTP → HTTPS redirect
- SSL certificate paths:
  - `/etc/letsencrypt/live/<domain>/fullchain.pem`
  - `/etc/letsencrypt/live/<domain>/privkey.pem`

Restart nginx:

```bash
docker compose -f docker-compose.prod.yml restart nginx
```

---

# 🔄 Auto Renew SSL

Manual renew:

```bash
docker run --rm \
  -v "$(pwd)/infra/letsencrypt:/etc/letsencrypt" \
  -v "$(pwd)/infra/nginx/www:/var/www/certbot" \
  certbot/certbot renew --webroot -w /var/www/certbot
```

Add weekly cron:

```bash
crontab -e
```

Add:

```
0 3 * * 1 cd /var/www/shimanto && docker run --rm -v "$(pwd)/infra/letsencrypt:/etc/letsencrypt" -v "$(pwd)/infra/nginx/www:/var/www/certbot" certbot/certbot renew --webroot -w /var/www/certbot && docker compose -f docker-compose.prod.yml restart nginx
```

---

# 🚀 Deployment Update Flow

```bash
cd /var/www/shimanto
git pull
docker compose -f docker-compose.prod.yml up -d --build
docker compose -f docker-compose.prod.yml logs -f --tail=200
```

---

# 🛡 Security Notes

- Only ports 80 & 443 exposed publicly
- Internal services are isolated
- SSL enforced
- Proper CORS configuration
- Docker network isolation

---

# 📦 Tech Stack

- Node.js
- Express
- Next.js
- MongoDB
- Docker
- Nginx
- Let's Encrypt
- Ubuntu VPS

---

## 👨‍💻 Author

Shimanto Sarkar  
Fullstack MERN Developer  
