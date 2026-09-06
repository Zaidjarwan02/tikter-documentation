# tikter — Deployment Guide

## Prerequisites

- Ubuntu 22.04+ / Debian 12+ server
- Node.js 20+ and npm
- PostgreSQL 16
- Nginx
- PM2 process manager
- Domain name with DNS configured

## 1. Server Setup

### Install Node.js 20

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v  # Should show v20.x.x
```

### Install PM2

```bash
sudo npm install -g pm2
pm2 startup
```

### Install PostgreSQL 16

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install -y postgresql-16
```

### Install Nginx

```bash
sudo apt-get install -y nginx
sudo ufw allow 'Nginx Full'
```

## 2. Database Setup

```bash
# Switch to postgres user
sudo -i -u postgres

# Create database
psql -c "CREATE DATABASE mssp_ticketing;"
psql -c "ALTER USER postgres PASSWORD 'your_secure_password';"

# Exit postgres user
exit
```

### Run Migrations

```bash
cd /opt/tikter/database
for f in migration_*.sql; do
  psql -U postgres -d mssp_ticketing -f "$f"
done
```

## 3. Application Setup

### Clone Repository

```bash
cd /opt
sudo git clone https://github.com/your-repo/tikter.git
cd tikter
```

### Backend Setup

```bash
cd backend
npm install

# Create .env from template
cp .env.example .env

# Edit with production values
nano .env
```

**Required .env values:**
```bash
NODE_ENV=production
PORT=4000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mssp_ticketing
DB_USER=postgres
DB_PASSWORD=your_secure_password
JWT_SECRET=your_64_char_random_string
JWT_REFRESH_SECRET=another_64_char_random_string
CLIENT_URL=https://yourdomain.com
FRONTEND_URL=https://yourdomain.com
```

### Frontend Setup

```bash
cd ../frontend
npm install

# Create .env
echo "REACT_APP_API_URL=https://yourdomain.com/api" > .env

# Build for production
npm run build
```

## 4. PM2 Configuration

### Start Backend

```bash
cd /opt/tikter
pm2 start ecosystem.config.js
pm2 save
```

### Verify PM2 Status

```bash
pm2 status
pm2 logs tikter-backend
```

## 5. Nginx Configuration

### Create Nginx Config

```bash
sudo nano /etc/nginx/sites-available/tikter
```

**Configuration:**
```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    # SSL (configured by Certbot)
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Frontend static files
    root /opt/tikter/frontend/build;
    index index.html;

    # API proxy
    location /api/ {
        proxy_pass http://localhost:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Socket.io proxy
    location /socket.io/ {
        proxy_pass http://localhost:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Static assets
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Enable Site

```bash
sudo ln -s /etc/nginx/sites-available/tikter /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

## 6. SSL Certificate

```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
sudo certbot renew --dry-run
```

## 7. Seed Admin User

```bash
cd /opt/tikter/backend
node src/utils/seed.js
```

**Default credentials (set via `.env`):**
- Email: `your_admin_email`
- Password: `your_secure_password`

## 8. Verify Deployment

```bash
# Check PM2 status
pm2 status

# Check Nginx status
sudo systemctl status nginx

# Test API
curl -k https://yourdomain.com/api/health

# Test frontend
curl -k https://yourdomain.com
```

## 9. Maintenance Commands

```bash
# Restart backend
pm2 restart tikter-backend

# View logs
pm2 logs tikter-backend

# Monitor
pm2 monit

# Rebuild frontend
cd /opt/tikter/frontend && npm run build

# Update application
cd /opt/tikter && git pull
cd backend && npm install
cd ../frontend && npm install && npm run build
pm2 restart tikter-backend
```

## 10. Backup Strategy

### Database Backup

```bash
# Manual backup
pg_dump -U postgres mssp_ticketing > backup_$(date +%Y%m%d).sql

# Automated daily backup (add to crontab)
0 2 * * * pg_dump -U postgres mssp_ticketing | gzip > /backups/tikter_$(date +\%Y\%m\%d).sql.gz
```

### Restore Backup

```bash
psql -U postgres mssp_ticketing < backup_20260903.sql
```

---

## Docker Deployment (Recommended)

### Prerequisites

- Docker Engine 20.10+
- Docker Compose v2+
- `credentials.json` (OAuth) placed in project root

### 1. Prepare Environment

```bash
cd /opt/tikter

# Create backend .env (or copy from template)
cp backend/.env.example backend/.env

# Generate secrets
openssl rand -hex 64  # Use for JWT_SECRET
openssl rand -hex 64  # Use for JWT_REFRESH_SECRET
openssl rand -hex 32  # Use for ENCRYPTION_KEY
openssl rand -hex 32  # Use for OAUTH_ENCRYPTION_KEY

# Edit backend/.env with production values
nano backend/.env
```

**Required `.env` variables:**
```bash
DB_HOST=db
DB_PORT=5432
DB_NAME=tikter_db
DB_USER=tikter_app
DB_PASSWORD=<strong_password>
JWT_SECRET=<128_char_hex>
JWT_REFRESH_SECRET=<128_char_hex>
ENCRYPTION_KEY=<64_char_hex>
OAUTH_ENCRYPTION_KEY=<64_char_hex>
```

### 2. Build & Start

```bash
# Build all images and start services
docker compose up -d --build

# Verify services are running
docker compose ps

# Check logs
docker compose logs -f
docker compose logs -f backend
docker compose logs -f frontend
```

### 3. Database Initialization (First Time)

```bash
# Run migrations
docker compose exec backend node src/utils/initDb.js

# Seed admin user
docker compose exec backend node src/utils/seed.js
```

**Default admin credentials (set via `.env`):**
- Email: `your_admin_email`
- Password: `your_secure_password`

### 4. Verify Deployment

```bash
# Health check
curl http://localhost/api/health

# Test frontend
curl -I http://localhost

# Check container status
docker compose ps
```

### 5. Production Commands

```bash
# Restart services
docker compose restart

# Rebuild after code changes
docker compose up -d --build

# View logs (last 100 lines)
docker compose logs --tail=100 -f

# Stop all services
docker compose down

# Stop and remove volumes (data loss!)
docker compose down -v

# Enter container shell
docker compose exec backend sh
docker compose exec frontend sh
```

### Architecture

| Service    | Container        | Port  | Description                    |
|------------|------------------|-------|--------------------------------|
| Backend    | tikter_backend   | 5000  | Node.js API server             |
| Frontend   | tikter_frontend  | 80/443| Nginx reverse proxy + SPA      |

- **Frontend (Nginx)** serves the React SPA and proxies `/api/` and `/socket.io/` to the backend container
- **Backend** connects to PostgreSQL (host machine or separate container)
- OAuth credentials are mounted read-only from `./credentials.json`
- Backend logs are persisted in a named volume `backend_logs`

### SSL with Docker (Production)

For HTTPS, place your SSL certificates in `./nginx/certs/` and update `frontend/nginx.conf`:

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
