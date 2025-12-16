# Riwi Wallet - Infrastructure and CI/CD

Complete documentation of the microservices-based (SOA) infrastructure with automated CI/CD for the Riwi Wallet project.

---

## 📋 Table of Contents

1. [General Architecture](#general-architecture)
2. [Initial VPS Setup](#initial-vps-setup)
3. [Deployed Services](#deployed-services)
4. [Implemented CI/CD](#implemented-cicd)
5. [Monitoring Stack](#monitoring-stack)
6. [Nginx Configuration](#nginx-configuration)
7. [Secrets Management](#secrets-management)
8. [Maintenance and Troubleshooting](#maintenance-and-troubleshooting)

---

## 🏗️ General Architecture

### Technology Stack

- **Backend API**: .NET Core 8.0
- **Chatbot**: Spring Boot (Java 21)
- **Frontend**: Vue.js 3.5 + Vite 7 + TypeScript + Pinia + TailwindCSS 4
- **Database**: PostgreSQL 16
- **Cache**: Redis 7
- **Automation**: n8n
- **Documentation**: Wiki.js
- **Monitoring**: Prometheus + Grafana + Portainer
- **Reverse Proxy**: Nginx
- **Containers**: Docker

### Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                    VPS (4GB RAM)                    │
├─────────────────────────────────────────────────────┤
│                                                     │
│  🌐 Nginx (Reverse Proxy)                           │
│      ├─ avaricia.crudzaso.com → Frontend (8080)    │
│      ├─ api.domain.com → .NET API (5001)           │
│      ├─ chatbot.domain.com → Chatbot (5002)        │
│      ├─ wiki.domain.com → Wiki.js (3000)           │
│      └─ n8n.domain.com → n8n (5678)                │
│                                                     │
│  📊 Monitoring                                      │
│      ├─ Prometheus (9090)                          │
│      ├─ Grafana (9000)                             │
│      └─ Portainer (9443)                           │
│                                                     │
│  🔵 Core Services                                  │
│      ├─ Vue.js Frontend (Port 8080)                │
│      │   └─ CI/CD: GitHub Actions                  │
│      ├─ .NET API (Port 5001)                       │
│      │   └─ CI/CD: GitHub Actions                  │
│      ├─ Spring Boot Chatbot (Port 5002)            │
│      │   └─ CI/CD: GitHub Actions                  │
│      ├─ Wiki.js (Port 3000)                        │
│      │   └─ Git Sync every 5 min                   │
│      └─ n8n (Port 5678)                            │
│          └─ Manual deployment                      │
│                                                     │
│  🗄️ Infrastructure                                │
│      ├─ PostgreSQL (Port 5432)                     │
│      │   └─ DBs: riwiwallet_db, wiki               │
│      └─ Redis (Port 6379)                          │
│                                                     │
│  🌐 Docker Network                                 │
│      └─ riwi_network (bridge)                      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 🚀 Initial VPS Setup

### 0. Firewall Configuration (UFW)

**Goal:** Implement the first layer of defense following the Principle of Least Privilege (deny all, allow only what is essential) to protect internal Docker services.

```bash
# Install UFW
sudo apt update
sudo apt install ufw -y

# Set default policies: Deny incoming, Allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential access: SSH (22) and Reverse Proxy (80, 443)
sudo ufw allow ssh comment 'Allow SSH access for administration'
sudo ufw allow http comment 'Allow HTTP web traffic'
sudo ufw allow https comment 'Allow HTTPS web traffic (SSL)'

# Enable Firewall
sudo ufw enable

# Check status
sudo ufw status numbered
```

### 1. Create Base Infrastructure

```bash
# Create Docker network
docker network create riwi_network

# Create persistent volumes
docker volume create riwi-wallet_riwi_data              # PostgreSQL
docker volume create riwi-wallet_riwi_wiki_assets       # Wiki.js
docker volume create riwi-wallet_n8n_data               # n8n
docker volume create riwi-wallet_redis_data             # Redis
```

### 2. Configure SSH for CI/CD

```bash
# On your local machine, generate SSH key pair
ssh-keygen -t ed25519 -C "ci-cd-riwi-wallet" -f ~/.ssh/riwi_cicd

# Copy public key to VPS
ssh-copy-id -i ~/.ssh/riwi_cicd.pub user@VPS_IP

# Verify connection
ssh -i ~/.ssh/riwi_cicd user@VPS_IP
```

---

## 🔵 Deployed Services

### PostgreSQL (Main Database)

```bash
docker run -d \
  --name db \
  --restart unless-stopped \
  --network riwi_network \
  -p 5432:5432 \
  -e POSTGRES_DB=riwiwallet_db \
  -e POSTGRES_USER=riwi_user \
  -e POSTGRES_PASSWORD=<PASSWORD> \
  -v riwi-wallet_riwi_data:/var/lib/postgresql/data \
  --memory="1500m" \
  --cpus="0.75" \
  postgres:16-alpine
```

**Databases:**
- `riwiwallet_db` - .NET API and Chatbot
- `wiki` - Wiki.js
- `n8n` - n8n (if configured)

### Redis (Cache and Queue)

```bash
docker run -d \
  --name riwi_redis \
  --restart unless-stopped \
  --network riwi_network \
  --memory="128m" \
  --cpus="0.25" \
  -v riwi-wallet_redis_data:/data \
  redis:7-alpine redis-server --maxmemory 100mb --maxmemory-policy allkeys-lru
```

### n8n (Automation)

```bash
docker run -d \
  --name riwi_n8n \
  --restart unless-stopped \
  --network riwi_network \
  -p 5678:5678 \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=<USER> \
  -e N8N_BASIC_AUTH_PASSWORD=<PASSWORD> \
  -e N8N_HOST=<DOMAIN_OR_IP> \
  -e DB_TYPE=postgresdb \
  -e DB_POSTGRESDB_HOST=db \
  -e DB_POSTGRESDB_PORT=5432 \
  -e DB_POSTGRESDB_DATABASE=n8n \
  -e DB_POSTGRESDB_USER=riwi_user \
  -e DB_POSTGRESDB_PASSWORD=<PASSWORD> \
  -e N8N_ENCRYPTION_KEY=<ENCRYPTION_KEY> \
  -e EXECUTIONS_MODE=queue \
  -e QUEUE_BULL_REDIS_HOST=riwi_redis \
  -e QUEUE_BULL_REDIS_PORT=6379 \
  -v riwi-wallet_n8n_data:/home/node/.n8n \
  --memory="768m" \
  --cpus="0.5" \
  n8nio/n8n:latest
```

**Generate encryption key:**
```bash
openssl rand -hex 32
```

### Wiki.js (Documentation)

```bash
# Create database
docker exec db psql -U riwi_user -c "CREATE DATABASE wiki;"

# Run Wiki.js
docker run -d \
  --name riwi_wiki \
  --restart unless-stopped \
  --network riwi_network \
  -p 3000:3000 \
  -e DB_TYPE=postgres \
  -e DB_HOST=db \
  -e DB_PORT=5432 \
  -e DB_USER=riwi_user \
  -e DB_PASS=<PASSWORD> \
  -e DB_NAME=wiki \
  -v riwi-wallet_riwi_wiki_assets:/wiki/data/content \
  --memory="512m" \
  --cpus="0.5" \
  ghcr.io/requarks/wiki:2
```

**Git Sync Configuration:**
1. Access `http://VPS_IP:3000`
2. Go to **Administration** → **Storage**
3. Configure Git with your documentation repository
4. Wiki.js will pull automatically every 5 minutes

### Vue.js Frontend (SPA)

```bash
# Container is deployed automatically via CI/CD
# Final configuration on the VPS:

docker run -d \
  --name front-vue \
  --restart unless-stopped \
  --network riwi_network \
  -p 8080:80 \
  --memory="512m" \
  --cpus="0.5" \
  <DOCKER_USERNAME>/front-vue:latest
```

**Frontend Stack:**
- Vue 3.5.25 (Composition API)
- Vite 7.1 (Ultra-fast build tool)
- TypeScript 5.9
- Pinia 3.0 (State management)
- Vue Router 4.6 (Routing)
- TailwindCSS 4.1 (Utility-first CSS)
- Axios 1.13 (HTTP client)
- Vue Toastification (Notifications)

**Build Features:**
- **Multi-stage build**: Reduces final image size
- **Optimized Nginx**: Gzip, asset caching, SPA routing
- **Environment variables**: API URLs injected at build time
- **Immutable cache**: Hashed assets cached for 1 year
- **Linux compatibility**: Regenerates package-lock.json during build

**Nginx Optimizations:**
- Gzip compression for all assets
- 1-year Cache-Control for `/assets/`
- `try_files` configured for SPA routing
- `index.html` never cached to always serve the latest version

---

## 🔄 Implemented CI/CD

### Vue.js Frontend

**Repository:** `Frontend`

**Structure:**
```
repo/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── nginx.conf
├── Dockerfile
├── .dockerignore
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.js
└── src/
    ├── main.ts
    ├── App.vue
    ├── router/
    ├── stores/
    ├── components/
    └── views/
```

**Dockerfile:** Multi-stage build (Node 20.19-alpine + Nginx Alpine)

**Required GitHub Secrets:**
- `DOCKER_USERNAME` - Docker Hub username
- `DOCKER_PASSWORD` - Docker Hub token
- `SSH_HOST` - VPS IP
- `SSH_USER` - SSH user
- `SSH_PRIVATE_KEY` - Full SSH private key
- `VITE_AUTH_API_URL` - Auth API URL (build time)
- `VITE_MAIN_API_URL` - Main API URL (build time)

**Workflow:** `.github/workflows/ci-cd.yml`

Trigger: Push to `main`

**Process:**
1. **Build and Push Job:**
   - Checkout code
   - Login to Docker Hub
   - Build Docker image with build args:
     - `VITE_AUTH_API_URL`
     - `VITE_MAIN_API_URL`
   - Push to Docker Hub with `latest` tag

2. **Deploy to VPS Job:**
   - SSH connection to VPS
   - Pull new image BEFORE stopping container (reduced downtime)
   - Stop and remove current container
   - Run new container with resource limits
   - Cleanup dangling images with `docker image prune -f`

**Pipeline Optimizations:**
- ✅ Pull image before stopping container (minimized downtime)
- ✅ Regenerate package-lock.json for Linux compatibility
- ✅ Remove npm dependency cache before install
- ✅ Build-time environment variable injection
- ✅ Automatic disk space cleanup

**Nginx Configuration (nginx.conf):**
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    gzip on;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript;

    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location / {
        add_header Cache-Control "no-store, no-cache, must-revalidate";
        try_files $uri $uri/ /index.html;
    }
}
```

### .NET API

**Repository:** `Micro-Back-Brahiam`

**Structure:**
```
repo/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── src/
│   └── API/
│       └── Dockerfile
└── ...
```

**Dockerfile location:** `src/API/Dockerfile`

**Required GitHub Secrets:**
- `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`
- `DOCKER_USERNAME`, `DOCKER_PASSWORD`
- `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_PORT`

Trigger: Push to `main` or `develop`

### Spring Boot Chatbot

**Repository:** `riwiwallet-sb-service`

**Additional GitHub Secrets:**
- All .NET API secrets plus:
- `OPENAI_API_KEY`, `OPENAI_MODEL`
- `TELEGRAM_BOT_TOKEN`
- `MS_CORE_API_KEY`
- `APP_FRONTEND_URL`
- `JWT_SECRET`, `JWT_EXPIRATION`
- `WHATSAPP_*`, `GOOGLE_*`, `MICROSOFT_*` (optional)

**Generate secrets:**
```bash
openssl rand -base64 64
openssl rand -hex 32
```

---

## 📊 Monitoring Stack

### Services

| Service | Port | RAM | Purpose |
|--------|------|-----|---------|
| Prometheus | 9090 | 256MB | Metrics collection |
| Grafana | 9000 | 256MB | Dashboards |
| Portainer | 9443 | 128MB | Container management |
| Node Exporter | 9100 | 64MB | VPS metrics |
| Postgres Exporter | 9187 | 64MB | PostgreSQL metrics |
| Redis Exporter | 9121 | 64MB | Redis metrics |

**Total usage:** ~832MB RAM

---

## 🌐 Nginx Configuration

Includes reverse proxy per service, WebSocket support, and SSL with Certbot.

---

## 🔐 Secrets Management

- Never hardcode credentials
- Always use GitHub Secrets
- Rotate credentials periodically
- Audit access regularly

---

## 🛠️ Maintenance and Troubleshooting

Includes Docker commands, logs, restarts, backups, common issues, and recovery procedures.

---

## 📈 Success Metrics

- 7+ services running
- RAM usage < 3.5GB
- Uptime > 99%
- API response time < 500ms
- Zero-downtime deployments

---

## 🤝 Contributing

This project follows an SOA architecture with independent repositories.

---

## 📝 Final Notes

Future improvements include staging environments, centralized logging, advanced alerts, and cloud backups.

---

*This README documents the complete infrastructure and CI/CD implementation for Riwi Wallet.*

