# tikter — API Documentation

## Base URL

```
Development: http://localhost:5000/api
Production:  https://yourdomain.com/api
```

## Authentication

### Headers

All authenticated requests require:

```http
Content-Type: application/json
Cookie: access_token=<jwt>; refresh_token=<jwt>
```

Or via Bearer token (for API clients / WebSocket):

```http
Authorization: Bearer <access_token>
```

### Token Lifecycle

| Token | Duration | Storage |
|-------|----------|---------|
| Access Token | 15 minutes | HttpOnly cookie |
| Refresh Token | 7 days | HttpOnly cookie |
| User Data | Session | localStorage |

---

## Role Hierarchy

| Role | Scope | Description |
|------|-------|-------------|
| `mssp_admin` | Platform-wide | System administrator (Super Admin) |
| `soc_manager` | Own tenant | Tenant Manager / Department Manager |
| `soc_analyst` | Assigned department | Department Employee |
| `client_admin` | Own organization | Client Administrator |
| `client_employee` | Own organization | Client End-User |

---

## Endpoints

### Authentication

#### POST `/api/auth/login`

Authenticate user and receive session cookies.

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
  "user": {
    "id": "uuid",
    "tenantId": "uuid",
    "email": "user@example.com",
    "fullName": "System Administrator",
    "role": "mssp_admin",
    "analystLevel": null,
    "mfaEnabled": false,
    "is2faEnabled": false,
    "departmentId": null,
    "departmentCode": null,
    "departmentName": null,
    "departmentIsInternal": null
  }
}
```

**Response — MFA Required (200):**
```json
{
  "mfaRequired": true,
  "message": "Multi-factor authentication required. Provide mfaCode."
}
```

**Response — Password Change Required (200):**
```json
{
  "requirePasswordChange": true,
  "tempToken": "jwt_token",
  "message": "You must change your password before continuing."
}
```

**Error Responses:**
```json
// 400 - Missing fields
{ "error": "Email and password are required." }

// 401 - Invalid credentials
{ "error": "Invalid credentials." }

// 403 - Account disabled
{ "error": "Account has been disabled. Contact your administrator." }

// 429 - Too many attempts
{ "error": "Too many failed attempts. Account locked for 15 minutes." }
```

---

#### GET `/api/auth/me`

Get current authenticated user profile.

**Response (200):**
```json
{
  "user": {
    "id": "uuid",
    "tenantId": "uuid",
    "email": "user@example.com",
    "fullName": "System Administrator",
    "role": "mssp_admin",
    "analystLevel": null,
    "mfaEnabled": false,
    "departmentId": null,
    "departmentCode": null,
    "departmentName": null,
    "departmentIsInternal": null
  }
}
```

---

#### POST `/api/auth/logout`

Clear session cookies and revoke refresh token.

**Response (200):**
```json
{ "message": "Logged out successfully." }
```

---

#### POST `/api/auth/refresh`

Refresh access token using refresh token cookie.

**Response (200):**
```json
{ "message": "Token refreshed." }
```

---

#### POST `/api/auth/forgot-password`

Request password reset email with temporary password.

**Request:**
```json
{
  "email": "user@example.com"
}
```

**Response (200):**
```json
{
  "message": "If the email exists, a temporary password has been sent."
}
```

---

#### POST `/api/auth/reset-password`

Reset password using token from email.

**Request:**
```json
{
  "token": "reset_token_from_email",
  "newPassword": "NewPassword123!"
}
```

---

#### POST `/api/auth/change-password`

Change password for authenticated user.

**Request:**
```json
{
  "currentPassword": "OldPassword123!",
  "newPassword": "NewPassword123!"
}
```

---

#### POST `/api/auth/register`

Register a new user (requires `mssp_admin` or `soc_manager`).

**Request:**
```json
{
  "email": "newuser@example.com",
  "password": "SecurePass123!",
  "fullName": "John Doe",
  "role": "soc_analyst",
  "departmentId": "uuid"
}
```

---

#### POST `/api/auth/change-password-forced`

Force-change password (e.g., first login).

---

#### POST `/api/auth/accept-invite`

Accept a user invitation with hashed token.

**Request:**
```json
{
  "token": "invitation_token",
  "password": "NewPassword123!"
}
```

---

### MFA / 2FA

#### POST `/api/auth/mfa/setup`

Initiate MFA/TOTP setup. Returns QR code URL and secret.

---

#### POST `/api/auth/mfa/verify`

Verify MFA TOTP code during login.

**Request:**
```json
{
  "tempToken": "jwt_token",
  "mfaCode": "123456"
}
```

---

#### POST `/api/auth/mfa/complete-setup`

Complete MFA setup callback.

---

#### POST `/api/auth/mfa/disable`

Disable MFA for authenticated user.

---

### Super Admin — Auth Operations

#### POST `/api/auth/admin/reset-password/:userId`

Admin reset another user's password.

#### POST `/api/auth/admin/toggle-status/:userId`

Admin activate/deactivate a user.

#### POST `/api/auth/admin/force-password-change/:userId`

Admin force a user to change password on next login.

#### POST `/api/auth/admin/toggle-2fa/:userId`

Admin toggle 2FA for a user.

#### PATCH `/api/auth/admin/users/:userId/2fa-status`

Admin update a user's 2FA status.

#### POST `/api/auth/admin/reset-all-2fa`

Admin reset all 2FA registrations (emergency).

#### PUT `/api/auth/admin/update-user/:userId`

Admin update user details (name, email, role).

#### DELETE `/api/auth/admin/delete-user/:userId`

Admin soft-delete a user.

---

### Tenants

#### GET `/api/superadmin/tenants`

List all tenants (Super Admin only).

**Response (200):**
```json
{
  "tenants": [
    {
      "id": "uuid",
      "name": "Acme Corp",
      "slug": "acme-corp",
      "isActive": true,
      "createdAt": "2026-01-01T00:00:00Z"
    }
  ]
}
```

#### POST `/api/superadmin/tenants`

Create a new tenant.

**Request:**
```json
{
  "name": "Acme Corp",
  "slug": "acme-corp",
  "adminEmail": "admin@acme.com",
  "adminPassword": "AdminPass123!"
}
```

#### DELETE `/api/superadmin/tenants/:id`

Delete a tenant (Super Admin only).

#### PATCH `/api/superadmin/tenants/:id/status`

Update tenant active/inactive status.

**Request:**
```json
{
  "isActive": false
}
```

#### PATCH `/api/superadmin/tenants/:id/limits`

Update tenant resource limits.

**Request:**
```json
{
  "maxUsers": 50,
  "maxDepartments": 10,
  "maxTicketsPerMonth": 500,
  "maxEmailsPerMonth": 1000
}
```

#### GET `/api/superadmin/tenants/:id/export`

Export a tenant's data (backup) as JSON.

#### POST `/api/superadmin/tenants/import`

Import/restore a tenant from backup (multipart, max 50MB).

#### PATCH `/api/superadmin/tenants/:id/admin/status`

Toggle a tenant admin's status.

---

### Super Admin — Tenant User Management

#### GET `/api/admin-tenant/:tenantId/users`

List users belonging to a specific tenant.

#### POST `/api/admin-tenant/:tenantId/users/:userId/reset-password`

Super admin override to reset a user's password.

#### POST `/api/admin-tenant/:tenantId/users/:userId/reset-2fa`

Super admin override to reset a user's 2FA.

#### POST `/api/admin-tenant/:tenantId/users/:userId/force-logout`

Super admin force-logout a user.

---

### System Admin — Analytics

#### GET `/api/system-admin/analytics`

Get system-wide analytics (ticket counts, tenant stats, user counts).

#### GET `/api/system-admin/tenants/:id/usage`

Get resource usage for a specific tenant.

#### GET `/api/system-admin/tenants/:id/quota-usage`

Get quota usage breakdown (users, departments, tickets, email).

---

### Manager — Tenant Dashboard & Operations

#### GET `/api/manager/settings`

Get current tenant settings (enable_clients, limits, etc.).

#### GET `/api/manager/overview`

Operational overview dashboard (ticket/user/dept stats, recent tickets).

**Response (200):**
```json
{
  "stats": {
    "totalTickets": 150,
    "openTickets": 23,
    "resolvedTickets": 120,
    "totalUsers": 45,
    "totalDepartments": 5
  },
  "recentTickets": [...]
}
```

#### GET `/api/manager/subscription`

Get the current tenant's subscription usage and limits.

**Response (200):**
```json
{
  "plan": "professional",
  "limits": {
    "maxUsers": 50,
    "maxDepartments": 10,
    "maxTicketsPerMonth": 500,
    "maxEmailsPerMonth": 1000
  },
  "usage": {
    "currentUsers": 25,
    "currentDepartments": 4,
    "ticketsThisMonth": 120,
    "emailsThisMonth": 350
  }
}
```

---

### Manager — Employee Management

#### GET `/api/manager/employees`

List employees in the current tenant.

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| role | string | Filter by role |
| status | string | Filter by active/inactive |
| departmentId | uuid | Filter by department |
| search | string | Search by name/email |

#### POST `/api/manager/employees`

Create a user account (manager, employee, or client).

**Request:**
```json
{
  "email": "employee@example.com",
  "fullName": "Jane Smith",
  "role": "soc_analyst",
  "departmentId": "uuid",
  "password": "SecurePass123!"
}
```

#### PUT `/api/manager/employees/:id`

Update an employee's profile.

#### PATCH `/api/manager/employees/:id/status`

Toggle employee active/inactive status.

#### POST `/api/manager/employees/:id/reset-password`

Reset an employee's password.

---

### Manager — Client Management

#### GET `/api/manager/clients`

List external client users in the tenant.

#### POST `/api/manager/clients`

Create an external client account.

#### PATCH `/api/manager/clients/:id/status`

Toggle client active/inactive status.

---

### Manager — Department Management

#### GET `/api/manager/departments`

List departments in the tenant (with user/ticket counts).

#### POST `/api/manager/departments`

Create a new department in the tenant.

**Request:**
```json
{
  "name": "Network Operations",
  "code": "NETOPS",
  "isInternal": false,
  "allowClientCommunication": true
}
```

#### PUT `/api/manager/departments/:id`

Update a department.

#### PATCH `/api/manager/departments/:id/rename`

Rename a department (with audit log).

#### PATCH `/api/manager/users/:id/transfer-department`

Transfer an employee to a different department.

---

### Manager — Ticket Operations

#### POST `/api/manager/tickets/:id/approve-assign`

Approve and assign a ticket to an employee.

#### POST `/api/manager/tickets/:id/reject`

Reject/decline a ticket assignment.

---

### Manager — Platform Support

#### POST `/api/manager/report-issue`

Submit a platform support ticket to System Admin (cross-tenant).

**Request:**
```json
{
  "subject": "Login issue on mobile",
  "description": "Users cannot login on iOS devices",
  "priority": "high"
}
```

#### GET `/api/manager/report-issue/status`

List platform support tickets submitted by the current tenant.

---

### Manager — Email Configuration

#### GET `/api/manager/email-config`

Get the tenant's email integration config.

#### POST `/api/manager/email-config`

Save/update the tenant's email integration config.

**Request:**
```json
{
  "provider": "microsoft",
  "clientId": "azure-app-id",
  "clientSecret": "azure-secret",
  "tenantId": "azure-tenant-id"
}
```

#### POST `/api/manager/email-config/test`

Test the tenant's email integration config.

#### DELETE `/api/manager/email-config`

Delete the tenant's email integration config.

---

### Manager — Invitations

#### POST `/api/manager/invite`

Send a user invitation email.

**Request:**
```json
{
  "email": "newuser@company.com",
  "role": "soc_analyst",
  "departmentId": "uuid"
}
```

#### GET `/api/manager/invitations`

List all invitations for the tenant.

#### DELETE `/api/manager/invitations/:id`

Revoke/cancel a pending invitation.

---

### Manager — Communication Permissions

#### GET `/api/manager/communication-permission`

Get the tenant's communication permission setting.

**Response (200):**
```json
{
  "permission": "BOTH"
}
```
Values: `INTERNAL_ONLY`, `EXTERNAL_ONLY`, `BOTH`

#### PATCH `/api/manager/communication-permission`

Set the tenant's communication permission.

---

### Department Management

#### GET `/api/departments`

List departments (tenant-scoped, for current user).

#### GET `/api/departments/all`

List all departments across tenants (`mssp_admin` only).

#### GET `/api/departments/:id`

Get a specific department by ID.

#### POST `/api/departments`

Create a new department (`mssp_admin`).

#### PATCH `/api/departments/:id`

Update a department (`mssp_admin`).

#### PATCH `/api/departments/:id/toggle`

Activate/deactivate a department (`mssp_admin`).

#### GET `/api/departments/:id/users`

List users in a specific department.

---

### Department Portal

#### POST `/api/department-portal/users`

Department manager creates a `soc_analyst` employee (auto-injects department_id).

**Request:**
```json
{
  "email": "analyst@company.com",
  "fullName": "Analyst Name",
  "password": "SecurePass123!"
}
```

#### GET `/api/department-portal/employee/my-tickets`

Employee views only tickets they created.

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| status | string | Filter by status |
| severity | string | Filter by severity |
| search | string | Search by subject/description |

---

### Tickets

#### GET `/api/tickets`

List/search tickets with filtering and pagination.

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| page | number | Page number (default: 1) |
| limit | number | Items per page (default: 20) |
| status | string | Filter by status |
| severity | string | Filter by severity |
| departmentId | uuid | Filter by department |
| assignedTo | uuid | Filter by assigned user |
| search | string | Search by subject/description |
| type | string | Filter by ticket type |

#### POST `/api/tickets`

Create a new ticket (quota-checked, emits via Socket.IO).

**Request:**
```json
{
  "subject": "Login issue",
  "description": "Cannot access dashboard after password reset",
  "severity": "high",
  "departmentId": "uuid",
  "clientId": "uuid",
  "type": "soc_client"
}
```

**Ticket Types:**
| Type | Description |
|------|-------------|
| `soc_client` | Client-facing ticket (requires clientId) |
| `internal_dept` | Cross-department internal ticket |
| `it_helpdesk` | IT Helpdesk support ticket |

**Response (201):**
```json
{
  "id": "uuid",
  "subject": "Login issue",
  "status": "open",
  "severity": "high",
  "createdAt": "2026-09-06T10:00:00Z"
}
```

---

#### GET `/api/tickets/:id`

Get a specific ticket by ID (tenant-scoped via RLS).

#### POST `/api/tickets/:id/messages`

Add a message/comment to a ticket (emits via Socket.IO).

**Request:**
```json
{
  "message": "We are investigating the issue.",
  "visibility": "internal"
}
```

Visibility values: `internal` (staff only) or `external` (visible to clients).

#### PATCH `/api/tickets/:id/status`

Update a ticket's status.

**Request:**
```json
{
  "status": "in_progress"
}
```

Status values: `open`, `pending_review`, `in_progress`, `pending_approval`, `resolved`, `closed`

#### PATCH `/api/tickets/:id/severity`

Update a ticket's severity (staff only).

**Request:**
```json
{
  "severity": "medium",
  "reason": "Downgraded after initial assessment"
}
```

#### PATCH `/api/tickets/:id/custom-fields`

Update a ticket's custom fields.

#### PATCH `/api/tickets/:id/assign`

Assign/assign-analyst to a ticket.

**Request:**
```json
{
  "assignedTo": "uuid",
  "departmentId": "uuid",
  "note": "Assigning to network team"
}
```

#### POST `/api/tickets/:id/approve`

Approve a ticket assignment (department manager).

#### POST `/api/tickets/:id/reject`

Reject/decline a ticket assignment (department manager).

#### POST `/api/tickets/:id/recall`

Recall/unassign a ticket (emits via Socket.IO).

#### DELETE `/api/tickets/:id`

Soft-delete a ticket (`mssp_admin` only).

---

#### GET `/api/tickets/dashboard/stats`

Get ticket dashboard statistics (counts by status, severity, department).

#### GET `/api/tickets/departments/available`

List departments that accept external/client communication.

#### GET `/api/tickets/routing-departments`

Get departments available for ticket routing.

#### GET `/api/tickets/clients`

List clients (blocked if `enable_clients` is false for tenant).

#### GET `/api/tickets/analysts`

List analysts.

#### GET `/api/tickets/analysts/:userId/clients`

Get clients assigned to a specific analyst.

#### PUT `/api/tickets/analysts/:userId/clients`

Update client assignments for an analyst.

---

#### GET `/api/tickets/:id/audit-logs`

Get audit logs for a specific ticket.

---

### Users

#### GET `/api/users`

List all users with filters (role, status, 2FA, search). System-level accounts hidden from tenant managers.

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| role | string | Filter by role |
| status | string | Filter by active/inactive |
| search | string | Search by name/email |

#### GET `/api/users/client-users`

List client users (for analyst assignment dropdown).

---

### Attachments

#### POST `/api/attachments`

Upload files (max 10 files, 25MB each). Multipart form.

#### GET `/api/attachments/ticket/:ticketId`

Get all attachments for a given ticket.

#### GET `/api/attachments/message/:messageId`

Get all attachments for a given message.

#### GET `/api/attachments/:id`

Download/view a specific attachment by ID.

#### DELETE `/api/attachments/:id`

Delete a specific attachment by ID.

---

### Audit Logs

#### GET `/api/audit`

Get paginated audit logs.

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| entityType | string | Filter by entity type |
| action | string | Filter by action type |
| limit | number | Results per page (default: 50) |
| offset | number | Pagination offset |

---

### Notifications

#### GET `/api/notifications`

Get all notifications for the current user.

#### PATCH `/api/notifications/:id/read`

Mark a single notification as read.

#### PATCH `/api/notifications/read-all`

Mark all notifications as read.

---

### Push Notifications (Web Push)

#### GET `/api/push/vapid-public-key`

Get VAPID public key (public, for service worker registration).

#### POST `/api/push/subscribe`

Register a push subscription.

**Request:**
```json
{
  "endpoint": "https://fcm.googleapis.com/...",
  "keys": {
    "p256dh": "...",
    "auth": "..."
  }
}
```

#### POST `/api/push/unsubscribe`

Unregister a push subscription.

#### GET `/api/push/subscriptions`

List all push subscriptions for the user.

#### GET `/api/push/preferences`

Get push notification preferences.

#### PUT `/api/push/preferences`

Update push notification preferences.

**Request:**
```json
{
  "newTickets": true,
  "ticketUpdates": true,
  "messages": true,
  "assignments": true,
  "severityChanges": true,
  "statusChanges": true
}
```

#### POST `/api/push/test`

Send a test push notification.

#### GET `/api/push/log`

Get push notification delivery log.

---

### Email Quotas

#### GET `/api/email-quotas`

Get all tenant email quotas (`mssp_admin` only).

#### GET `/api/email-quotas/:tenantId`

Get email quota for a single tenant.

#### PUT `/api/email-quotas/:tenantId/limit`

Update the monthly email cap for a tenant (`mssp_admin`).

#### GET `/api/email-quotas/:tenantId/history`

Get paginated email send history for a tenant.

#### POST `/api/email-quotas/:tenantId/reset`

Manually reset the email counter mid-cycle (`mssp_admin`).

---

### Reports

#### GET `/api/reports/dashboard`

Get client dashboard report data.

#### POST `/api/reports/export`

Generate/export a report (PDF or CSV).

---

## WebSocket Events

### Connection

```javascript
const socket = io('https://yourdomain.com', {
  withCredentials: true
});
```

### Client → Server Events

| Event | Payload | Description |
|-------|---------|-------------|
| `join-ticket` | `{ ticketId }` | Subscribe to ticket updates |
| `leave-ticket` | `{ ticketId }` | Unsubscribe from ticket updates |
| `typing` | `{ ticketId }` | User is typing indicator |
| `stop-typing` | `{ ticketId }` | User stopped typing |

### Server → Client Events

| Event | Payload | Description |
|-------|---------|-------------|
| `notification` | `{ id, type, message, ticketId, createdAt }` | New notification received |
| `ticket:created` | `{ ticket }` | New ticket created |
| `ticket:updated` | `{ ticket }` | Ticket status/severity changed |
| `ticket:message` | `{ message, ticketId }` | New message on subscribed ticket |
| `user-online` | `{ userId, fullName }` | User came online |
| `user-offline` | `{ userId }` | User went offline |

---

## Error Response Schema

All error responses follow this format:

```json
{
  "error": "Human-readable error message"
}
```

### HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request (validation error) |
| 401 | Unauthorized (no token / invalid token) |
| 403 | Forbidden (insufficient permissions) |
| 404 | Not Found |
| 409 | Conflict (duplicate resource) |
| 429 | Too Many Requests (rate limited) |
| 500 | Internal Server Error |

## Rate Limits

| Endpoint | Limit | Window |
|----------|-------|--------|
| `/api/auth/login` | 5 requests | 15 minutes |
| `/api/auth/register` | 3 requests | 1 hour |
| `/api/auth/forgot-password` | 3 requests | 1 hour |
| `/api/auth/reset-password` | 3 requests | 1 hour |
| `/api/auth/change-password-forced` | 3 requests | 1 hour |
| `/api/auth/mfa/setup` | 5 requests | 15 minutes |
| `/api/auth/mfa/verify` | 5 requests | 15 minutes |
| `/api/auth/mfa/disable` | 5 requests | 15 minutes |
| `/api/auth/accept-invite` | 5 requests | 15 minutes |
| All `/api/` routes | 100 requests | 15 minutes |
