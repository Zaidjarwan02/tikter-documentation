# tikter — Enterprise B2B Multi-Tenant Service Desk SaaS

## Complete Technical Documentation

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [Technology Stack](#3-technology-stack)
4. [Getting Started](#4-getting-started)
5. [Environment Configuration](#5-environment-configuration)
6. [Database Schema](#6-database-schema)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [Multi-Tenancy & Security](#8-multi-tenancy--security)
9. [API Reference](#9-api-reference)
10. [Frontend Architecture](#10-frontend-architecture)
11. [Real-Time Features](#11-real-time-features)
12. [Ticket Lifecycle](#12-ticket-lifecycle)
13. [User Roles & Permissions](#13-user-roles--permissions)
14. [Deployment](#14-deployment)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. System Overview

tikter is an enterprise-grade B2B Multi-Tenant Service Desk & Operations Management SaaS platform. It is designed to be rented out by Service Providers (IT Managed Services, Software Houses, Infrastructure Ops, Security Teams) to their end-clients, with strict data isolation and accountability.

### Key Capabilities

- **Multi-tenant architecture** with PostgreSQL Row-Level Security (RLS)
- **Dual-portal design**: Provider internal dashboard + Client self-service portal
- **Communication bridge** with visibility controls (internal notes vs. external replies)
- **Dual-layer notifications** — Socket.io (0ms) + Web Push (background/closed) via VAPID
- **PWA desktop installation** — standalone display, offline caching, native OS notifications
- **Dual email integration** — Microsoft Outlook (Graph API) + Google Workspace (Gmail API)
- **Per-tenant email config** — each tenant connects their own Microsoft/Google account
- **System SMTP fallback** — tenants without email config use system email
- **Email invitation workflow** — invite users by email with role/department assignment
- **Cross-department approval workflow** — assign tickets between departments with manager approval
- **RBAC** with 5 role types (System Admin, Tenant Manager, Dept Manager, Dept Employee, Client)
- **2FA support** via TOTP (Google Authenticator / Authy)
- **Audit logging** on every significant action
- **SLA tracking** with breach monitoring
- **PDF/CSV report export**
- **Enable/disable client module** per tenant
- **CI/CD pipeline** — GitHub Actions with SSH deploy and health checks

### Use Cases

| Persona | Primary Use |
|---------|-------------|
| System Admin | Cross-tenant oversight, tenant management, system configuration |
| Tenant Manager | Dashboard analytics, department management, employee oversight |
| Dept Manager | Department tickets, team management, cross-dept approval |
| Dept Employee | Ticket handling, internal notes, client communication |
| Client Admin | User management, ticket creation, report export |
| Client Employee | Ticket creation, ticket tracking |

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        CLIENT PORTAL                             │
│   React SPA  ──────────────────────────────────────────────     │
│   • View tickets, reply, quick actions                           │
│   • Dashboard with org-specific metrics                          │
│   • Export reports (PDF/CSV)                                     │
│   • All provider responses appear as "Support Team"              │
└───────────────────────────┬──────────────────────────────────────┘
                            │ HTTP / WebSocket
                   ┌────────▼────────┐
                   │   REST API +    │
                   │   Socket.io     │
                   │   (Express.js)  │
                   └────────┬────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
   │  PostgreSQL │  │  SMTP Mail  │  │  File System│
   │  (RLS)      │  │  (Nodemail) │  │  (PDF/CSV)  │
   └─────────────┘  └─────────────┘  └─────────────┘
          │                 │
          │          ┌──────▼──────┐
          │          │  Dual Email │
          │          │  Microsoft  │
          │          │  + Google   │
          │          └─────────────┘
          │
┌─────────▼─────────────────────────────────────────────────────────┐
│                  PROVIDER OPERATIONAL CORE                        │
│   Full agent identity on every action                            │
│   Internal notes for pre-reply discussion                         │
│   Severity management with audit trail                            │
│   SLA tracking & agent KPI dashboards                             │
│   Cross-tenant admin visibility                                   │
│   Cross-department approval workflow                              │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│                DUAL-LAYER NOTIFICATION ENGINE                     │
│                                                                   │
│   Layer 1: Socket.io (0ms latency)                               │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ io.to('user_${userId}').emit('notification', payload)  │    │
│   │ → instant UI update + sound alert                       │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│   Layer 2: Web Push (background/closed)                          │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ VAPID auth → OS Push Service → Native notification      │    │
│   │ → click opens PWA directly to ticket                    │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│   Service Worker (sw.js):                                        │
│   • Offline caching (network-first + cache fallback)             │
│   • Web Push reception + notification display                    │
│   • Notification click → navigate to ticket                      │
└───────────────────────────────────────────────────────────────────┘
```

### Directory Structure

```
TKT/
├── backend/                    # Node.js + Express API server
│   ├── src/
│   │   ├── config/            # Database & app configuration
│   │   ├── controllers/       # Request handlers
│   │   │   ├── authController.js
│   │   │   ├── mfaController.js
│   │   │   ├── notificationController.js
│   │   │   ├── emailConfigController.js    # Dual email provider CRUD
│   │   │   ├── invitationController.js     # Email invitation workflow
│   │   │   ├── pushSubscriptionController.js  # Push subscription CRUD
│   │   │   ├── reportController.js
│   │   │   ├── tenantController.js
│   │   │   └── ticketController.js
│   │   ├── middleware/        # Auth, rate limiting, sanitization
│   │   │   ├── auth.js
│   │   │   ├── rateLimiter.js
│   │   │   └── sanitize.js
│   │   ├── models/           # Database query helpers
│   │   ├── routes/           # API route definitions
│   │   │   ├── auth.js
│   │   │   ├── notifications.js
│   │   │   ├── push.js               # Push notification routes
│   │   │   ├── reports.js
│   │   │   ├── tickets.js
│   │   │   ├── users.js
│   │   │   └── manager.js            # Manager routes (email, invitations, settings)
│   │   ├── services/         # Business logic services
│   │   │   ├── auditService.js
│   │   │   ├── emailService.js
│   │   │   ├── microsoftGraphService.js  # Microsoft OAuth + Graph API
│   │   │   ├── googleOAuthService.js     # Google OAuth + Gmail API
│   │   │   ├── providerEmailService.js   # Unified provider-aware sender
│   │   │   ├── pushNotificationService.js # VAPID + Web Push
│   │   │   ├── notificationService.js     # Dual-layer notification engine
│   │   │   └── reportService.js
│   │   ├── utils/            # Helper utilities
│   │   └── server.js         # Entry point (Socket.io + routes)
│   ├── scripts/              # Admin scripts
│   ├── .env.example          # Environment template
│   ├── Dockerfile
│   └── package.json
├── frontend/                  # React 18 SPA (PWA)
│   ├── public/
│   │   ├── manifest.json     # PWA manifest
│   │   ├── sw.js             # Service Worker
│   │   └── index.html        # Entry HTML with SW registration
│   ├── src/
│   │   ├── components/       # Shared layout components
│   │   │   ├── SOCLayout.js
│   │   │   ├── ClientLayout.js
│   │   │   ├── ManagerLayout.js
│   │   │   ├── AdminLayout.js
│   │   │   ├── NotificationBell.js   # Bell icon with badge + dropdown
│   │   │   └── CustomSelect.js
│   │   ├── contexts/         # React Context providers
│   │   │   ├── AuthContext.js
│   │   │   ├── LanguageContext.js
│   │   │   └── ThemeContext.js
│   │   ├── hooks/            # Custom React hooks
│   │   │   └── useNotifications.js   # Dual-layer notification hook
│   │   ├── pages/            # Route-level page components
│   │   │   ├── Login.js
│   │   │   ├── Dashboard.js
│   │   │   ├── TicketList.js
│   │   │   ├── TicketDetail.js
│   │   │   ├── CreateTicket.js
│   │   │   ├── ClientDashboard.js
│   │   │   ├── ClientTickets.js
│   │   │   ├── Reports.js
│   │   │   ├── UserManagement.js
│   │   │   ├── Settings.js
│   │   │   ├── TwoFactorSetup.js
│   │   │   ├── ForcePasswordChange.js
│   │   │   ├── NotificationSettings.js  # Notification preferences page
│   │   │   ├── ManagerEmailConfig.js    # Email provider config
│   │   │   ├── ManagerInvitations.js    # Invitation management
│   │   │   └── AcceptInvitation.js      # Public invitation acceptance
│   │   ├── services/         # API client & socket service
│   │   │   ├── api.js
│   │   │   ├── socket.js
│   │   │   ├── pushNotifications.js     # SW registration + push
│   │   │   └── notificationSound.js     # Web Audio API sounds
│   │   ├── utils/            # Translations, helpers
│   │   ├── App.js            # Route definitions
│   │   ├── index.js          # Entry point
│   │   └── index.css         # Global styles & glassmorphism
│   ├── tailwind.config.js
│   └── package.json
├── database/
│   ├── schema.sql            # Full schema + RLS + seed data
│   └── migration_*.sql       # Schema migrations (001-024)
├── docker-compose.yml        # Container orchestration
├── setup.sh / setup.bat      # One-command setup
└── start-servers.sh          # Dev server launcher
```

---

## 3. Technology Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Frontend** | React | 18.2 | UI framework |
| | React Router | 7.x | Client-side routing |
| | TailwindCSS | 3.4 | Utility-first CSS |
| | Recharts | 2.10 | Dashboard charts |
| | Lucide React | 0.312 | Icon library |
| | Socket.io Client | 4.7 | Real-time communication |
| | Axios | 1.6 | HTTP client |
| | React Hot Toast | 2.4 | Toast notifications |
| | qrcode.react | 4.2 | 2FA QR code generation |
| **Backend** | Node.js | 18+ | Runtime |
| | Express.js | 4.18 | Web framework |
| | PostgreSQL | 16 | Primary database |
| | pg (node-postgres) | 8.11 | Database driver |
| | Socket.io | 4.7 | WebSocket server |
| | JWT (jsonwebtoken) | 9.0 | Authentication tokens |
| | bcryptjs | 2.4 | Password hashing |
| | Helmet | 7.1 | Security headers |
| | express-rate-limit | 7.1 | API rate limiting |
| | express-validator | 7.0 | Input validation |
| | Nodemailer | 6.9 | Email delivery (SMTP fallback) |
| | web-push | 3.6 | Web Push (VAPID) |
| | PDFKit | 0.15 | PDF generation |
| | csv-writer | 1.6 | CSV export |
| | otplib / speakeasy | 13.x / 2.x | TOTP 2FA |
| | sanitize-html | 2.17 | XSS prevention |
| | multer | 2.2 | File upload handling |
| | archiver | 8.0 | Archive creation |
| **Database** | PostgreSQL | 16-alpine | Relational DB with RLS |
| **DevOps** | Docker Compose | 3.8 | Container orchestration |
| | Nodemon | 3.x | Dev auto-restart |

---

## 4. Getting Started

### Prerequisites

- **Node.js** 18 or higher
- **PostgreSQL** 16+ (or Docker)
- **npm** or **yarn**

### Quick Setup (Recommended)

```bash
# Clone the repository
git clone <repository-url>
cd TKT

# Run setup script
bash setup.sh          # Linux/Mac
# or
setup.bat              # Windows
```

The setup script will:
1. Verify Node.js version
2. Install backend and frontend dependencies
3. Start PostgreSQL via Docker (if Docker is available)
4. Initialize the database with schema and seed data
5. Create `.env` configuration file

### Docker Setup (Full Stack)

```bash
docker-compose up
```

This starts all three services:
- **PostgreSQL** on port `5432`
- **Backend API** on port `5000`
- **Frontend** on port `3000`

### Manual Setup

```bash
# 1. Start PostgreSQL and create database
createdb mssp_ticketing
psql -U postgres -d mssp_ticketing -f database/schema.sql

# 2. Configure backend
cd backend
cp .env.example .env    # Edit with your settings
npm install

# 3. Setup frontend
cd ../frontend
npm install

# 4. Start servers
# Terminal 1:
cd backend && npm run dev

# Terminal 2:
cd frontend && npm start
```

### Access

| Service | URL |
|---------|-----|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:5000/api |

---

## 5. Environment Configuration

### Backend `.env`

```bash
# Server
PORT=5000
NODE_ENV=development

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mssp_ticketing
DB_USER=postgres
DB_PASSWORD=postgres

# JWT Authentication
JWT_SECRET=your-super-secret-key-change-in-production
JWT_REFRESH_SECRET=your-refresh-secret-key

# Email (SMTP) - Optional fallback
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
SOC_EMAIL=support@yourcompany.com

# Frontend URL (for CORS)
FRONTEND_URL=http://localhost:3000

# VAPID Keys (Web Push) - Auto-generated if not provided
# Generate with: npx web-push generate-vapid-keys
VAPID_PUBLIC_KEY=your-vapid-public-key
VAPID_PRIVATE_KEY=your-vapid-private-key
VAPID_SUBJECT=mailto:support@yourcompany.com

# Email Encryption Key (AES-256-CBC)
# Used for encrypting OAuth credentials in tenant_email_configs
ENCRYPTION_KEY=your-32-character-encryption-key!
```

### Frontend Environment

Set via `REACT_APP_API_URL` in `docker-compose.yml` or `.env`:

```bash
REACT_APP_API_URL=http://localhost:5000/api
```

---

## 6. Database Schema

### Entity Relationship

```
tenants ─────┬────────────────────────────────────┐
             │                                    │
             ├──── users ─────────────────────────┤
             │       │                            │
             │       ├──── tickets ───────────────┤
             │       │       │                    │
             │       │       ├─ ticket_messages   │
             │       │       ├─ ticket_severity_history
             │       │       └─ notifications     │
             │       │                            │
             │       └──── audit_logs ────────────┘
             │
             └────────────────────────────────────┘
```

### Core Tables

#### `tenants` - Organizations

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| name | VARCHAR(255) | Organization name |
| slug | VARCHAR(100) | URL-friendly identifier (unique) |
| logo_url | VARCHAR(500) | Branding logo |
| primary_color | VARCHAR(7) | Brand color (hex) |
| is_active | BOOLEAN | Account status |
| enable_clients | BOOLEAN | Client module visibility toggle |

#### `users` - All System Users

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| tenant_id | UUID | FK to tenants |
| email | VARCHAR(255) | Unique login identifier |
| password_hash | VARCHAR(500) | bcrypt hash |
| full_name | VARCHAR(255) | Display name |
| role | VARCHAR(50) | RBAC role |
| department_id | UUID | FK to departments (nullable) |
| analyst_level | VARCHAR(10) | L1, L2, or L3 (department agents) |
| is_active | BOOLEAN | Account enabled/disabled |
| is_2fa_enabled | BOOLEAN | 2FA status |
| require_2fa_setup | BOOLEAN | Force 2FA setup on login |
| last_login | TIMESTAMPTZ | Last authentication time |

**Roles:** `super_admin`, `tenant_admin`, `department_agent`, `client_admin`, `client_user`

#### `tickets` - Security Incidents

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| tenant_id | UUID | FK to tenants (isolation scope) |
| ticket_number | SERIAL | Auto-incrementing display ID |
| title | VARCHAR(500) | Incident summary |
| description | TEXT | Full incident details |
| severity | ENUM | `high`, `medium`, `low` |
| status | ENUM | `open`, `pending_soc`, `in_progress`, `pending_approval`, `resolved`, `closed` |
| type | VARCHAR(20) | `soc_client`, `internal_dept`, `it_helpdesk` |
| source | VARCHAR(50) | `provider` or `client` (who created it) |
| assigned_analyst_id | UUID | FK to users (assigned agent) |
| department_id | UUID | FK to departments (routing) |
| created_by | UUID | FK to users (creator) |
| sla_breach_at | TIMESTAMPTZ | Calculated SLA deadline |
| first_response_at | TIMESTAMPTZ | Time of first agent response |
| resolved_at | TIMESTAMPTZ | Resolution timestamp |

#### `ticket_messages` - Communication Thread

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| ticket_id | UUID | FK to tickets |
| tenant_id | UUID | FK to tenants |
| author_id | UUID | FK to users |
| content | TEXT | Message body |
| visibility | VARCHAR(20) | `internal` (staff-only) or `external` (client-visible) |
| is_escalation | BOOLEAN | Escalation flag |

**Critical Design:** Messages with `visibility: internal` are **never** returned to client-facing API endpoints.

#### `ticket_assignments` - Cross-Department Approval

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| ticket_id | UUID | FK to tickets |
| from_department_id | UUID | Source department |
| to_department_id | UUID | Destination department |
| assigned_by | UUID | FK to users (requester) |
| approved_by | UUID | FK to users (approver, nullable) |
| status | VARCHAR(20) | `pending_approval`, `approved`, `rejected` |
| notes | TEXT | Approval/rejection notes |

#### `ticket_severity_history` - Severity Audit Trail

| Column | Type | Description |
|--------|------|-------------|
| ticket_id | UUID | FK to tickets |
| changed_by | UUID | FK to users |
| old_severity | ENUM | Previous level |
| new_severity | ENUM | New level |
| reason | TEXT | Change justification |

#### `audit_logs` - Action Tracking

| Column | Type | Description |
|--------|------|-------------|
| tenant_id | UUID | FK to tenants |
| user_id | UUID | FK to users |
| action | VARCHAR(100) | Action type (create, update, delete, etc.) |
| entity_type | VARCHAR(100) | Affected table |
| entity_id | UUID | Affected row |
| old_values | JSONB | Previous state |
| new_values | JSONB | New state |
| ip_address | INET | Client IP |
| user_agent | TEXT | Browser fingerprint |

#### `notifications` - In-App Alerts

| Column | Type | Description |
|--------|------|-------------|
| tenant_id | UUID | FK to tenants |
| user_id | UUID | FK to users (recipient) |
| ticket_id | UUID | FK to tickets (optional context) |
| type | VARCHAR(50) | Notification type |
| title | VARCHAR(500) | Notification heading |
| message | TEXT | Notification body |
| is_read | BOOLEAN | Read status |

#### `push_subscriptions` - Web Push Subscriptions

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| user_id | UUID | FK to users (cascade delete) |
| tenant_id | UUID | FK to tenants |
| endpoint | TEXT | Push service endpoint |
| p256dh_key | TEXT | Diffie-Hellman public key |
| auth_key | TEXT | Authentication secret |
| device_type | VARCHAR(20) | `desktop` or `mobile` |
| is_active | BOOLEAN | Subscription status |
| last_used_at | TIMESTAMPTZ | Last push sent |

**Unique constraint:** `user_id + endpoint`

#### `notification_preferences` - Per-User Settings

| Column | Type | Description |
|--------|------|-------------|
| user_id | UUID | FK to users (unique) |
| ticket_created | BOOLEAN | New ticket notifications |
| ticket_assigned | BOOLEAN | Assignment notifications |
| ticket_status_changed | BOOLEAN | Status change notifications |
| ticket_comment | BOOLEAN | Comment notifications |
| ticket_cross_deptApproval | BOOLEAN | Approval request notifications |
| ticket_escalated | BOOLEAN | Escalation notifications |
| push_enabled | BOOLEAN | Web Push enabled |
| sound_enabled | BOOLEAN | Sound alerts enabled |
| browser_notifications | BOOLEAN | Browser notifications enabled |

#### `notification_log` - Delivery Audit

| Column | Type | Description |
|--------|------|-------------|
| user_id | UUID | FK to users |
| tenant_id | UUID | FK to tenants |
| notification_type | VARCHAR(50) | Event type |
| title | VARCHAR(255) | Notification title |
| ticket_id | UUID | Related ticket |
| delivery_channel | VARCHAR(20) | `socket`, `push`, or `both` |
| delivered_at | TIMESTAMPTZ | Delivery timestamp |
| clicked_at | TIMESTAMPTZ | User clicked (nullable) |

#### `vapid_keys` - VAPID Key Storage

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| public_key | TEXT | VAPID public key |
| encrypted_private_key | TEXT | AES-256-CBC encrypted private key |
| subject | VARCHAR(255) | Contact email |

#### `tenant_email_configs` - Email Provider Config

| Column | Type | Description |
|--------|------|-------------|
| tenant_id | UUID | FK to tenants (unique) |
| provider | VARCHAR(20) | `microsoft` or `google` |
| encrypted_client_id | TEXT | AES-256-CBC encrypted |
| encrypted_client_secret | TEXT | AES-256-CBC encrypted |
| encrypted_azure_tenant_id | TEXT | Microsoft only |
| encrypted_refresh_token | TEXT | AES-256-CBC encrypted |
| sender_email | VARCHAR(255) | From address |
| sender_name | VARCHAR(255) | Display name |

#### `invitations` - Pending Invitations

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| tenant_id | UUID | FK to tenants |
| email | VARCHAR(255) | Invitee email |
| role | VARCHAR(50) | Assigned role |
| department_id | UUID | FK to departments (nullable) |
| invited_by | UUID | FK to users |
| token_hash | VARCHAR(255) | SHA-256 hashed token |
| expires_at | TIMESTAMPTZ | 7-day expiry |
| accepted_at | TIMESTAMPTZ | Acceptance timestamp |

### Indexes

Performance indexes are defined on all frequently queried columns:

- `idx_users_tenant`, `idx_users_email`, `idx_users_role`
- `idx_tickets_tenant`, `idx_tickets_status`, `idx_tickets_severity`, `idx_tickets_assigned`, `idx_tickets_created`
- `idx_ticket_messages_ticket`, `idx_ticket_messages_visibility`
- `idx_audit_logs_tenant`, `idx_audit_logs_user`, `idx_audit_logs_created`
- `idx_notifications_user`, `idx_notifications_unread`

### Migrations

Applied in order:

| Migration | Purpose |
|-----------|---------|
| `migration_001.sql` | Initial schema creation |
| `migration_002_soft_delete_and_seed.sql` | Soft delete support + seed data |
| `migration_003_security_hardening.sql` | Security enhancements |
| `migration_004_fix_analyst_assignments.sql` | Assignment fixes |
| `migration_005_fix_email_unique_constraint.sql` | Email uniqueness |
| `migration_006_must_change_password_and_2fa.sql` | Password policy + 2FA |
| `migration_007_require_2fa_setup.sql` | Mandatory 2FA setup |
| `migration_008` - `migration_021` | Various feature additions |
| `migration_022_features.sql` | `enable_clients`, `ticket_assignments`, `pending_approval` |
| `migration_023_email_features.sql` | `tenant_email_configs`, `invitations` |
| `migration_024_pwa_notifications.sql` | `vapid_keys`, `push_subscriptions`, `notification_preferences`, `notification_log` |

---

## 7. Authentication & Authorization

### Authentication Flow

```
1. Client submits email + password
         │
2. Backend validates credentials (bcrypt compare)
         │
3. Backend generates JWT token (24h expiry)
   Token payload: { userId, tenantId, role, email }
         │
4. Token stored in HTTP-only cookie or localStorage
         │
5. Subsequent requests include token in Authorization header
         │
6. Auth middleware verifies JWT and sets session variables:
   SET app.current_tenant = '<tenant_id>'
   SET app.current_user_id = '<user_id>'
   SET app.current_role = '<role>'
         │
7. PostgreSQL RLS policies use these session variables
   for automatic tenant isolation
```

### JWT Configuration

```javascript
{
  secret: process.env.JWT_SECRET,    // Must be strong in production
  expiresIn: '24h'                    // Token expiry
}
```

### Password Security

- **Hashing:** bcrypt with 12 salt rounds
- **Policy:** Minimum 8 characters, complexity requirements
- **Force change:** Admin can require password change on first login
- **2FA:** TOTP-based (RFC 6238) compatible with Google Authenticator, Authy

### Two-Factor Authentication (2FA)

```
1. User initiates 2FA setup
         │
2. Backend generates TOTP secret (speakeasy)
         │
3. QR code generated for authenticator app (qrcode.react)
         │
4. User scans QR and enters 6-digit code to verify
         │
5. Backend enables 2FA for the account
         │
6. Login flow: password → 2FA code verification
```

### Role-Based Access Control (RBAC)

| Resource | super_admin | tenant_admin | department_agent | client_user |
|----------|:----------:|:-----------:|:-----------:|:-----------:|
| All tenants data | ✅ | ❌ | ❌ | ❌ |
| User management | ✅ | ❌ | ❌ | ❌ |
| All tickets | ✅ | ✅ (own tenant) | ✅ (assigned) | ❌ |
| Own org tickets | ✅ | ✅ | ✅ | ✅ |
| Create tickets | ✅ | ✅ | ✅ | ✅ |
| Internal notes | ✅ | ✅ | ✅ | ❌ |
| Change severity | ✅ | ✅ | ❌ | ❌ |
| Assign analysts | ✅ | ✅ | ❌ | ❌ |
| Reports | ✅ | ✅ | ❌ | ✅ (own data) |
| Dashboard stats | ✅ | ✅ | ✅ | ✅ (own data) |

---

## 8. Multi-Tenancy & Security

### Three-Layer Isolation

#### 1. Database-Level (PostgreSQL RLS)

Every tenant-scoped table has Row-Level Security policies:

```sql
-- Example: Tickets isolation
CREATE POLICY ticket_isolation ON tickets
    FOR ALL
    USING (
        tenant_id = current_setting('app.current_tenant')::UUID
        OR current_setting('app.current_role') = 'super_admin'
    );
```

**How it works:**
- Backend sets session variables before each query:
  ```sql
  SET app.current_tenant = 'tenant-uuid';
  SET app.current_user_id = 'user-uuid';
  SET app.current_role = 'department_agent';
  ```
- PostgreSQL automatically filters rows based on RLS policies
- Super Admin bypasses tenant filtering for cross-tenant access

#### 2. Application-Level

- Every API query is scoped by `tenant_id` via middleware
- `req.user.tenantId` is injected from the JWT token
- Service layer enforces role-based permissions

#### 3. UI-Level

- Client users can only see their own organization's data
- Navigation and features are conditionally rendered based on role
- Client portal shows "Support Team" instead of individual agent names

### Security Measures

| Measure | Implementation |
|---------|---------------|
| **Authentication** | JWT with HTTP-only + Secure + SameSite=Strict cookies |
| **Password hashing** | bcrypt (12 rounds) |
| **Security headers** | Helmet.js + Permissions-Policy (13 headers) |
| **Rate limiting** | express-rate-limit (5 auth/15min, 3 strict/hour, 100 global/15min) |
| **Input validation** | express-validator + sanitize-html |
| **XSS prevention** | Input sanitization + CSP + output escaping |
| **SQL injection** | Parameterized queries via pg driver |
| **CORS** | Strict origin allowlist (no dev bypass) |
| **Internal data** | `visibility: internal` messages never sent to client endpoints |
| **Audit trail** | Every action logged with old/new values, IP, user agent |
| **Cookie security** | HttpOnly, Secure, SameSite=Strict on all auth cookies |
| **Brute-force** | 5 failed attempts → 15min lockout + rate limiter |

### Rate Limiting

```javascript
// General API: 100 requests per 15 minutes
// Auth endpoints: 10 requests per 15 minutes
// Ticket creation: 20 requests per 15 minutes
```

---

## 9. API Reference

### Base URL

```
http://localhost:5000/api
```

### Authentication

All endpoints (except login) require:

```
Authorization: Bearer <jwt_token>
```

### Auth Endpoints

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/auth/login` | Authenticate user | No |
| POST | `/api/auth/register` | Create user (Admin/Manager) | Admin/Manager |
| GET | `/api/auth/me` | Get current user profile | Yes |
| POST | `/api/auth/change-password` | Change own password | Yes |

#### POST `/api/auth/login`

**Request:**
```json
{
  "email": "user@example.com",
  "password": "your_password"
}
```

**Response (200):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "fullName": "System Administrator",
    "role": "super_admin",
    "tenantId": "uuid"
  }
}
```

**Response (401):**
```json
{
  "error": "Invalid email or password"
}
```

### Ticket Endpoints

| Method | Endpoint | Description | Roles |
|--------|----------|-------------|-------|
| GET | `/api/tickets` | List tickets (filtered by tenant) | All |
| POST | `/api/tickets` | Create new ticket | All |
| GET | `/api/tickets/:id` | Get ticket with messages | All (tenant-scoped) |
| POST | `/api/tickets/:id/messages` | Add message to ticket | All |
| PATCH | `/api/tickets/:id/status` | Update ticket status | Admin/Manager |
| PATCH | `/api/tickets/:id/severity` | Update severity (with audit) | Admin/Manager |
| PATCH | `/api/tickets/:id/assign` | Assign agent | Admin/Manager |
| GET | `/api/tickets/dashboard/stats` | Dashboard statistics | All |

#### GET `/api/tickets`

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| status | string | - | Filter by status |
| severity | string | - | Filter by severity |
| page | number | 1 | Page number |
| limit | number | 20 | Items per page |
| search | string | - | Search in title/description |

**Response (200):**
```json
{
  "tickets": [
    {
      "id": "uuid",
      "ticket_number": 8001,
      "title": "Brute force detected",
      "severity": "high",
      "status": "open",
      "assigned_analyst_name": "Zaid Ali",
      "created_at": "2026-01-15T10:30:00Z",
      "message_count": 3
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "pages": 3
  }
}
```

#### POST `/api/tickets/:id/messages`

**Request:**
```json
{
  "content": "Investigating the brute force source...",
  "visibility": "internal"
}
```

**Visibility Rules:**
- `external` — Visible to both provider and client (client sees "Support Team")
- `internal` — Staff-only private notes (never sent to client endpoints)

### User Management Endpoints

| Method | Endpoint | Description | Roles |
|--------|----------|-------------|-------|
| GET | `/api/users` | List all users | Admin |
| POST | `/api/users` | Create new user | Admin |
| PATCH | `/api/users/:id` | Update user | Admin |
| DELETE | `/api/users/:id` | Deactivate user | Admin |
| POST | `/api/users/:id/reset-password` | Reset password | Admin |
| POST | `/api/users/:id/toggle-2fa` | Enable/disable 2FA | Admin |

### Report Endpoints

| Method | Endpoint | Description | Roles |
|--------|----------|-------------|-------|
| GET | `/api/reports/dashboard` | Client dashboard metrics | Client |
| POST | `/api/reports/export` | Export PDF/CSV | All |

#### POST `/api/reports/export`

**Request:**
```json
{
  "format": "pdf",
  "dateFrom": "2026-01-01",
  "dateTo": "2026-01-31",
  "severity": "high"
}
```

### Notification Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/notifications` | List user's notifications |
| PATCH | `/api/notifications/:id/read` | Mark as read |
| PATCH | `/api/notifications/read-all` | Mark all as read |

### Push Notification Endpoints

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/push/vapid-public-key` | Get VAPID public key | No |
| POST | `/api/push/subscribe` | Register push subscription | Yes |
| POST | `/api/push/unsubscribe` | Remove subscription | Yes |
| GET | `/api/push/subscriptions` | List active devices | Yes |
| GET | `/api/push/preferences` | Get notification preferences | Yes |
| PUT | `/api/push/preferences` | Update preferences | Yes |
| POST | `/api/push/test` | Send test notification | Yes |
| GET | `/api/push/log` | Notification delivery log | Yes |

### Email Configuration Endpoints

| Method | Endpoint | Description | Roles |
|--------|----------|-------------|-------|
| GET | `/api/manager/email-config` | Get email config (Microsoft/Google) | Super Admin, Tenant Admin |
| POST | `/api/manager/email-config` | Save email config | Super Admin, Tenant Admin |
| POST | `/api/manager/email-config/test` | Test connection | Super Admin, Tenant Admin |
| DELETE | `/api/manager/email-config` | Delete config | Super Admin, Tenant Admin |

### Invitation Endpoints

| Method | Endpoint | Description | Roles |
|--------|----------|-------------|-------|
| GET | `/api/manager/invitations` | List invitations | Manager |
| POST | `/api/manager/invite` | Send invitation | Manager |
| DELETE | `/api/manager/invitations/:id` | Revoke invitation | Manager |
| POST | `/api/auth/accept-invite` | Accept invitation (public) | No |

### Tenant Settings Endpoints

| Method | Endpoint | Description | Roles |
|--------|----------|-------------|-------|
| GET | `/api/manager/settings` | Get tenant settings (enable_clients) | Manager/Agent |

---

## 10. Frontend Architecture

### State Management

The app uses React Context for global state:

| Context | Purpose |
|---------|---------|
| `AuthContext` | User authentication state, login/logout |
| `ThemeContext` | Dark/light theme toggle (persisted in localStorage) |
| `LanguageContext` | Arabic/English toggle (RTL support) |

### Custom Hooks

| Hook | Purpose |
|------|---------|
| `useNotifications` | Dual-layer notification management (socket + push) |

### Routing

#### System Admin Routes

| Path | Component | Access |
|------|-----------|--------|
| `/admin` | AdminDashboard | Super Admin |
| `/admin/tenants` | AdminTenants | Super Admin |
| `/admin/tenants/:tenantId/manage` | AdminTenantManage | Super Admin |
| `/admin/users` | UserManagement | Super Admin |
| `/admin/support` | AdminSupport | Super Admin |
| `/admin/analytics` | AdminAnalytics | Super Admin |
| `/admin/notifications` | NotificationSettings | Super Admin |
| `/admin/settings` | Settings | Super Admin |
| `/admin/settings/2fa` | TwoFactorSetup | Super Admin |

#### Tenant Admin Routes

| Path | Component | Access |
|------|-----------|--------|
| `/manager` | ManagerOverview | Tenant Admin |
| `/manager/employees` | ManagerEmployees | Tenant Admin |
| `/manager/clients` | ManagerClients | Tenant Admin (enable_clients) |
| `/manager/departments` | ManagerDepartments | Tenant Admin |
| `/manager/tickets` | TicketList | Tenant Admin / Agent |
| `/manager/tickets/new` | CreateTicket | Tenant Admin / Agent |
| `/manager/tickets/:id` | TicketDetail | Tenant Admin / Agent |
| `/manager/settings` | Settings | Tenant Admin / Agent |
| `/manager/settings/2fa` | TwoFactorSetup | Tenant Admin / Agent |
| `/manager/settings/email` | ManagerEmailConfig | Super Admin / Tenant Admin |
| `/manager/invitations` | ManagerInvitations | Tenant Admin |
| `/manager/report-issue` | ReportIssue | Tenant Admin |
| `/manager/notifications` | NotificationSettings | Tenant Admin / Agent |

#### Department Manager Routes

| Path | Component | Access |
|------|-----------|--------|
| `/department/manager` | ManagerOverview | Dept Manager |
| `/department/manager/employees` | ManagerEmployees | Dept Manager |
| `/department/manager/tickets` | TicketList | Dept Manager |
| `/department/manager/tickets/:id` | TicketDetail | Dept Manager |
| `/department/manager/notifications` | NotificationSettings | Dept Manager |
| `/department/manager/settings` | Settings | Dept Manager |

#### Department Employee Routes

| Path | Component | Access |
|------|-----------|--------|
| `/department/employee` | TicketList | Department Agent |
| `/department/employee/my-tickets` | EmployeeMyTickets | Department Agent |
| `/department/employee/tickets/:id` | TicketDetail | Department Agent |
| `/department/employee/notifications` | NotificationSettings | Department Agent |
| `/department/employee/settings` | Settings | Department Agent |

#### Client Routes

| Path | Component | Access |
|------|-----------|--------|
| `/client` | ClientDashboard | Client |
| `/client/tickets` | ClientTickets | Client |
| `/client/tickets/:id` | TicketDetail (isClient) | Client |
| `/client/settings` | Settings | Client |

#### Public Routes

| Path | Component | Access |
|------|-----------|--------|
| `/login` | Login | Public |
| `/auth/force-change-password` | ForcePasswordChange | Public |
| `/auth/setup-2fa` | TwoFactorSetup | Public |
| `/auth/accept-invite` | AcceptInvitation | Public |

### Layout Components

All four layout components (`AdminLayout`, `SOCLayout`, `ManagerLayout`, `ClientLayout`) share a consistent **profile dropdown** in the header:

| Menu Item | Action |
|-----------|--------|
| Change Password | Navigates to settings page |
| Enable/Disable MFA | Navigates to 2FA setup page |
| Light Mode / Dark Mode | Toggles theme (persisted in localStorage) |
| English / العربية | Toggles language (persisted in localStorage) |
| Logout | Clears session, redirects to `/login` |

Standalone theme and language toggles have been removed from the main dashboard — they are now only accessible via the profile dropdown. The logout button is only available inside the profile dropdown (no standalone logout button in sidebar or header).

The UI uses a custom glassmorphism design with CSS variables:

```css
/* Theme variables */
--bg-primary: #06080f;          /* Dark background */
--text-primary: #e2e8f0;        /* Primary text */
--glass-bg: rgba(255,255,255,0.04);  /* Glass panel background */
--glass-border: rgba(255,255,255,0.07);
--accent: #3b82f6;              /* Primary action color */
```

**Key CSS Classes:**

| Class | Usage |
|-------|-------|
| `.glass` | Glassmorphism panel |
| `.card` | Elevated card with glass effect |
| `.btn-primary` | Primary action button |
| `.btn-secondary` | Secondary action button |
| `.input` | Form input field |
| `.badge-high/medium/low` | Severity badges |
| `.status-open/pending/progress/resolved/closed` | Status badges |
| `.nav-item` | Sidebar navigation item |
| `.table-row` | Table row with hover effect |

### Internationalization

- **Arabic (RTL)** and **English (LTR)** supported
- Direction changes via `dir` attribute on root element
- Font family switches to Readex Pro for Arabic
- All UI text uses translation keys via `useLang()` hook

### Real-Time Features

Socket.io events:

| Event | Direction | Description |
|-------|-----------|-------------|
| `notification:new` | Server → Client | New notification received |
| `ticket:created` | Server → Client | New ticket created |
| `ticket:updated` | Server → Client | Ticket status/severity changed |
| `ticket:message` | Server → Client | New message on subscribed ticket |

---

## 11. Real-Time Features

### Dual-Layer Notification Engine

The notification system uses two delivery layers to ensure 100% delivery regardless of user connection state:

#### Layer 1: Socket.io (Active Session — 0ms Latency)

```javascript
// Connection flow
connectSocket(userId)     // Connect to server
joinRole(role)            // Join role-specific room
joinTicket(ticketId)      // Join ticket room for live updates
disconnectSocket()        // Cleanup on unmount

// Events
socket.on('notification', handler)        // All notifications
socket.on('ticket_created', handler)     // New ticket
socket.on('ticket_assigned', handler)    // Assignment
socket.on('ticket_status_changed', handler) // Status change
socket.on('ticket_comment', handler)     // New comment
socket.on('ticket_cross_dept_approval', handler) // Approval needed
socket.on('ticket_escalated', handler)   // Escalation
```

**When:** User is actively using the app (WebSocket connected)
**Delivery:** Instant (0ms) via `io.to('user_${userId}').emit()`

#### Layer 2: Web Push (Background/Closed — VAPID)

```javascript
// Service Worker handles push
self.addEventListener('push', (event) => {
  const data = event.data.json();
  self.registration.showNotification(data.title, {
    body: data.body,
    icon: '/assets/logo.png',
    tag: data.tag,
    data: { url: data.url, ticketId: data.ticketId },
  });
});

// Click notification → navigate to ticket
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  self.clients.openWindow(event.notification.data.url);
});
```

**When:** App is closed or backgrounded (no WebSocket)
**Delivery:** Via OS push service (Windows/macOS/Linux/Android/iOS)

#### Notification Flow

```
Event occurs (ticket created, assigned, etc.)
         │
         ▼
notificationService.js
         │
         ├── Check user preferences (notification_preferences table)
         │
         ├── Layer 1: Socket.io
         │   io.to('user_${userId}').emit('notification', payload)
         │   → Instant UI update + sound alert
         │
         ├── Layer 2: Web Push (if user not online OR push_enabled)
         │   sendPushToUser(userId, payload)
         │   → VAPID auth → OS Push Service → Native notification
         │
         └── Log to notification_log (channel: 'socket'|'push'|'both')
```

### Notification Types

| Type | Description | Default Sound |
|------|-------------|---------------|
| `ticket_created` | New ticket in your department | Ascending tone |
| `ticket_assigned` | Ticket assigned to you | Descending tone |
| `ticket_status_changed` | Ticket status updated | Mid tone |
| `ticket_comment` | New comment on your ticket | High tone |
| `ticket_cross_deptApproval` | Approval needed for cross-dept transfer | Triple tone |
| `ticket_escalated` | Ticket escalated to you | Descending triple |

### PWA Features

- **manifest.json** — Standalone display, installable from address bar
- **sw.js** — Offline caching (network-first with cache fallback)
- **Notification click** — Opens app directly to relevant ticket
- **Online tracking** — Server tracks active socket connections per user

### Sound Alerts

Configurable via `notificationSound.js` using Web Audio API:

```javascript
import { playNotificationSound } from '../services/notificationSound';

// Play sound for specific notification type
playNotificationSound('ticket_created');  // Ascending tone
playNotificationSound('ticket_escalated'); // Descending triple tone
```

Different tones for different notification types to help users distinguish urgency without looking at the screen.

---

## 12. Ticket Lifecycle

### Status Flow

```
                    ┌──────────────────────────────────────────────┐
                    │                                              │
                    ▼                                              │
    ┌────────┐   ┌────────────┐   ┌────────────┐   ┌──────────┐  │
    │  OPEN  │──▶│ PENDING    │──▶│ IN         │──▶│ RESOLVED │──┤
    │        │   │ Support    │   │ PROGRESS   │   │          │  │
    └────────┘   └────────────┘   └────────────┘   └──────────┘  │
         │                                                       │
         │              ┌──────────┐                             │
         └─────────────▶│ CLOSED   │◀────────────────────────────┘
                        └──────────┘
```

### Status Definitions

| Status | Description | SLA Clock |
|--------|-------------|-----------|
| `open` | New ticket, awaiting triage | Running |
| `pending_approval` | Pending manager approval | Paused |
| `in_progress` | Actively being worked | Running |
| `resolved` | Issue fixed, pending client confirmation | Stopped |
| `closed` | Confirmed closed | Stopped |

### Severity & SLA

| Severity | SLA Target | Description |
|----------|-----------|-------------|
| High | 4 hours | Critical security incidents, active breaches |
| Medium | 24 hours | Standard incidents, suspicious activity |
| Low | 72 hours | Informational, non-urgent issues |

SLA breach is calculated at ticket creation:

```javascript
sla_breach_at = created_at + sla_duration(severity)
```

### Severity Change Audit

Every severity change is recorded with:
- Previous and new severity levels
- Who made the change (agent ID)
- Reason/justification
- Timestamp

---

## 13. User Roles & Permissions

### Super Admin (`super_admin`)

**Full system access across all tenants:**

- View all tickets across all client organizations
- Create, edit, delete users
- Assign roles and agent levels
- Manage 2FA policies
- View audit logs
- Generate reports
- Cross-tenant dashboard analytics

### Tenant Admin (`tenant_admin`)

**Team management within own tenant:**

- View all tickets in own organization
- Assign tickets to analysts
- Change ticket severity
- View team performance reports
- Manage ticket status
- Access dashboard with team metrics

### Department Agent (`department_agent`)

**Operational ticket handling:**

- View assigned tickets
- Respond to tickets (external messages)
- Create internal notes (hidden from client)
- View own performance metrics
- Available levels: L1 (triage), L2 (investigation), L3 (escalation)

### Client User (`client_user`)

**Self-service portal for own organization:**

- View own organization's tickets only
- Reply to provider responses
- Create new tickets
- View client dashboard
- Export own reports
- All provider responses appear as "Support Team" (no agent names)

---

## 14. Deployment

### Docker Production Deployment

```yaml
# docker-compose.prod.yml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mssp_ticketing
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: always

  backend:
    build: ./backend
    environment:
      - NODE_ENV=production
      - JWT_SECRET=${JWT_SECRET}
      - DB_HOST=postgres
      - DB_PASSWORD=${DB_PASSWORD}
    depends_on:
      - postgres
    restart: always

  frontend:
    build: ./frontend
    depends_on:
      - backend
    restart: always
```

### Production Checklist

- [ ] Change `JWT_SECRET` to a strong, unique value
- [ ] Use production PostgreSQL with proper credentials
- [ ] Enable HTTPS (reverse proxy with Nginx/Caddy)
- [ ] Configure SMTP for email notifications
- [ ] Set `NODE_ENV=production`
- [ ] Enable database backups
- [ ] Configure rate limiting for production traffic
- [ ] Set up monitoring and logging
- [ ] Remove default seed accounts

### Nginx Reverse Proxy Example

```nginx
server {
    listen 443 ssl http2;
    server_name support.yourcompany.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /api {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /socket.io {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }
}
```

---

## 15. Troubleshooting

### Common Issues

#### Database Connection Failed

```
Error: connect ECONNREFUSED 127.0.0.1:5432
```

**Solution:**
```bash
# Ensure PostgreSQL is running
docker-compose up -d postgres

# Verify connection
psql -U postgres -h localhost -d mssp_ticketing
```

#### JWT Authentication Errors

```
Error: invalid token / token expired
```

**Solution:**
- Check `JWT_SECRET` in `.env` matches between restarts
- Clear browser localStorage and re-login
- Verify token expiry is reasonable (15min access + 1d refresh)

#### CORS Errors

```
Error: Blocked by CORS policy
```

**Solution:**
- Ensure `FRONTEND_URL` in backend `.env` matches your frontend URL
- Check CORS configuration in `server.js`

#### Socket.io Connection Issues

```
Error: WebSocket connection failed
```

**Solution:**
- Verify backend is running on the expected port
- Check firewall/proxy WebSocket configuration
- Ensure `REACT_APP_API_URL` is correct in frontend

#### RLS Policy Errors

```
Error: permission denied for table tickets
```

**Solution:**
- Verify `SET app.current_tenant` is executed before queries
- Check the user's `tenant_id` matches the session variable
- Ensure RLS policies are applied to the correct tables

#### Push Notifications Not Working

```
Error: Push subscription failed
```

**Solution:**
- Ensure HTTPS is enabled (Web Push requires secure context)
- Check VAPID keys are configured in `.env`
- Verify browser notification permission is granted
- Check service worker is registered (Application tab in DevTools)

#### Email Configuration Fails

```
Error: Token refresh failed
```

**Solution:**
- Verify OAuth credentials are correct for the provider (Microsoft/Google)
- Check refresh token hasn't been revoked
- Ensure sender email matches the OAuth account domain
- Use the "Test Connection" button to diagnose issues

### Development Tips

```bash
# Reset database completely
docker-compose down -v
docker-compose up -d postgres
psql -U postgres -d mssp_ticketing -f database/schema.sql

# Reset 2FA for all users
cd backend && node scripts/reset-all-2fa.js

# Generate VAPID keys for Web Push
npx web-push generate-vapid-keys

# Watch backend logs
docker-compose logs -f backend

# Watch frontend build
cd frontend && npm start
```

---

## License

Proprietary - Internal Use Only

---

*Last updated: August 2026*
