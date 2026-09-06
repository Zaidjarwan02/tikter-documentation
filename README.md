# tikter — MSSP SOC Ticketing & Incident Management System

<p align="center">
  <strong>A multi-tenant ticketing platform for Managed Security Service Providers (MSSPs)</strong>
</p>

<p align="center">
  <a href="https://github.com/Zaidjarwan02/tikter">Source Code</a> •
  <a href="https://github.com/Zaidjarwan02/tikter-documentation">Documentation</a>
</p>

---

## Overview

**tikter** is a comprehensive SOC (Security Operations Center) ticketing and incident management system designed for MSSPs. It bridges internal SOC analyst communication with external client responses while maintaining strict multi-tenant data isolation and accountability.

### Key Features

- **Multi-Tenant Architecture** — PostgreSQL Row-Level Security (RLS) for complete data isolation
- **Dual-Portal Design** — SOC internal dashboard + Client self-service portal
- **Communication Bridge** — Visibility-controlled messaging (internal notes vs. external replies)
- **Dual-Layer Notifications** — Socket.io (0ms) + Web Push (background/closed) via VAPID
- **PWA Support** — Desktop installation, offline caching, native OS notifications
- **Dual Email Integration** — Microsoft Outlook (Graph API) + Google Workspace (Gmail API)
- **Cross-Department Workflow** — Assign tickets between departments with manager approval
- **RBAC** — 5 role types with granular permissions
- **2FA** — TOTP-based (Google Authenticator, Authy)
- **Audit Logging** — Every action tracked with old/new values, IP, user agent
- **SLA Tracking** — Breach monitoring with severity-based deadlines
- **PDF/CSV Export** — Report generation by date range
- **Enable/Disable Client Module** — Per-tenant configurable

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 18, React Router 7, TailwindCSS, Recharts, Socket.io-client |
| **Backend** | Node.js, Express.js, PostgreSQL 16 |
| **Auth** | JWT (HttpOnly cookies), bcrypt, TOTP 2FA |
| **Real-time** | Socket.io (WebSocket) |
| **PWA** | Web Push Notifications (VAPID), Service Worker |
| **Email** | Microsoft Graph API + Gmail API (OAuth) |
| **Export** | PDFKit, csv-writer |
| **Deployment** | Docker, PM2, Nginx |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      CLIENT PORTAL                          │
│  View tickets, reply, quick actions                         │
│  Dashboard with org-specific metrics                        │
│  Export reports (PDF/CSV)                                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
              ┌────────▼────────┐
              │  COMMUNICATION  │
              │     BRIDGE      │
              │  external       │◄── Client sees this
              │  internal       │◄── SOC-only (hidden from client)
              └────────┬────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                     SOC INTERNAL                            │
│  Full analyst identity on every action                      │
│  Internal notes for pre-reply discussion                    │
│  Severity management with audit trail                       │
│  SLA tracking & analyst KPI dashboards                      │
│  Cross-department approval workflow                         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  DUAL-LAYER NOTIFICATIONS                    │
│  Layer 1: Socket.io — instant (0ms) via WebSocket           │
│  Layer 2: Web Push — background/closed via VAPID/OS         │
└─────────────────────────────────────────────────────────────┘
```

---

## Documentation

| Document | Description |
|----------|-------------|
| [Full Documentation](docs/DOCUMENTATION.md) | Complete technical documentation (all-in-one) |
| [System Architecture](docs/ARCHITECTURE.md) | Technical architecture, data flow, security layers |
| [API Reference](docs/API_DOCS.md) | Complete REST API documentation with examples |
| [User Roles & Permissions](docs/USER_ROLES_PERMISSIONS.md) | RBAC matrix, department access, invitation workflow |
| [Environment Variables](docs/ENV_VARIABLES.md) | Backend & frontend configuration reference |
| [Deployment Guide](docs/DEPLOYMENT_GUIDE.md) | Docker, PM2+Nginx, and production setup |
| [Quick Deploy](docs/DEPLOY.md) | Fast deployment commands reference |
| [Security Audit](docs/security/Security_Audit_2026-08-31.md) | Penetration test results and CVE audit |
| [CVE Audit](docs/security/Fresh_CVE_Audit_2026-08-25.md) | Dependency vulnerability assessment |

---

## User Roles

| Role | Description | Redirect |
|------|-------------|----------|
| **MSSP Admin** | Cross-tenant system management | `/admin` |
| **Tenant Admin** | Full tenant management (no dept) | `/manager` |
| **Dept Manager** | Department-scoped management | `/department/manager` |
| **SOC Analyst** | Ticket handling & response | `/department/employee` |
| **Client Admin** | Client organization management | `/client` |
| **Client Employee** | Basic ticket submission | `/client` |

---

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
cd backend && npm install && npm run dev

# Frontend (new terminal)
cd frontend && npm install && npm start
```

---

## Security

- **Database-level:** PostgreSQL Row-Level Security (RLS) on all tenant-scoped tables
- **Application-level:** Every query scoped by `tenant_id` via session variables
- **Authentication:** JWT with HttpOnly + Secure + SameSite cookies
- **Encryption:** AES-256-CBC for OAuth credentials, VAPID key encryption
- **Headers:** Helmet.js (13 headers) + Permissions-Policy
- **Rate Limiting:** 5 auth/15min, 100 global/15min
- **Brute-force Protection:** 5 failed attempts → 15min lockout
- **Audit Trail:** Every action logged with old/new values, IP, user agent

---

## License

Private repository. All rights reserved.

---

<p align="center">
  Developed by <a href="https://www.linkedin.com/in/zaid-jarwan/">Zaid Jarwan</a>
</p>
