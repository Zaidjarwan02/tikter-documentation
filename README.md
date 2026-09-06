# tikter — Documentation

<p align="center">
  <strong>Enterprise B2B Multi-Tenant Service Desk SaaS Platform</strong>
</p>

<p align="center">
  <a href="https://github.com/Zaidjarwan02/tikter">Source Code</a> •
  <a href="https://github.com/Zaidjarwan02/tikter-documentation">Documentation</a>
</p>

---

## Overview

**tikter** is an enterprise-grade B2B Multi-Tenant Service Desk & Operations Management SaaS platform. It is designed to be rented out by Service Providers (IT Managed Services, Software Houses, Infrastructure Ops, Security Teams) to their end-clients.

### Key Features

- **Multi-Tenant Architecture** — PostgreSQL Row-Level Security (RLS) for complete data isolation
- **3-Tier Model** — Platform Owner → Tenants (Providers) → Clients (End-Users)
- **Dual-Portal Design** — Provider internal dashboard + Client self-service portal
- **Communication Bridge** — Visibility-controlled messaging (internal notes vs. external replies)
- **Dual-Layer Notifications** — Socket.io (0ms) + Web Push (background/closed) via VAPID
- **PWA Support** — Desktop installation, offline caching, native OS notifications
- **Dual Email Integration** — Microsoft Outlook (Graph API) + Google Workspace (Gmail API)
- **Cross-Department Workflow** — Assign tickets between departments with manager approval
- **RBAC** — 5 role types with granular permissions
- **2FA** — TOTP-based (Google Authenticator, Authy)
- **Audit Logging** — Every action tracked with old/new values, IP, user agent
- **SLA Tracking** — Breach monitoring with severity-based deadlines
- **Subscription Plans** — Basic / Professional / Enterprise / Custom tiers
- **Dynamic Quotas** — Per-tenant limits (users, departments, tickets, email)

---

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/ARCHITECTURE.md) | 3-tier multi-tenant model, data flow, security layers |
| [API Reference](docs/API_DOCS.md) | Complete REST API documentation (124 endpoints) |
| [Deployment Guide](docs/DEPLOYMENT.md) | Docker, Nginx, CI/CD pipeline |
| [User Roles & Permissions](docs/USER_ROLES_PERMISSIONS.md) | RBAC matrix and permission details |
| [Environment Variables](docs/ENV_VARIABLES.md) | Configuration reference |
| [Full Documentation](docs/DOCUMENTATION.md) | Comprehensive all-in-one technical doc |
| [Security Audit](docs/security/Security_Audit_2026-08-31.md) | Security assessment report |
| [CVE Audit](docs/security/Fresh_CVE_Audit_2026-08-25.md) | Dependency vulnerability audit |

---

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

---

## Quick Start

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



## License

MIT License

---

Developed by [Zaid Jarwan](https://www.linkedin.com/in/zaid-jarwan/)
