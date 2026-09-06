# tikter — Production Deployment Guide

## Quick Start

### Option A: Docker Compose (Recommended)
```bash
# 1. Clone and configure
cp .env.example .env
# Edit .env with real secrets

# 2. Build and start
docker compose up -d --build

# 3. Seed database
docker compose exec backend node src/utils/seed.js

# 4. Check status
docker compose ps
docker compose logs -f
```

### Option B: PM2 + Nginx (Ubuntu)
```bash
# 1. Run setup script
sudo chmod +x deploy/setup-ubuntu.sh
sudo ./deploy/setup-ubuntu.sh

# 2. Configure
sudo nano /opt/tikter/backend/.env

# 3. Seed database
cd /opt/tikter/backend && node src/utils/seed.js

# 4. Setup SSL
sudo certbot --nginx -d yourdomain.com
```

### Option C: PM2 (Windows)
```powershell
# 1. Run setup
.\deploy\setup-windows.ps1

# 2. Configure
notepad C:\tikter\backend\.env

# 3. Seed database
cd C:\tikter\backend; node src\utils\seed.js
```

## Environment Variables

### Required (must set before deployment)
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
| `DB_NAME` | `tikter_db` | Database name |
| `FRONTEND_URL` | `http://localhost:3000` | Frontend URL for CORS |
| `CORS_ORIGIN` | `http://localhost:3000` | Allowed CORS origin |

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Browser   │────▶│  Nginx:443   │────▶│ Backend:5000 │
│             │     │  (HTTPS)     │     │  (Node.js)   │
└─────────────┘     │              │     └──────┬───────┘
                    │  / → static  │            │
                    │  /api → proxy│     ┌──────▼───────┐
                    │  /ws → proxy │     │ PostgreSQL   │
                    └──────────────┘     │    :5432     │
                                         └──────────────┘
```

## Commands Reference

```bash
# PM2
pm2 status                    # Show all processes
pm2 logs tikter-backend       # View backend logs
pm2 monit                     # Real-time monitoring
pm2 restart tikter-backend    # Restart backend
pm2 stop tikter-backend       # Stop backend
pm2 delete tikter-backend     # Remove from PM2
pm2 save                      # Save current process list
pm2 startup                   # Enable auto-start on boot

# Database
node src/utils/seed.js        # Reset & seed admin user
node src/utils/initDb.js      # Initialize database schema

# Docker
docker compose up -d          # Start all services
docker compose down           # Stop all services
docker compose logs -f        # Follow logs
docker compose exec backend sh  # Shell into backend
docker compose restart backend  # Restart backend only
```

## SSL / HTTPS

### Let's Encrypt (Ubuntu)
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
sudo certbot renew --dry-run  # Test renewal
```

### Self-signed (Development)
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/selfsigned.key \
  -out nginx/selfsigned.crt
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Backend won't start | Check `pm2 logs tikter-backend` for errors |
| Database connection refused | Verify PostgreSQL is running: `pg_isready` |
| CORS errors | Ensure `FRONTEND_URL` matches your domain |
| Cookie not set | Verify `COOKIE_SECURE=true` and using HTTPS |
| Port already in use | `netstat -ano | findstr :5000` and kill the process |
