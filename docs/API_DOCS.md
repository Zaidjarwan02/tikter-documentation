# tikter — API Documentation

## Base URL

```
Development: http://localhost:4000/api
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

**Request:**
```http
GET /api/auth/me
Cookie: access_token=<jwt>
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
    "departmentId": null,
    "departmentCode": null,
    "departmentName": null,
    "departmentIsInternal": null
  }
}
```

**Error Response (401):**
```json
{ "error": "No token provided." }
```

---

#### POST `/api/auth/logout`

Clear session cookies and revoke refresh token.

**Request:**
```http
POST /api/auth/logout
Cookie: access_token=<jwt>; refresh_token=<jwt>
```

**Response (200):**
```json
{ "message": "Logged out successfully." }
```

---

#### POST `/api/auth/refresh`

Refresh access token using refresh token cookie.

**Request:**
```http
POST /api/auth/refresh
Cookie: refresh_token=<jwt>
```

**Response (200):**
```json
{ "message": "Token refreshed." }
```

**Error Response (401):**
```json
{ "error": "No refresh token provided." }
```

---

#### POST `/api/auth/forgot-password`

Request password reset email with temporary password.

**Request:**
```json
{
  "email": "z.sysadmin@tikter.com"
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

**Response (200):**
```json
{
  "message": "Password reset successfully. Please login with your new password."
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

**Response (200):**
```json
{
  "message": "Password changed successfully."
}
```

---

### Tickets

#### GET `/api/tickets`

List tickets with filtering and pagination.

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| page | number | Page number (default: 1) |
| limit | number | Items per page (default: 20) |
| status | string | Filter by status |
| priority | string | Filter by priority |
| departmentId | uuid | Filter by department |

---

#### POST `/api/tickets`

Create a new ticket.

**Request:**
```json
{
  "subject": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "departmentId": "uuid",
  "clientId": "uuid"
}
```

---

#### GET `/api/tickets/:id`

Get ticket details with messages and attachments.

---

#### PATCH `/api/tickets/:id`

Update ticket status or properties.

---

#### POST `/api/tickets/:id/messages`

Add message to ticket.

---

### Users

#### GET `/api/users`

List users (Admin/Manager only).

#### POST `/api/users`

Create new user (Admin/Manager only).

#### PATCH `/api/users/:id`

Update user properties.

#### DELETE `/api/users/:id`

Soft-delete user (Admin only).

---

### Departments

#### GET `/api/departments`

List all departments.

#### POST `/api/departments`

Create department (Admin/Manager only).

---

### Tenants (Admin only)

#### GET `/api/tenants`

List all tenants.

#### POST `/api/tenants`

Create new tenant.

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
| 429 | Too Many Requests (rate limited) |
| 500 | Internal Server Error |

## Rate Limits

| Endpoint | Limit | Window |
|----------|-------|--------|
| `/api/auth/login` | 5 requests | 15 minutes |
| `/api/auth/register` | 3 requests | 1 hour |
| All `/api/` routes | 100 requests | 15 minutes |
