# Riwi Wallet - Infrastructure and CI/CD

Complete documentation of the microservices-based infrastructure (SOA) with automated CI/CD for the Riwi Wallet project.

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
9. [VPS Security](#vps-security)

---

## 🏗️ General Architecture

### Technology Stack

- **Backend API**: .NET Core 8.0
- **Chatbot**: Spring Boot (Java 21)
- **Frontend**: Vue.js (pending)
- **Database**: PostgreSQL 16
- **Cache**: Redis 7
- **Automation**: n8n
- **Documentation**: Wiki.js
- **Monitoring**: Prometheus + Grafana + Portainer
- **Reverse proxy**: Nginx
- **Containers**: Docker

### Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                    VPS (4GB RAM)                    │
├─────────────────────────────────────────────────────┤
│                                                     │
│  🌐 Nginx (Reverse Proxy)                          │
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
│  🔵 Main Services                                   │
│      ├─ .NET API (Port 5001)                       │
│      │   └─ CI/CD: GitHub Actions                  │
│      ├─ Spring Boot Chatbot (Port 5002)            │
│      │   └─ CI/CD: GitHub Actions                  │
│      ├─ Wiki.js (Port 3000)                        │
│      │   └─ Git Sync every 5 min                   │
│      └─ n8n (Port 5678)                            │
│          └─ Manual deployment                      │
│                                                     │
│  🗄️ Infrastructure                                 │
│      ├─ PostgreSQL (Port 5432)                     │
│      │   └─ DBs: riwiwallet_db, wiki               │
│      └─ Redis (Port 6379)                          │
│                                                     │
│  🌐 Docker Network                                  │
│      └─ riwi_network (bridge)                      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 🚀 Initial VPS Setup

### 0. Firewall Configuration (UFW)

**Goal:** Implement the first layer of defense following the Principle of Least Privilege (deny all, allow only essentials) to protect internal Docker services.

```bash
# Install UFW
sudo apt update
sudo apt install ufw -y

# Set default policies: Deny incoming, Allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential access: SSH (port 22) and Reverse Proxy (80, 443)
sudo ufw allow ssh comment 'Allow SSH access for administration'
sudo ufw allow http comment 'Allow HTTP web traffic'
sudo ufw allow https comment 'Allow HTTPS web traffic (SSL)'

# Enable Firewall
sudo ufw enable

# Verify status
sudo ufw status numbered
```

**Expected result:**
```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere                   # Allow SSH access for administration
[ 2] 80/tcp                     ALLOW IN    Anywhere                   # Allow HTTP web traffic
[ 3] 443/tcp                    ALLOW IN    Anywhere                   # Allow HTTPS web traffic (SSL)
[ 4] 22/tcp (v6)                ALLOW IN    Anywhere (v6)              # Allow SSH access for administration
[ 5] 80/tcp (v6)                ALLOW IN    Anywhere (v6)              # Allow HTTP web traffic
[ 6] 443/tcp (v6)               ALLOW IN    Anywhere (v6)              # Allow HTTPS web traffic (SSL)
```

**Important notes:**
- ✅ **Only these ports are externally accessible**. All Docker services (PostgreSQL, Redis, n8n, Wiki.js, Grafana, Portainer) are **protected** and only accessible through Nginx or within the Docker network.
- ⚠️ **Do not expose service ports directly** (5001, 5002, 3000, 5678, 9000, 9090, etc.) unless absolutely necessary for temporary development/debugging.
- 🔒 **The Principle of Least Privilege** ensures only HTTP/HTTPS (reverse proxy) and SSH (administration) are accessible from the Internet.

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

**Why?** Volumes persist data even if containers are deleted. The network allows containers to communicate by name.

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

# Start Wiki.js
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

**Git Sync configuration:**
1. Access `http://VPS_IP:3000`
2. Go to **Administration** → **Storage**
3. Configure Git with your documentation repository
4. Wiki.js will pull every 5 minutes automatically

---

## 🔄 Implemented CI/CD

### .NET API

**Repository:** `riwi-wallet-api-net` (or your name)

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
- `VPS_HOST` - VPS IP
- `VPS_USER` - SSH user
- `VPS_SSH_KEY` - Complete SSH private key
- `DOCKER_USERNAME` - Docker Hub user
- `DOCKER_PASSWORD` - Docker Hub token
- `DB_HOST` - `db`
- `DB_NAME` - `riwiwallet_db`
- `DB_USER` - `riwi_user`
- `DB_PASSWORD` - PostgreSQL password
- `DB_PORT` - `5432`

**Workflow:** `.github/workflows/ci-cd.yml`

Trigger: Push to `main` or `development`

**Process:**
1. Checkout code
2. Login to Docker Hub
3. Build Docker image (multi-stage if applicable)
4. Push to Docker Hub with `latest` and `SHA` tags
5. SSH to VPS
6. Stop previous container
7. Pull new image
8. Start new container
9. Verify status
10. Clean old images

### Spring Boot Chatbot

**Repository:** `riwiwallet-sb-service`

**Structure:**
```
repo/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── Dockerfile
├── .dockerignore
├── pom.xml
├── src/
└── ...
```

**Dockerfile:** Project root (multi-stage with Maven)

**Additional GitHub Secrets:**
- All from .NET API +
- `OPENAI_API_KEY`
- `OPENAI_MODEL` (e.g., `gpt-4o-mini`)
- `TELEGRAM_BOT_TOKEN`
- `MS_CORE_API_KEY`
- `APP_FRONTEND_URL`
- `JWT_SECRET`
- `JWT_EXPIRATION`
- `WHATSAPP_ACCESS_TOKEN` (empty if not used)
- `WHATSAPP_PHONE_NUMBER_ID` (empty if not used)
- `WHATSAPP_VERIFY_TOKEN` (empty if not used)
- `GOOGLE_CLIENT_ID` (empty if not using Google OAuth)
- `GOOGLE_CLIENT_SECRET` (empty if not used)
- `MICROSOFT_CLIENT_ID` (empty if not using Microsoft OAuth)
- `MICROSOFT_CLIENT_SECRET` (empty if not used)

**Generate secrets:**
```bash
# JWT Secret
openssl rand -base64 64

# MS Core API Key
openssl rand -hex 32
```

**Important environment variables:**
- `SPRING_PROFILES_ACTIVE=production`
- `ms.core.base-url=http://netapi_gestor_finanzas:8080`
- `server.forward-headers-strategy=FRAMEWORK`
- `server.use-forward-headers=true`
- All Spring Boot configurations from secrets

### Advantages of Implemented CI/CD

✅ **Automatic deployment** on every push  
✅ **Zero downtime** - only affected service restarts  
✅ **Easy rollback** - tags with commit SHA  
✅ **Security** - credentials in GitHub Secrets, never in code  
✅ **Independence** - each service with its own pipeline  
✅ **Traceability** - complete history in GitHub Actions  

---

## 📊 Monitoring Stack

### Installation

```bash
# Create directory
mkdir -p ~/monitoring
cd ~/monitoring

# Create prometheus.yml (see content below)
nano prometheus.yml

# Create docker-compose.yml (see content below)
nano docker-compose.yml

# Start services
docker-compose up -d
```

### Stack Services

| Service | Port | RAM | Function |
|---------|------|-----|----------|
| Prometheus | 9090 | 256MB | Metrics collection and storage |
| Grafana | 9000 | 256MB | Visualization (dashboards) |
| Portainer | 9443 | 128MB | Visual container management |
| Node Exporter | 9100 | 64MB | VPS metrics |
| Postgres Exporter | 9187 | 64MB | PostgreSQL metrics |
| Redis Exporter | 9121 | 64MB | Redis metrics |

**Total consumption:** ~832MB RAM

### Interface Access

- **Prometheus:** `http://VPS_IP:9090`
- **Grafana:** `http://VPS_IP:9000`
  - User: `admin`
  - Password: (configured in `.env` or `admin123` by default)
- **Portainer:** `https://VPS_IP:9443`
  - Create admin user on first access

### Grafana Configuration

1. **Add Data Source:**
   - Configuration → Data sources → Add data source
   - Select Prometheus
   - URL: `http://prometheus:9090`
   - Save & Test

2. **Import Dashboards:**
   - + → Import
   - Use these IDs:
     - `1860` - Node Exporter Full (VPS)
     - `14282` - cAdvisor alternative / Docker
     - `9628` - PostgreSQL Database
     - `763` - Redis Dashboard

### Monitored Metrics

**System (Node Exporter):**
- CPU usage, load average
- RAM usage, swap
- Available disk, I/O
- Network traffic

**PostgreSQL:**
- Active connections
- Queries per second
- Cache hit ratio
- Locks and deadlocks

**Redis:**
- Connections
- Total keys
- Hit/miss ratio
- Memory used

**Chatbot (Spring Boot Actuator):**
- JVM heap memory
- HTTP requests
- Response times
- Thread count

**Containers (Portainer):**
- Status of each container
- CPU and RAM per container
- Network I/O
- Real-time logs

---

## 🌐 Nginx Configuration

### Installation

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

### File Structure

```
/etc/nginx/
├── sites-available/
│   ├── api.riwi-wallet.conf
│   ├── chatbot.riwi-wallet.conf
│   ├── wiki.riwi-wallet.conf
│   └── n8n.riwi-wallet.conf
└── sites-enabled/
    └── (symlinks to sites-available)
```

### Per-Service Configuration

#### Chatbot (Example)

```nginx
# /etc/nginx/sites-available/chatbot.riwi-wallet.conf

upstream chatbot-backend {
    server localhost:5002;
    keepalive 16;
}

server {
    listen 80;
    server_name chatbot.your-domain.com;
    
    location / {
        proxy_pass http://chatbot-backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port 443;
        proxy_set_header X-Forwarded-Ssl on;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        proxy_redirect off;
    }
}
```

**Enable configuration:**
```bash
sudo ln -s /etc/nginx/sites-available/chatbot.riwi-wallet.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### SSL with Certbot

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Get certificates
sudo certbot --nginx -d chatbot.your-domain.com -d api.your-domain.com -d wiki.your-domain.com

# Automatic renewal (already configured by Certbot)
sudo certbot renew --dry-run
```

**Required DNS configuration:**
```
Type: A
Name: chatbot
Value: VPS_IP
TTL: 3600
```

Repeat for each subdomain (api, wiki, n8n).

---

## 🔐 VPS Security

### Implemented Security Principles

This project follows cloud infrastructure security best practices:

1. **Principle of Least Privilege** - Only essential ports are exposed
2. **Defense in Depth** - Multiple security layers (Firewall, Nginx, Docker networks)
3. **Secrets Management** - Credentials never in code, always in GitHub Secrets
4. **Reverse Proxy** - Nginx as the only entry point
5. **Isolated Networks** - Services in private Docker network

### Security Architecture

```
Internet
   ↓
[UFW Firewall] ← Only ports 22, 80, 443
   ↓
[Nginx Reverse Proxy] ← Rate limiting, security headers
   ↓
[Docker Network: riwi_network] ← Services isolated internally
   ↓
[Containers] ← Internal communication by names
```

### Security Configuration by Layer

#### 1. Firewall (UFW)

**Status:** Active and enabled at boot

**Configured rules:**
```bash
# View current rules
sudo ufw status numbered

# Essential rules
22/tcp  - SSH (administration)
80/tcp  - HTTP (redirects to HTTPS)
443/tcp - HTTPS (Nginx reverse proxy)
```

**Rule management:**
```bash
# Allow temporary port (e.g., debugging)
sudo ufw allow 9000/tcp comment 'Temporary: Grafana debug'

# Delete rule by number
sudo ufw status numbered
sudo ufw delete [number]

# Reload firewall
sudo ufw reload
```

#### 2. Nginx as Reverse Proxy

**Security benefits:**
- ✅ Rate limiting (DDoS protection)
- ✅ Security headers (X-Frame-Options, CSP, etc.)
- ✅ SSL/TLS termination (Let's Encrypt certificates)
- ✅ Malicious request filtering
- ✅ Centralized logging

**Implemented security headers:**
```nginx
server_tokens off;                                           # Hide Nginx version
add_header X-Frame-Options "DENY" always;                   # Anti-clickjacking
add_header X-Content-Type-Options "nosniff" always;         # Anti-MIME sniffing
add_header X-XSS-Protection "1; mode=block" always;         # XSS protection
add_header Referrer-Policy "strict-origin" always;          # Referrer control
```

**Rate limiting per service:**
```nginx
# Chatbot: 50 requests/minute
limit_req_zone $binary_remote_addr zone=chatbot_limit:10m rate=50r/m;

# Webhooks: 100 requests/minute
limit_req_zone $binary_remote_addr zone=webhook_limit:10m rate=100r/m;
```

#### 3. Isolated Docker Networks

**Main network:** `riwi_network`

**Access by internal names:**
- `db` → PostgreSQL (port 5432, NOT externally exposed)
- `riwi_redis` → Redis (port 6379, NOT exposed)
- `netapi_gestor_finanzas` → .NET API
- `riwiwallet_chatbot` → Chatbot

**Verify isolation:**
```bash
# View services on the network
docker network inspect riwi_network

# Services are NOT accessible from Internet directly
# Only through Nginx (reverse proxy)
```

#### 4. Secrets Management

**GitHub Secrets (per repository):**
- Encryption at rest
- Audited access
- Never visible in logs
- Rotation without code changes

**Never in code:**
```bash
❌ password: "J9YoXTAy77bVPxwMtArRHfXDC"
✅ password: ${{ secrets.DB_PASSWORD }}
```

**Best practices:**
```bash
# Generate secure secrets
openssl rand -base64 32    # Passwords
openssl rand -hex 32       # API Keys
openssl rand -base64 64    # JWT Secrets

# Rotate credentials regularly
# 1. Generate new secret
# 2. Update in GitHub Secrets
# 3. Automatic redeploy via CI/CD
```

### OAuth Configuration in Nginx

For OAuth (Google, Microsoft) to work correctly with reverse proxy:

```nginx
# CRITICAL headers for OAuth
proxy_set_header Host $host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port 443;
proxy_set_header X-Forwarded-Ssl on;
proxy_redirect off;
```

**Environment variables in Spring Boot:**
```bash
-e "server.forward-headers-strategy=FRAMEWORK"
-e "server.use-forward-headers=true"
```
---

## 🔐 Secrets Management

### Security Principles

✅ **NEVER** hardcode credentials in code  
✅ **ALWAYS** use GitHub Secrets for CI/CD  
✅ **ROTATE** credentials regularly  
✅ **AUDIT** secret access in GitHub  

### Secrets per Repository

#### .NET API
```
VPS_HOST, VPS_USER, VPS_SSH_KEY
DOCKER_USERNAME, DOCKER_PASSWORD
DB_HOST, DB_NAME, DB_USER, DB_PASSWORD, DB_PORT
```

#### Chatbot
```
(All from .NET API +)
OPENAI_API_KEY, OPENAI_MODEL
TELEGRAM_BOT_TOKEN
MS_CORE_API_KEY, APP_FRONTEND_URL
JWT_SECRET, JWT_EXPIRATION
WHATSAPP_*, GOOGLE_*, MICROSOFT_* (optional)
```

### Add Secret in GitHub

1. Go to repository
2. Settings → Secrets and variables → Actions
3. New repository secret
4. Name: `SECRET_NAME`
5. Value: `secret_value`
6. Add secret

### Generate Secure Credentials

```bash
# JWT Secret (64 bytes in base64)
openssl rand -base64 64

# API Keys (32 bytes in hex)
openssl rand -hex 32

# Passwords (random generator)
openssl rand -base64 32

# SSH Key pair for CI/CD
ssh-keygen -t ed25519 -C "ci-cd-service-name"
```

---

## 🛠️ Maintenance and Troubleshooting

### Useful Commands

#### View Service Status
```bash
# All containers
docker ps

# Only Riwi Wallet services
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep riwi

# View resource usage
docker stats
```

#### Logs
```bash
# View service logs
docker logs riwiwallet_chatbot

# Last 50 lines
docker logs riwiwallet_chatbot --tail 50

# Follow logs in real time
docker logs riwiwallet_chatbot -f

# CI/CD logs
# View in GitHub → Actions → specific workflow
```

#### Restart Services
```bash
# Restart a container
docker restart riwiwallet_chatbot

# Restart entire monitoring stack
cd ~/monitoring
docker-compose restart

# Restart Nginx
sudo systemctl restart nginx
```

#### Clean Resources
```bash
# Clean unused images
docker image prune -a

# Clean stopped containers
docker container prune

# Clean everything (CAUTION: doesn't delete volumes)
docker system prune -a

# View disk usage
docker system df
```

### Common Issues

#### 1. Container restarts constantly

**Symptom:** `docker ps` shows "Restarting"

**Diagnosis:**
```bash
docker logs container_name --tail 100
docker inspect container_name | grep -A 10 "State"
```

**Common causes:**
- DB connection error (verify PostgreSQL is running)
- Missing environment variable
- Port in use
- Out of memory

#### 2. CI/CD fails on deploy

**Symptom:** Workflow in red in GitHub Actions

**Diagnosis:**
- View workflow logs in GitHub Actions
- Verify configured secrets
- Test SSH connection manually

**Common causes:**
- Incorrect or missing secret
- SSH key without permissions
- Firewall blocking SSH
- Docker volume or network doesn't exist

#### 3. Nginx 502 Bad Gateway

**Symptom:** 502 error when accessing via domain

**Diagnosis:**
```bash
sudo nginx -t
sudo tail -f /var/log/nginx/error.log
curl http://localhost:PORT  # test direct
```

**Common causes:**
- Backend container not running
- Incorrect port in upstream
- Incorrect container name

#### 4. Out of Memory (OOM)

**Symptom:** Containers killed unexpectedly

**Diagnosis:**
```bash
docker stats
free -h
dmesg | grep -i "out of memory"
```

**Solution:**
- Reduce memory limits per container
- Add swap
- Optimize services (e.g., reduce workers)
---
## 📚 References and Resources

### Official Documentation
- [Docker Documentation](https://docs.docker.com/)
- [GitHub Actions](https://docs.github.com/en/actions)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Prometheus](https://prometheus.io/docs/)
- [Grafana](https://grafana.com/docs/)
- [Portainer](https://docs.portainer.io/)

## 🤝 Contributing

This project follows an SOA architecture with independent repositories:

- **API .NET:** [[repo-link](https://github.com/Team-Avaricia/Micro-Back-Brahiam.git)]
- **Chatbot:** [[repo-link](https://github.com/Team-Avaricia/riwiwallet-sb-service.git)]
- **Frontend Vue.js:** [[repo-link](https://github.com/Team-Avaricia/Frontend.git)] - Vue 3.5 + Vite + TypeScript
- **Documentación:** Wiki.js sincronizado con [[repo-link](https://github.com/Team-Avaricia/Documentaci-n-Riwi-wallet.git)]

To contribute:
1. Fork the specific repository
2. Create branch: `git checkout -b feature/nueva-funcionalidad`
3. Commit: `git commit -m 'feat: descripción'`
4. Push: `git push origin feature/nueva-funcionalidad`
5. Open Pull Request
