# tikter — User Roles & Permissions

## Role Hierarchy

```
super_admin (Platform Owner)
    ├── tenant_admin (Service Provider Manager)
    │   └── department_agent (Department Staff Member)
    └── client_admin (End-Client Administrator)
        └── client_user (End-Client User / Submitter)
```

## Role Definitions

### `super_admin` — Platform Owner

Full system access across all tenants and modules. Responsible for platform-wide configuration, tenant provisioning, and global oversight.

| Permission | Access |
|------------|--------|
| Tenant Management | Create, Read, Update, Delete |
| User Management | Create, Read, Update, Delete (all tenants) |
| Department Management | Full CRUD |
| Ticket Management | Full access across all tenants |
| System Settings | Full access |
| Email Configuration | Full access |
| Audit Logs | Full access |
| Analytics | Full access |
| Client Module | Full access |

**Default redirect:** `/admin`

---

### `tenant_admin` — Service Provider Manager

Manages own tenant workspace, including departments, staff members, and client organizations. Scope is limited to assigned tenant.

| Permission | Access |
|------------|--------|
| Own Tenant Users | Create, Read, Update |
| Own Department(s) | Full management |
| Ticket Assignment | Assign to agents/staff within managed departments |
| Ticket Approval | Approve/reject cross-department assignments |
| Reports | Tenant- and department-specific |
| Client Module | If `enable_clients=true` on tenant |

**With `department_id`:** Redirects to `/department/manager`
**Without `department_id`:** Redirects to `/manager`

---

### `department_agent` — Department Staff Member

Handles tickets within assigned department. Works tickets, communicates with clients, and tracks resolution progress.

| Permission | Access |
|------------|--------|
| Assigned Tickets | Read, Update status, Add messages |
| Own Profile | Read, Update |
| Ticket Creation | Yes |
| Ticket Assignment | No (admin only) |
| User Management | No |
| Reports | Own statistics only |

**Default redirect:** `/department/employee`

---

### `client_admin` — End-Client Administrator

Manages client organization users and tickets. Operates within own organization scope.

| Permission | Access |
|------------|--------|
| Own Organization Users | Create, Read, Update |
| Own Organization Tickets | Full CRUD |
| Ticket Creation | Yes |
| Reports | Organization-specific |

**Default redirect:** `/client`

---

### `client_user` — End-Client User / Submitter

Basic ticket submission and tracking. Submits requests and monitors their progress.

| Permission | Access |
|------------|--------|
| Own Tickets | Create, Read |
| Ticket Messages | Add messages to own tickets |
| Profile | Read, Update |

**Default redirect:** `/client`

---

## Permission Matrix

| Feature | super_admin | tenant_admin | department_agent | client_admin | client_user |
|---------|-------------|--------------|------------------|--------------|-------------|
| **Dashboard** | System-wide | Tenant/Dept | Dept only | Org only | Own only |
| **Create Ticket** | Yes | Yes | Yes | Yes | Yes |
| **View All Tickets** | All tenants | Own tenant | Own dept | Own org | Own only |
| **Assign Tickets** | Yes | Own dept | No | Own org | No |
| **Close Tickets** | Yes | Own dept | Own assigned | Own org | No |
| **Delete Tickets** | Yes | No | No | No | No |
| **Create User** | Yes | Own tenant | No | Own org | No |
| **Edit User** | Any user | Own tenant | No | Own org | No |
| **Delete User** | Any user | Own tenant | No | No | No |
| **View Reports** | All | Tenant/Dept | Own stats | Org stats | No |
| **System Settings** | Yes | No | No | No | No |
| **Email Config** | Yes | Yes (own tenant) | No | No | No |
| **Audit Logs** | Yes | No | No | No | No |
| **Manage Tenants** | Yes | No | No | No | No |
| **Client Module** | Yes | If enabled | No | Yes | Yes |

---

## Department Access

### Multi-Department Managers

Tenant admins can be assigned to multiple departments via the `user_departments` junction table.

**Database schema:**
```sql
CREATE TABLE user_departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    department_id UUID NOT NULL REFERENCES departments(id),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, department_id)
);
```

**Access check function:**
```javascript
function hasDeptAccess(user, deptId) {
    if (user.role === 'super_admin') return true;
    if (!user.managed_department_ids) return false;
    return user.managed_department_ids.includes(deptId);
}
```

---

## Tenant Module Visibility

| Module | Controlled By | Default |
|--------|---------------|---------|
| Client Portal | `tenants.enable_clients` | `false` |
| Department Management | Always enabled | — |
| Email Configuration | Always enabled | — |
| Audit Logs | Always enabled | — |

---

## Invitation Workflow

1. Admin/Manager creates invitation via `POST /api/manager/invite`
2. Email sent to invitee via tenant's configured email provider (or system SMTP fallback)
3. Invitee clicks link to `/auth/accept-invite`
4. Account created with assigned role and department(s)
5. Invitation marked as accepted

**Who can invite:**
- `super_admin` — can invite users to any tenant
- `tenant_admin` — can invite users to own tenant

---

## Forced Password Change

Admins can force password change on next login:

```bash
POST /api/admin/force-password-change
{
  "userId": "target_user_uuid"
}
```

**Effect:**
- User's `must_change_password` flag set to `true`
- All existing sessions invalidated
- User redirected to password change form on next login
