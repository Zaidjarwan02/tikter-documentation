# tikter — Deployment Guide

## Overview

tikter supports three deployment methods:
1. **Docker Compose** (recommended for production)
2. **PM2 + Nginx** (Ubuntu/Debian servers)
3. **Development** (local setup)

---

## 1. Docker Deployment (Recommended)

### Prerequisites
- Docker Engine 20.10+
- Docker Compose v2+
- `credentials.json` (OAuth) placed in project root

### Quick Start

```bash
# Clone the repository
git clone https://github.com/Zaidjarwan02/tikter.git
cd tikter

# Create backend .env
cp backend/.env.example backend/.env

# Generate secrets
openssl rand -hex 64  # JWT_SECRET
openssl rand -hex 64  # JWT_REFRESH_SECRET
openssl rand -hex 32  # ENCRYPTION_KEY
openssl rand -hex 32  # OAUTH_ENCRYPTION_KEY

# Edit backend/.env with production values
nano backend/.env

# Build and start all services
docker compose up -d --build

# Verify services
docker compose ps
docker compose logs -f
```

### First-Time Setup

```bash
# Run database migrations
docker compose exec backend node src/utils/initDb.js

# Seed admin user
docker compose exec backend node src/utils/seed.js
```

### Service Architecture

| Service | Container | Port | Description |
|---------|-----------|------|-------------|
| Backend | tikter_backend | 5000 | Node.js API server |
| Frontend | tikter_frontend | 80/443 | Nginx reverse proxy + SPA |

- Frontend (Nginx) serves the React SPA and proxies `/api/` and `/socket.io/` to the backend
- Backend connects to PostgreSQL (host machine or separate container)
- OAuth credentials mounted read-only from `./credentials.json`
- Backend logs persisted in named volume `backend_logs`

### Production Commands

```bash
# Restart services
docker compose restart

# Rebuild after code changes
docker compose up -d --build

# View logs
docker compose logs --tail=100 -f

# Stop all services
docker compose down

# Stop and remove volumes (data loss!)
docker compose down -v

# Enter container shell
docker compose exec backend sh
```

### SSL with Docker

Place SSL certificates in `./nginx/certs/` and update `frontend/nginx.conf`:

```nginx
server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ...
}
```

Then restart:
```bash
docker compose up -d --build
```

---

## 2. PM2 + Nginx Deployment (Ubuntu)

### Prerequisites
- Ubuntu 22.04+ / Debian 12+
- Node.js 20+, npm
- PostgreSQL 16
- Nginx
- PM2

### Server Setup

```bash
# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install PM2
sudo npm install -g pm2
pm2 startup

# Install PostgreSQL 16
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install -y postgresql-16

# Install Nginx
sudo apt-get install -y nginx
sudo ufw allow 'Nginx Full'
```

### Database Setup

```bash
sudo -i -u postgres
psql -c "CREATE DATABASE mssp_ticketing;"
psql -c "ALTER USER postgres PASSWORD 'your_secure_password';"
exit

# Run migrations
cd /opt/tikter/database
for f in migration_*.sql; do
  psql -U postgres -d mssp_ticketing -f "$f"
done
```

### Application Setup

```bash
cd /opt
sudo git clone https://github.com/Zaidjarwan02/tikter.git
cd tikter

# Backend
cd backend
npm install
cp .env.example .env
nano .env  # Set production values

# Frontend
cd ../frontend
npm install
echo "REACT_APP_API_URL=https://yourdomain.com/api" > .env
npm run build
```

### PM2 Configuration

```bash
cd /opt/tikter
pm2 start ecosystem.config.js
pm2 save
pm2 status
pm2 logs tikter-backend
```

### Nginx Configuration

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    root /opt/tikter/frontend/build;
    index index.html;

    location /api/ {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /socket.io/ {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }

    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### SSL Certificate

```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
sudo certbot renew --dry-run
```

---

## 3. Development Setup

### Prerequisites
- Node.js 18+
- PostgreSQL 16+ (or Docker)
- npm

### Quick Start

```bash
git clone https://github.com/Zaidjarwan02/tikter.git
cd tikter

# Option A: Setup script
bash setup.sh        # Linux/Mac
# or
setup.bat            # Windows

# Option B: Docker
docker-compose up

# Option C: Manual
cd backend && npm install && npm run dev    # Terminal 1
cd frontend && npm install && npm start     # Terminal 2
```

### Access

| Service | URL |
|---------|-----|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:5000/api |

---

## 4. CI/CD Pipeline (GitHub Actions)

The project includes a GitHub Actions workflow (`.github/workflows/ci-cd.yml`) that:

1. **Triggers** on push to `main` branch
2. **Builds** Docker images for backend and frontend
3. **Pushes** images to GitHub Container Registry (GHCR)
4. **Verifies** images are accessible

### Pipeline Structure

```yaml
Jobs:
  build-backend:
    - Checkout code
    - Setup Docker Buildx
    - Login to GHCR
    - Build and push backend image
  build-frontend:
    - Checkout code
    - Setup Docker Buildx
    - Login to GHCR
    - Build and push frontend image
  verify:
    needs: [build-backend, build-frontend]
    - Pull and verify both images
```

### Image Registry

| Image | Registry |
|-------|----------|
| Backend | `ghcr.io/zaidjarwan02/tikter-backend:latest` |
| Frontend | `ghcr.io/zaidjarwan02/tikter-frontend:latest` |

---

## 5. Environment Variables

### Required

| Variable | Description | How to generate |
|----------|-------------|-----------------|
| `JWT_SECRET` | Access token signing key | `node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"` |
| `JWT_REFRESH_SECRET` | Refresh token signing key | Same command, different value |
| `ENCRYPTION_KEY` | AES-256-GCM key | `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `OAUTH_ENCRYPTION_KEY` | OAuth credential encryption | Same command, different value |
| `DB_PASSWORD` | PostgreSQL password | Use a strong password |

### Optional (have defaults)

| Variable | Default | Description |
|----------|---------|-------------|
| `NODE_ENV` | `production` | Environment mode |
| `PORT` | `5000` | Backend listening port |
| `DB_HOST` | `localhost` | Database host |
| `DB_PORT` | `5432` | Database port |
| `DB_NAME` | `mssp_ticketing` | Database name |
| `CLIENT_URL` | `http://localhost:3000` | Frontend URL for CORS |

---

## 6. Backup Strategy

### Database Backup

```bash
# Manual backup
pg_dump -U postgres mssp_ticketing > backup_$(date +%Y%m%d).sql

# Automated daily backup (crontab)
0 2 * * * pg_dump -U postgres mssp_ticketing | gzip > /backups/tikter_$(date +\%Y\%m\%d).sql.gz
```

### Restore

```bash
psql -U postgres mssp_ticketing < backup_20260906.sql
```

---

## 7. Maintenance Commands

```bash
# PM2
pm2 status                    # Show all processes
pm2 logs tikter-backend       # View backend logs
pm2 monit                     # Real-time monitoring
pm2 restart tikter-backend    # Restart backend
pm2 save                      # Save process list

# Docker
docker compose up -d          # Start all services
docker compose down           # Stop all services
docker compose logs -f        # Follow logs
docker compose restart backend # Restart backend only

# Database
node src/utils/seed.js        # Reset & seed admin user
node src/utils/initDb.js      # Initialize database schema
```

---

## 8. Troubleshooting

| Issue | Solution |
|-------|----------|
| Backend won't start | Check `pm2 logs tikter-backend` or `docker compose logs backend` |
| Database connection refused | Verify PostgreSQL: `pg_isready` or `docker compose ps` |
| CORS errors | Ensure `CLIENT_URL` matches your domain |
| Cookie not set | Verify HTTPS and `secure` flag configuration |
| Port already in use | `netstat -ano | findstr :5000` and kill the process |
| Frontend build fails | Clear `node_modules` and reinstall: `rm -rf node_modules && npm install` |
