# tikter — System Architecture

## Overview

tikter is an enterprise-grade B2B Multi-Tenant Service Desk & Operations Management SaaS platform. It is designed to be rented out by Service Providers (IT Managed Services, Software Houses, Infrastructure Ops, Support Teams) to their end-clients.

## 3-Tier Multi-Tenant Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    TIER 1: PLATFORM OWNER                       │
│                                                                 │
│  Super Admin (super_admin)                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ • Tenant account CRUD                                    │    │
│  │ • Subscription plan management (Basic/Pro/Enterprise)    │    │
│  │ • Dynamic quota limits (users, depts, tickets, email)   │    │
│  │ • Cross-tenant analytics dashboard                       │    │
│  │ • User password/2FA reset, force logout                  │    │
│  │ • Tenant status management (active/suspended/inactive)   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  System Admin tenant: platform-admin                            │
│  Default admin: zjrwan6@gmail.com                               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
┌─────────▼─────────┐ ┌───▼───────────┐ ┌──▼──────────────┐
│  TIER 2: TENANT A  │ │ TIER 2: B    │ │ TIER 2: C       │
│  (Service Provider)│ │ (Provider)   │ │ (Provider)      │
│                    │ │              │ │                  │
│ Tenant Admin       │ │              │ │                  │
│ (tenant_admin)     │ │              │ │                  │
│ ┌────────────────┐ │ │              │ │                  │
│ │ Departments:   │ │ │              │ │                  │
│ │ • Support      │ │ │              │ │                  │
│ │ • Network      │ │ │              │ │                  │
│ │ • Hardware     │ │ │              │ │                  │
│ │                │ │ │              │ │                  │
│ │ Staff:         │ │ │              │ │                  │
│ │ • Dept Managers│ │ │              │ │                  │
│ │ • Agents       │ │ │              │ │                  │
│ └────────────────┘ │ │              │ │                  │
│                    │ │              │ │                  │
│ Client Module:     │ │              │ │                  │
│ ┌────────────────┐ │ │              │ │                  │
│ │ Client Admin   │ │ │              │ │                  │
│ │ Client Users   │ │ │              │ │                  │
│ └────────────────┘ │ │              │ │                  │
└─────────┬──────────┘ └───┬───────────┘ └──┬──────────────┘
          │                │                │
┌─────────▼─────────┐ ┌───▼───────────┐ ┌──▼──────────────┐
│  TIER 3: CLIENTS   │ │ TIER 3:       │ │ TIER 3:         │
│  (End-Users)       │ │ CLIENTS       │ │ CLIENTS         │
│                    │ │               │ │                 │
│ • Submit tickets   │ │               │ │                 │
│ • Track SLAs       │ │               │ │                 │
│ • View reports     │ │               │ │                 │
│ • Reply to support │ │               │ │                 │
└────────────────────┘ └───────────────┘ └─────────────────┘
```

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | React 18, React Router 7, TailwindCSS | SPA with glassmorphism UI |
| Backend | Node.js, Express.js | REST API + WebSocket server |
| Database | PostgreSQL 16 | Primary datastore with RLS |
| Auth | JWT + bcrypt + TOTP | HttpOnly cookies, 2FA |
| Real-time | Socket.io | WebSocket communication |
| PWA | Web Push (VAPID), Service Worker | Background notifications |
| Email | Microsoft Graph API + Gmail API | Dual OAuth email providers |
| Export | PDFKit, csv-writer | Report generation |
| Deployment | Docker, Nginx, GitHub Actions | Containerized CI/CD |

## System Architecture

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

## Data Flow

### Authentication Flow
```
1. User submits email + password
         │
2. Backend validates via bcrypt compare
         │
3. Backend generates access token (15min) + refresh token (7 days)
   Tokens stored as HttpOnly cookies (access_token, refresh_token)
         │
4. User data returned in response body, stored in localStorage
         │
5. Frontend AuthContext sets user state
         │
6. Background call to GET /api/auth/me validates session
```

### API Request Flow
```
1. Frontend api.js sends request with credentials: 'include'
2. Browser automatically attaches HttpOnly cookies
3. Backend authenticate middleware extracts token from cookies
4. JWT verified, user loaded from database
5. Tenant context set via SET app.current_tenant
6. PostgreSQL RLS automatically filters rows
7. Route handler executes with req.user populated
```

### Tenant Isolation Flow
```
┌─────────────────────────────────────────────────┐
│                 REQUEST INCOMING                 │
└──────────────────────┬──────────────────────────┘
                       │
              ┌────────▼────────┐
              │  JWT Middleware  │
              │  Extract user    │
              │  Set tenant_id   │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  PostgreSQL RLS  │
              │  SET app.current_tenant = '<uuid>'
              │  SET app.current_user_id = '<uuid>'
              │  SET app.current_role = '<role>'
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  Query Execution │
              │  RLS policies    │
              │  filter rows by  │
              │  tenant_id       │
              └─────────────────┘
```

## Security Layers

### Database-Level (PostgreSQL RLS)
```sql
-- Every tenant-scoped table has RLS policies
CREATE POLICY ticket_isolation ON tickets
    FOR ALL
    USING (
        tenant_id = current_setting('app.current_tenant')::UUID
        OR current_setting('app.current_role') = 'super_admin'
    );
```

### Application-Level
- Every API query scoped by `tenant_id` via middleware
- `req.user.tenantId` injected from JWT token
- Service layer enforces role-based permissions
- Internal notes (`visibility: internal`) never sent to client endpoints

### Network-Level
- Strict CORS with origin allowlist (no dev bypass)
- Dynamic CORS for LAN access (private IP ranges only)
- Helmet.js security headers (13 headers including CSP, HSTS)
- Permissions-Policy header (camera, microphone, geolocation disabled)
- Rate limiting on all API endpoints

### Encryption
- AES-256-GCM for OAuth email credentials (`OAUTH_ENCRYPTION_KEY`)
- AES-256-CBC for general encryption (`ENCRYPTION_KEY`)
- VAPID key encryption for Web Push
- Bcrypt (12 rounds) for password hashing

## Role-Based Access Control

| Role | Scope | Default Redirect |
|------|-------|-----------------|
| `super_admin` | Platform-wide | `/admin` |
| `tenant_admin` (no dept) | Own tenant | `/manager` |
| `tenant_admin` (with dept) | Assigned departments | `/department/manager` |
| `department_agent` | Assigned department | `/department/employee` |
| `client_admin` | Own organization | `/client` |
| `client_user` | Own organization | `/client` |

### Permission Matrix

| Feature | Super Admin | Tenant Admin | Dept Manager | Agent | Client Admin | Client User |
|---------|:-----------:|:------------:|:------------:|:-----:|:------------:|:-----------:|
| Tenant Management | Yes | - | - | - | - | - |
| Department Management | Yes | Yes | Own depts | - | - | - |
| Staff Management | Yes | Yes | Own depts | - | Own org | - |
| Create Tickets | Yes | Yes | Yes | Yes | Yes | Yes |
| View All Tickets | All tenants | Own tenant | Own depts | Assigned | Own org | Own only |
| Assign Tickets | Yes | Yes | Own depts | - | Own org | - |
| Internal Notes | Yes | Yes | Yes | Yes | - | - |
| Change Severity | Yes | Yes | Yes | - | - | - |
| Reports | All | Tenant/Dept | Dept | Own stats | Org stats | Own only |
| Client Module | Yes | If enabled | - | - | Yes | Yes |

## Real-Time Features

### Socket.io Events
| Event | Direction | Description |
|-------|-----------|-------------|
| `notification` | Server → Client | New notification received |
| `ticket:created` | Server → Client | New ticket created |
| `ticket:updated` | Server → Client | Ticket status/severity changed |
| `ticket:message` | Server → Client | New message on subscribed ticket |
| `user-online` / `user-offline` | Bidirectional | Online status tracking |

### Web Push Notifications
- VAPID keys auto-generated on first run
- Push subscriptions stored per user per device
- Notification preferences per user (6 types)
- Service Worker handles push reception + notification display
- Click notification → navigate to relevant ticket

## Dynamic OAuth Key Loading

OAuth credentials for email providers are loaded from `D:\credentials.json` via `oauthCredentialsService.js`:

```javascript
// Path configured via GOOGLE_CREDENTIALS_PATH env var
// Fallback: D:\credentials.json
const credentials = oauthCredentialsService.getCredentials();
```

## File Structure

```
tikter/
├── backend/
│   ├── src/
│   │   ├── config/           # DB, JWT, SMTP, OAuth config
│   │   ├── controllers/      # Route handlers
│   │   │   ├── authController.js          # Login, register, password reset
│   │   │   ├── ticketController.js        # Ticket CRUD + messaging
│   │   │   ├── departmentController.js    # Department management
│   │   │   ├── adminTenantController.js   # Super admin tenant ops
│   │   │   ├── tenantController.js        # Tenant CRUD
│   │   │   ├── mfaController.js           # 2FA setup/verify
│   │   │   ├── reportController.js        # PDF/CSV export
│   │   │   ├── emailConfigController.js   # Email provider CRUD
│   │   │   ├── invitationController.js    # Email invitations
│   │   │   └── pushSubscriptionController.js # Push subscriptions
│   │   ├── middleware/        # Auth, CORS, rate limiting, tenant guard
│   │   ├── routes/           # API route definitions
│   │   ├── services/         # Business logic
│   │   │   ├── emailService.js            # SMTP fallback
│   │   │   ├── microsoftGraphService.js   # Microsoft OAuth + Graph API
│   │   │   ├── googleOAuthService.js      # Google OAuth + Gmail API
│   │   │   ├── providerEmailService.js    # Unified provider sender
│   │   │   ├── pushNotificationService.js # VAPID + Web Push
│   │   │   ├── notificationService.js     # Dual-layer engine
│   │   │   └── reportService.js           # PDF/CSV generation
│   │   ├── utils/            # Helpers, seed scripts
│   │   └── server.js         # Express + Socket.io entry
│   ├── Dockerfile
│   └── package.json
├── frontend/
│   ├── public/               # Static assets, PWA manifest, service worker
│   ├── src/
│   │   ├── components/       # Shared layout components
│   │   ├── contexts/         # Auth, Language, Theme providers
│   │   ├── pages/            # Route-level page components
│   │   ├── services/         # API client, socket, push notifications
│   │   └── utils/            # Translations, helpers
│   ├── Dockerfile
│   └── package.json
├── database/                 # SQL migrations (001-024)
├── documents/                # Technical documentation
├── docker-compose.yml        # Container orchestration
├── .github/workflows/        # GitHub Actions CI/CD
└── nginx.conf                # Nginx reverse proxy config
```

## Database Schema

### Core Tables

| Table | Purpose |
|-------|---------|
| `tenants` | Organizations (providers + clients), with plan quotas + `enable_clients` |
| `users` | All users across all tenants |
| `departments` | Tenant-scoped departments with `allow_client_communication` |
| `tickets` | Incidents with severity, status, and type lifecycle |
| `ticket_messages` | Messages with `visibility` flag (internal vs external) |
| `ticket_assignments` | Cross-department approval workflow tracking |
| `ticket_severity_history` | Audit trail for severity changes |
| `audit_logs` | Every action tracked with user, IP, old/new values |
| `notifications` | In-app notifications |
| `vapid_keys` | Encrypted VAPID key storage |
| `push_subscriptions` | Per-user per-device push endpoints |
| `notification_preferences` | Per-user notification type toggles |
| `notification_log` | Delivery audit trail |
| `tenant_email_configs` | Per-tenant encrypted OAuth credentials |
| `invitations` | Pending user invitations with hashed tokens |

### Ticket Types

| Type | Visibility | Description |
|------|-----------|-------------|
| `client_ticket` | External | Client-facing ticket (requires clientTenantId) |
| `internal_dept` | Internal | Cross-department internal ticket |
| `it_helpdesk` | Internal | IT Helpdesk support ticket |

### Ticket Lifecycle

```
Open → Pending Review → In Progress → Resolved → Closed
                  ↘ Pending Approval (cross-department)
```

### Severity Levels

| Level | SLA | Description |
|-------|-----|-------------|
| High | 4h | Critical incidents requiring immediate attention |
| Medium | 24h | Standard incidents requiring timely resolution |
| Low | 72h | Informational / non-urgent requests |
