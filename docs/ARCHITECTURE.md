# tikter — System Architecture

## Overview

tikter is a multi-tenant SOC & IT Service Management platform built with a React frontend, Express/Node.js backend, and PostgreSQL database.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | React 18, Tailwind CSS, Socket.io-client |
| Backend | Node.js, Express.js, PostgreSQL 16 |
| Authentication | JWT (HttpOnly cookies + Bearer header) |
| Real-time | Socket.io (WebSocket) |
| PWA | Web Push Notifications (VAPID) |
| Deployment | PM2 (cluster), Nginx, Docker |

## System Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   React     │────▶│   Nginx     │────▶│  Express    │
│   Frontend  │     │   (Proxy)   │     │  Backend    │
│   :3000     │     │   :80/443   │     │  :4000      │
└─────────────┘     └─────────────┘     └─────────────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │  PostgreSQL  │
                                        │  Database    │
                                        │  :5432       │
                                        └─────────────┘
```

## Data Flow

### Authentication Flow
1. User submits credentials to `POST /api/auth/login`
2. Backend validates credentials via bcrypt
3. Backend generates access token (15min) + refresh token (7 days)
4. Tokens stored as HttpOnly cookies (`access_token`, `refresh_token`)
5. User data returned in response body, stored in localStorage
6. Frontend AuthContext sets `user` state
7. ProtectedRoute renders dashboard

### Session Validation Flow
1. On page load, AuthContext reads user from localStorage
2. AuthContext immediately sets `user` state (prevents flash redirect)
3. Background call to `GET /api/auth/me` validates session
4. If valid → user state updated with fresh data
5. If invalid → localStorage cleared, user set to null

### API Request Flow
1. Frontend `api.js` sends request with `credentials: 'include'`
2. Browser automatically attaches HttpOnly cookies
3. Backend `authenticate` middleware extracts token from cookies
4. JWT verified, user loaded from database
5. Route handler executes with `req.user` populated

## Security Layers

### CORS
- Strict origin validation with regex patterns
- Allows `localhost:*` for development
- Allows configured `CLIENT_URL` for production
- `credentials: true` for cookie-based auth

### Rate Limiting
- Global rate limiter on all `/api/` routes
- Per-route limiters for auth endpoints
- Brute-force protection on login (15-minute lockout)

### Password Security
- Bcrypt hashing with 12 salt rounds
- Password complexity requirements (min 8 chars, uppercase, lowercase, number, special char)
- Forced password change on first login

### Input Sanitization
- XSS protection via input sanitization middleware
- SQL injection prevention via parameterized queries
- Helmet security headers

## Multi-Tenancy

### Tenant Isolation
- All tables include `tenant_id` foreign key
- Database-level row security via `SET app.current_tenant`
- API middleware enforces tenant context

### Role-Based Access Control
- Roles: `mssp_admin`, `soc_manager`, `soc_analyst`, `client_admin`, `client_employee`
- Department-based access for managers and analysts
- Client module visibility controlled by `enable_clients` flag

## Real-Time Features

### Socket.io Events
- `user-online` / `user-offline` — Online status tracking
- `ticket-updated` — Ticket state changes
- `notification` — Push notifications

### Web Push Notifications
- VAPID keys auto-generated on first run
- Push subscriptions stored in database
- Notification preferences per user

## File Structure

```
tikter/
├── backend/
│   ├── src/
│   │   ├── config/         # DB, JWT, SMTP config
│   │   ├── controllers/    # Route handlers
│   │   ├── middleware/      # Auth, CORS, rate limiting
│   │   ├── models/         # Database schemas
│   │   ├── routes/         # API routes
│   │   ├── services/       # Business logic
│   │   ├── utils/          # Helpers, seed scripts
│   │   └── server.js       # Express app entry
│   └── package.json
├── frontend/
│   ├── public/             # Static assets
│   ├── src/
│   │   ├── components/     # UI components
│   │   ├── contexts/       # Auth, Language, Theme
│   │   ├── hooks/          # Custom React hooks
│   │   ├── pages/          # View components
│   │   ├── services/       # API client, socket
│   │   └── utils/          # Translations, helpers
│   └── package.json
├── database/               # SQL migrations
├── documents/              # Technical documentation
├── deploy/                 # Deployment scripts
├── nginx/                  # Nginx configurations
└── docker-compose.yml      # Docker stack
```
