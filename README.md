# tikter — Enterprise B2B Multi-Tenant Service Desk SaaS

A white-label, multi-tenant ticketing and operations management platform built for Service Providers. Rent it out to IT Managed Services, Software Houses, Infrastructure Ops, Security Teams, or any service organization.

## Platform Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    SUPER ADMIN (Platform Owner)                  │
│  • Tenant account management                                    │
│  • Subscription plans & billing                                 │
│  • Global SaaS configuration                                    │
│  • Cross-tenant analytics                                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
   │   TENANT A   │  │   TENANT B   │  │   TENANT C   │
   │  (Provider)  │  │  (Provider)  │  │  (Provider)  │
   │              │  │              │  │              │
   │ Departments: │  │ Departments: │  │ Departments: │
   │ • Support    │  │ • SOC        │  │ • DevOps     │
   │ • Network    │  │ • IT         │  │ • Backend    │
   │ • Hardware   │  │ • GRC        │  │ • Frontend   │
   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
          │                │                │
   ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐
   │   CLIENTS    │  │   CLIENTS    │  │   CLIENTS    │
   │  (End-Users) │  │  (End-Users) │  │  (End-Users) │
   └──────────────┘  └──────────────┘  └──────────────┘
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, React Router 7, TailwindCSS, Recharts, Socket.io-client |
| Backend | Node.js, Express.js, PostgreSQL 16 |
| Auth | JWT (HttpOnly cookies), bcrypt, TOTP 2FA |
| Real-time | Socket.io (WebSocket) |
| PWA | Web Push Notifications (VAPID), Service Worker |
| Email | Microsoft Graph API + Gmail API (OAuth2) |
| Export | PDFKit, csv-writer |
| Deployment | Docker, Docker Compose, Nginx, GitHub Actions CI/CD |

## Core Features

### Multi-Tenant Architecture
- **Database-level:** PostgreSQL Row-Level Security (RLS) on all tenant-scoped tables
- **Application-level:** Every query scoped by `tenant_id` via session variables
- **UI-level:** Client users see only their organization's data
- **Communication:** Clients see provider responses as "Support Team"; provider sees full identities
- **Quotas:** Dynamic per-tenant limits (users, departments, tickets, emails)
- **Plans:** Basic / Professional / Enterprise / Custom subscription tiers
- **Client module:** Configurable per-tenant via `enable_clients` toggle

### 3-Tier Role System
| Role | Scope | Capabilities |
|------|-------|-------------|
| **Super Admin** | Platform-wide | Tenant CRUD, subscription management, global analytics |
| **Tenant Manager** | Own tenant | Department management, employee oversight, ticket routing |
| **Department Manager** | Assigned departments | Team management, ticket approval, department analytics |
| **Department Employee** | Assigned department | Ticket handling, internal notes, client communication |
| **Client Admin** | Own organization | User management, ticket creation, report export |
| **Client Employee** | Own organization | Ticket creation, ticket tracking |

### Dual-Portal Design
- **Provider Portal:** Full ticket lifecycle, internal notes, severity management, SLA tracking, KPI dashboards
- **Client Portal:** Ticket submission, reply tracking, quick actions, PDF/CSV export

### Dual-Layer Notifications
- **Layer 1: Socket.io** — Instant (0ms) via WebSocket for active sessions
- **Layer 2: Web Push** — Background/closed delivery via VAPID + OS push service
- PWA desktop installation with native notifications

### Dual Email Integration
- **Microsoft Outlook/Azure** — OAuth2 via Microsoft Graph API
- **Google Workspace/Gmail** — OAuth2 via Gmail API
- Per-tenant encrypted credential storage (AES-256-GCM)
- Email invitation workflow with role/department assignment

### Cross-Department Workflow
- Manager assigns ticket to another department
- Destination manager receives approval request
- Approve → ticket transfers | Reject → ticket returns
- Status `pending_approval` with dedicated UI

### Security
- PostgreSQL RLS with `FORCE ROW LEVEL SECURITY`
- JWT with HttpOnly + Secure + SameSite cookies
- Helmet.js (13 headers) + Permissions-Policy
- Rate limiting (5 auth/15min, 100 global/15min)
- Brute-force protection (5 attempts → 15min lockout)
- AES-256-GCM encryption for OAuth credentials
- Audit logging on every significant action
- MFA/2FA with TOTP (Google Authenticator, Authy)

## Quick Start

### Prerequisites
- Node.js 18+
- PostgreSQL 16+ (or Docker)
- npm

### Docker (Recommended)

```bash
git clone https://github.com/Zaidjarwan02/tikter.git
cd tikter
docker-compose up -d
```

### Manual Setup

```bash
git clone https://github.com/Zaidjarwan02/tikter.git
cd tikter

# Backend
cd backend && npm install && cp .env.example .env
# Edit .env with your database credentials
npm run dev

# Frontend (new terminal)
cd frontend && npm install && npm start
```

### Default Admin
- Email: Configured via `SEED_ADMIN_EMAIL` env var (e.g. `admin@tikter.local`)
- Password: Set via `SEED_ADMIN_PASSWORD` env var (required, no default)

## Project Structure

```
tikter/
├── backend/
│   ├── src/
│   │   ├── config/           # Database, JWT, SMTP, OAuth config
│   │   ├── controllers/      # Route handlers
│   │   ├── middleware/        # Auth, CORS, rate limiting, tenant isolation
│   │   ├── routes/           # API route definitions
│   │   ├── services/         # Business logic (email, push, reports)
│   │   ├── utils/            # Helpers, seed scripts
│   │   └── server.js         # Express + Socket.io entry point
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
└── .github/workflows/        # CI/CD pipeline
```

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture](documents/ARCHITECTURE.md) | 3-tier multi-tenant model, data flow, security layers |
| [API Reference](documents/API_DOCS.md) | Complete REST API documentation |
| [Deployment Guide](documents/DEPLOYMENT.md) | Docker, Nginx, CI/CD pipeline |
| [User Roles](documents/USER_ROLES_PERMISSIONS.md) | RBAC matrix and permission details |
| [Environment Variables](documents/ENV_VARIABLES.md) | Configuration reference |

## License

Private repository. All rights reserved.
