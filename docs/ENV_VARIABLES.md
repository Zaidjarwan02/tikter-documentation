# tikter — Environment Variables Reference

## Backend `.env`

### Server Configuration

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `PORT` | `4000` | No | Backend server port |
| `NODE_ENV` | `development` | No | Environment mode (`development` / `production`) |

### Database (PostgreSQL)

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `DB_HOST` | `localhost` | Yes | PostgreSQL host |
| `DB_PORT` | `5432` | No | PostgreSQL port |
| `DB_NAME` | `mssp_ticketing` | Yes | Database name |
| `DB_USER` | `postgres` | Yes | Database user |
| `DB_PASSWORD` | `postgres` | Yes | Database password |

### JWT Authentication

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `JWT_SECRET` | — | **Yes** | Access token signing secret (64+ chars) |
| `JWT_REFRESH_SECRET` | — | **Yes** | Refresh token signing secret (64+ chars) |
| `JWT_EXPIRES_IN` | `15m` | No | Access token expiration |

### Encryption Keys

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `OAUTH_ENCRYPTION_KEY` | — | No | AES-256-GCM key for OAuth credentials |
| `ENCRYPTION_KEY` | — | No | General-purpose encryption key |

Generate keys:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### SMTP Email

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `SMTP_HOST` | `smtp.gmail.com` | No | SMTP server host |
| `SMTP_PORT` | `587` | No | SMTP server port |
| `SMTP_USER` | — | Yes | SMTP username/email |
| `SMTP_PASS` | — | Yes | SMTP password/app password |
| `SOC_EMAIL` | `soc@company.com` | No | Sender email address |

### CORS & Frontend URL

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `CLIENT_URL` | `http://localhost:3000` | Yes | Frontend URL for CORS + email links |
| `FRONTEND_URL` | `http://localhost:3000` | Yes | Alias for CLIENT_URL |

### Web Push (VAPID)

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `VAPID_PUBLIC_KEY` | — | No | VAPID public key for Web Push |
| `VAPID_PRIVATE_KEY` | — | No | VAPID private key for Web Push |
| `VAPID_SUBJECT` | `mailto:support@tikter.com` | No | VAPID contact email |

Generate VAPID keys:
```bash
npx web-push generate-vapid-keys
```

### Seed Script

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `SEED_ADMIN_EMAIL` | `your_admin_email` | No | Admin email for seed script |
| `SEED_ADMIN_PASSWORD` | `your_secure_password` | No | Admin password for seed script |

---

## Frontend `.env`

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `REACT_APP_API_URL` | Auto-detected | No | Backend API URL (e.g., `https://yourdomain.com/api`) |
| `GENERATE_SOURCEMAP` | `false` | No | Generate source maps for debugging |

### Auto-Detection

If `REACT_APP_API_URL` is empty, the frontend auto-detects:
```javascript
const API_URL = process.env.REACT_APP_API_URL || `http://${window.location.hostname}:4000/api`;
```

---

## Docker Compose `.env`

For Docker deployments, set variables in `docker-compose.yml`:

```yaml
environment:
  - NODE_ENV=production
  - DB_HOST=postgres
  - DB_PORT=5432
  - DB_NAME=mssp_ticketing
  - DB_USER=postgres
  - DB_PASSWORD=your_secure_password
  - JWT_SECRET=your_64_char_random_string
  - JWT_REFRESH_SECRET=another_64_char_random_string
  - CLIENT_URL=https://yourdomain.com
```

---

## Production Checklist

- [ ] `JWT_SECRET` generated and set
- [ ] `JWT_REFRESH_SECRET` generated and set
- [ ] `DB_PASSWORD` changed from default
- [ ] `CLIENT_URL` set to production domain
- [ ] `SMTP_USER` and `SMTP_PASS` configured
- [ ] `NODE_ENV=production`
- [ ] `VAPID_PUBLIC_KEY` and `VAPID_PRIVATE_KEY` generated
