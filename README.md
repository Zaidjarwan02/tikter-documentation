# tikter — Enterprise B2B Multi-Tenant Service Desk SaaS

A white-label, multi-tenant ticketing and operations management platform built for Service Providers. Rent it out to IT Managed Services, Software Houses, Infrastructure Ops, Security Teams, or any service organization.

## What Is tikter?

tikter is a **software-as-a-service platform** that a service provider operates and sells to its own clients. Each client (tenant) gets an isolated workspace with their own departments, staff, and end-users, while a platform Super Admin manages all tenants and subscriptions.

A single deployment gives you:

- **Service Desk & Ticketing** — full ticket lifecycle, severities, SLAs, approvals, internal notes
- **Multi-Tenant Isolation** — strict data separation between every client at the database level
- **Rich Role System** — Super Admin / Tenant Manager / Department Manager / Agent / Client Admin / Client User
- **Client Portal** — end-clients submit and track tickets; provider staff work internal tickets
- **Real-Time** — instant updates via WebSocket and background delivery via Web Push (PWA)
- **Email Integration** — per-tenant Microsoft Outlook or Google Workspace integration
- **Reports & Export** — dashboards, KPIs, PDF/CSV export

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
   │  Departments │  │  Departments │  │  Departments │
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
| Frontend | React, React Router, TailwindCSS, Recharts, Socket.io-client |
| Backend | Node.js, Express.js, PostgreSQL |
| Auth | JWT (HttpOnly cookies), bcrypt, TOTP 2FA |
| Real-time | Socket.io (WebSocket) |
| PWA | Web Push Notifications (VAPID), Service Worker |
| Email | Microsoft Graph API + Gmail API (OAuth2) |
| Deployment | Docker, Docker Compose, Nginx, GitHub Actions CI/CD |

## Role System

| Role | Scope | Capabilities |
|------|-------|-------------|
| **Super Admin** | Platform-wide | Tenant CRUD, subscription management, global analytics |
| **Tenant Manager** | Own tenant | Department management, employee oversight, ticket routing |
| **Department Manager** | Assigned departments | Team management, ticket approval, department analytics |
| **Department Employee** | Assigned department | Ticket handling, internal notes, client communication |
| **Client Admin** | Own organization | User management, ticket creation, report export |
| **Client Employee** | Own organization | Ticket creation, ticket tracking |

## Security

- Multi-tenant isolation via PostgreSQL Row-Level Security
- JWT with HttpOnly + Secure + SameSite cookies
- Hardened HTTP security headers (CSP, Permissions-Policy, HSTS, COOP/COEP, CORP)
- Rate limiting and brute-force protection
- Encrypted per-tenant email credentials
- Audit logging on significant actions
- MFA/2FA with TOTP

## Quick Start

```bash
git clone https://github.com/Zaidjarwan02/tikter.git
cd tikter
docker compose up -d --build
```

## License

Built by [ZaidJarwan](https://www.linkedin.com/in/zaidjarwan) — Private repository. All rights reserved.