# RBAC Implementation Guide — Backend-Driven Permission Model

Porting the IndusTowers backend-driven RBAC model into Skill_Bridge, reusing the existing ASP.NET backend API patterns.

---

## Architecture Overview

**Current State (Client-Side Only):**
- Permissions stored in `localStorage` via `PermissionContext`
- Catalog defined in `src/access-control/catalog.js` (hardcoded)
- Custom roles and user overrides persist only in `localStorage`
- Mock data in `peopleMock.js` — no real user data

**Target State (Backend-Driven):**
- Permissions served from backend via `GET /User/me` (or similar) → `user.effective` map
- Full CRUD for roles via role API endpoints
- Full CRUD for user-role assignments via `UserRoleMapping` API
- Catalog from backend `GET /AccessControl/catalog`
- Per-user overrides via backend `PUT /AccessControl/users/{userId}/overrides`
- Audit log via backend `GET /AccessControl/audit`
- Super admin bypass via sentinel `{ "*": true }`

---

## Phase 1: Backend — New API Endpoints Required

The Skill_Bridge backend already has `Role` and `UserRoleMapping` CRUD APIs. The following new endpoints are needed to match the IndusTowers RBAC model.

### 1.1 Access Control Catalog

```
GET /api/AccessControl/catalog?template={org|team}
```

Response shape:
```json
{
  "sections": [
    {
      "key": "userManagement",
      "label": "User Management",
      "columns": ["VIEW", "CREATE", "EDIT", "DELETE", "EXPORT"],
      "features": [
        {
          "key": "users",
          "label": "Users",
          "actions": [
            { "column": "VIEW", "action": "view" },
            { "column": "CREATE", "action": "create" },
            ...
          ]
        }
      ]
    }
  ],
  "dependencies": {
    "pages": [
      { "feature": "users", "requires": ["overview"] },
      { "feature": "courses", "requires": ["users"] }
    ],
    "masterFilters": [
      { "master": "department", "usedBy": ["users", "courses"] }
    ]
  }
}
```

### 1.2 Role CRUD (Extend Existing `/api/Role`)

The existing `roleAPI` has basic CRUD. Extend it to include:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/Role/{id}` | Return role with `templateKey` (standard), `permissions` map, `sections`, `userCount` |
| POST | `/api/Role` | Create custom role. Body: `{ name, cloneFromTemplateKey }` |
| PUT | `/api/Role/{id}/permissions` | Update role permissions. Body: `{ permissions: { section: { feature: { action: bool } } }, sections: { sectionKey: bool } }` |
| PATCH | `/api/Role/{id}` | Rename role. Body: `{ name }` |
| DELETE | `/api/Role/{id}` | Delete custom role. Return 409 if users assigned |

Add new fields to the Role model:
- `templateKey` (string) — `org`, `team`, or null for custom
- `isTemplate` (bool) — `true` for locked standard templates
- `permissions` (JSON) — the section → feature → action permission map
- `sections` (JSON) — section toggle states `{ sectionKey: bool }`
- `userCount` (int, computed) — number of users assigned to this role

### 1.3 User Me — Include Effective Permissions

```diff
- GET /api/User/me  (or /api/unified-auth/me)
+ Return enhanced payload with effective permissions:
```

```json
{
  "id": 1,
  "name": "John",
  "email": "john@example.com",
  "role": "admin",
  "roles": ["admin", "trainer"],
  "primaryRole": "admin",
  "isSuperAdmin": true,
  "effective": {
    "users": { "view": true, "create": true, "edit": true, "delete": false },
    "courses": { "view": true, "create": false, "edit": true, "delete": false }
  },
  "roleObjects": [
    { "id": 1, "name": "Admin", "templateKey": "org", "overridesLocked": false }
  ],
  "overrides": {},
  "roleDefaults": {},
  "sections": {},
  "sectionOverrides": {}
}
```

For super admins:
```json
{ "effective": { "*": true } }
```

### 1.4 User Permission Overrides

```
GET  /api/AccessControl/users/{userId}?roleId={roleId}
PUT  /api/AccessControl/users/{userId}/overrides?roleId={roleId}
```

GET response:
```json
{
  "user": { "id": 1, "name": "John", "employeeId": "EMP001" },
  "assignedRoles": [{ "id": 1, "name": "Admin", "templateKey": "org" }],
  "overrides": {
    "permissions": { "userManagement": { "users": { "create": false } } }
  },
  "sections": { "userManagement": true },
  "roleDefaults": { "userManagement": { "users": { "view": true, "create": true } } },
  "effective": { ... }
}
```

PUT body (delta-based — only changed values):
```json
{
  "permissions": { "userManagement": { "users": { "create": false } } },
  "sections": { "userManagement": false }
}
```

### 1.5 User Role Assignment

```
PUT /api/UserRoleMapping/user/{userId}/roles
Body: { "roleIds": [1, 2, 3] }
```

### 1.6 Audit Log

```
GET /api/AccessControl/audit?page=1&pageSize=50
```

Response:
```json
{
  "items": [
    {
      "id": "uuid",
      "timestamp": "2025-06-26T10:00:00Z",
      "actorId": 1,
      "actorName": "Admin",
      "action": "update_role",
      "targetType": "role",
      "targetId": "2",
      "targetName": "Custom Role",
      "description": "Updated permissions for role 'Custom Role'",
      "severity": "medium"
    }
  ],
  "total": 100,
  "page": 1,
  "pageSize": 50
}
```

---

## Phase 2: Frontend — New API Service Module

Create a dedicated access control API service.

### 2.1 New file: `src/services/accessControlAPI.js`

```javascript
import { apiClient } from '@/utils/api';

export const accessControlAPI = {
  // Catalog
  getCatalog: (template) =>
    apiClient.get('/AccessControl/catalog', { params: { template } }),

  // Roles
  getRoles: () => apiClient.get('/Role'),
  getRole: (id) => apiClient.get(`/Role/${id}`),
  createRole: (data) => apiClient.post('/Role', data),
  updateRolePermissions: (id, body) =>
    apiClient.put(`/Role/${id}/permissions`, body),
  renameRole: (id, name) => apiClient.patch(`/Role/${id}`, { name }),
  deleteRole: (id) => apiClient.delete(`/Role/${id}`),

  // People / Users
  getPeople: (params) =>
    apiClient.get('/AccessControl/people', { params }),
  getUserAccess: (userId, roleId) =>
    apiClient.get(`/AccessControl/users/${userId}`, { params: { roleId } }),
  updateUserOverrides: (userId, body, roleId) =>
    apiClient.put(`/AccessControl/users/${userId}/overrides`, body, { params: { roleId } }),
  updateUserRoles: (userId, roleIds) =>
    apiClient.put(`/UserRoleMapping/user/${userId}/roles`, { roleIds }),

  // Audit
  getAuditLog: (params) =>
    apiClient.get('/AccessControl/audit', { params }),
};
```

### 2.2 Update `src/utils/api.js`

Add the existing `roleAPI` and `userRoleMappingAPI` if not already present (they are present at lines 641-765, verify they export correctly).

---

## Phase 3: Frontend — Auth Provider Changes

### 3.1 Update `src/context/AuthContext.jsx`

Add `user.effective` permission map handling:

```diff
// After login success, or on app mount via /me endpoint:
+ // Normalize user shape including effective permissions
+ const normalizeUser = (raw) => {
+   const roles = raw.roles || [];
+   if (raw.isSuperAdmin && !roles.includes('super_admin')) {
+     roles.push('super_admin');
+   }
+   return {
+     ...raw,
+     roles,
+     primaryRole: raw.primaryRole || raw.role,
+     effective: raw.effective || {},
+     roleObjects: raw.roleObjects || [],
+     overrides: raw.overrides || {},
+     roleDefaults: raw.roleDefaults || {},
+   };
+ };
```

```diff
// Add to context value:
+ refreshPermissions: async (roleId) => {
+   try {
+     const res = await authAPI.me();
+     if (res.data.success) {
+       const normalized = normalizeUser(res.data.data);
+       setUser(normalized);
+       localStorage.setItem('user', JSON.stringify(normalized));
+     }
+   } catch (err) {
+     console.error('Failed to refresh permissions:', err);
+   }
+ },
+ switchRole: (nextRole) => {
+   setUser((prev) => ({
+     ...prev,
+     primaryRole: nextRole,
+     role: nextRole,
+     _chipPicked: true,
+   }));
+   // Optionally refresh permissions from server with new role context
+ },
```

### 3.2 Add `user.effective` to the auth API login response

The login endpoint (`POST /api/unified-auth/login-with-cookies` or `POST /api/unified-auth/login-with-otp`) should return the `effective` permission map in the user object.

---

## Phase 4: Frontend — New Permission Hook (Backend-Driven)

The existing `usePermission` hook in `src/access-control/usePermission.js` resolves from `PermissionContext` (localStorage). Create a parallel hook that reads from `user.effective` (backend-driven).

### 4.1 New file: `src/hooks/useEffectivePermission.js`

```javascript
import { useAuth } from '@/context/AuthContext';

export function useEffectivePermission() {
  const { user } = useAuth();

  const effective = user?.effective || {};
  const isSuperAdmin = effective['*'] === true;
  const roles = user?.roles || [];
  const isServerSideSuperAdmin =
    isSuperAdmin ||
    roles.includes('super_admin') ||
    user?.isSuperAdmin === true;
  const activeRole = user?.primaryRole || user?.role;

  // True if the user's active chip is Super Admin (bypass all checks)
  const actingAsSuperAdmin = isServerSideSuperAdmin && activeRole === 'super_admin';

  function can(featureKey, action = 'view') {
    if (actingAsSuperAdmin) return true;
    if (isSuperAdmin) return true;
    const feature = effective[featureKey];
    if (!feature) return false;
    return feature[action] === true;
  }

  function canAny(featureKey, actions = []) {
    return actions.some((action) => can(featureKey, action));
  }

  function canFilter(masterKey) {
    return can(masterKey, 'view') || can(masterKey, 'dropdown');
  }

  function canSection(featureKeys = []) {
    return featureKeys.some((key) => can(key, 'view'));
  }

  function isSectionEnabled(sectionKey) {
    // Sections are enabled/disabled via section toggles in the backend
    return actingAsSuperAdmin || (user?.sections?.[sectionKey] !== false);
  }

  return {
    can,
    canAny,
    canFilter,
    canSection,
    isSectionEnabled,
    isSuperAdmin: actingAsSuperAdmin,
    isServerSideSuperAdmin,
    effective,
    activeRole,
    actingAsSuperAdmin,
  };
}
```

### 4.2 Migration Strategy

Use a feature flag approach — components can gradually switch:

```javascript
// In any component:
const { can } = useEffectivePermission(); // backend-driven (new)
// vs
const permission = usePermission('users', 'create'); // localStorage (old)
```

Replace the `PermissionGate` equivalent:

### 4.3 New component: `src/components/common/PermissionGate.jsx`

```jsx
import { useEffectivePermission } from '@/hooks/useEffectivePermission';

export function PermissionGate({ feature, action, anyOf, filter, fallback = null, children }) {
  const { can, canAny, canFilter } = useEffectivePermission();

  let granted = false;
  if (filter) {
    granted = canFilter(filter);
  } else if (anyOf) {
    granted = canAny(feature, anyOf);
  } else {
    granted = can(feature, action);
  }

  return granted ? children : fallback;
}
```

---

## Phase 5: Frontend — Route Guards (Role-Based)

### 5.1 Update `src/components/ProtectedRoute.jsx`

The existing guards check `user.role` or `user.roleName` for a simple string match. Keep them but enhance to support the `effective` permission model for finer-grained access within roles.

Key change — use `primaryRole` from the normalized user shape:

```diff
- const role = (user?.role || user?.roleName || '').toLowerCase();
+ const role = (user?.primaryRole || user?.role || user?.roleName || '').toLowerCase();
```

### 5.2 Add a `RoleRoute` guard (like IndusTowers)

For routes that multiple roles can access but need permission gating:

Add to `src/components/ProtectedRoute.jsx`:

```javascript
export const RoleRoute = ({ allowedRoles = [], children }) => {
  const { isAuthenticated, user, loading } = useAuth();
  const location = useLocation();

  if (loading) return <LoadingSpinner />;
  if (!isAuthenticated) return <Navigate to={getLoginPath()} state={{ from: location }} replace />;

  const userRole = (user?.primaryRole || user?.role || user?.roleName || '').toLowerCase();
  const hasAccess = allowedRoles.map(r => r.toLowerCase()).includes(userRole);

  if (!hasAccess) return <Navigate to={getDashboardByRole(user)} replace />;
  return children;
};
```

---

## Phase 6: Frontend — Connect Access Control UI to Backend APIs

### 6.1 Update `src/context/PermissionContext.jsx`

The existing context stores everything in `localStorage`. Modify it to:

1. **Fetch catalog from backend** on mount (fall back to local `catalog.js`)
2. **Fetch roles from backend** (merge with local `customRoles` for now)
3. **Fetch people from backend** (replace `peopleMock.js`)
4. **Save/load overrides to/from backend** instead of `localStorage`

Key changes:

```diff
- const initial = loadStore();
+ const [catalog, setCatalog] = useState(null);
+ const [roles, setRoles] = useState([]);
+ const [people, setPeople] = useState([]);
+ const [loading, setLoading] = useState(true);

+ useEffect(() => {
+   const init = async () => {
+     try {
+       const [catalogRes, rolesRes] = await Promise.all([
+         accessControlAPI.getCatalog('org'),
+         accessControlAPI.getRoles(),
+       ]);
+       setCatalog(catalogRes.data.data);
+       setRoles(rolesRes.data.data);
+     } catch (err) {
+       console.error('Failed to load AC data, falling back to local:', err);
+       // Fall back to catalog.js defaults
+     } finally {
+       setLoading(false);
+     }
+   };
+   init();
+ }, []);
```

### 6.2 Update Access Control Pages

The UI pages in `src/pages/Admin/AccessControl/` already have the right structure. They need to:

| Tab | Current Source | Change To |
|-----|---------------|-----------|
| `RolesTab.jsx` | `PermissionContext` (localStorage) | Fetch roles from backend `roleAPI.getAll()` |
| `PeopleTab.jsx` | `peopleMock.js` | Fetch from backend `accessControlAPI.getPeople()` |
| `PermissionMatrix.jsx` | `CATALOG_DEFAULTS` | Use catalog from backend API |
| `RoleEditor` | `customRoles` in localStorage | `roleAPI.getById(id)` + `roleAPI.updateRolePermissions()` |
| `NewRoleModal` | `createCustomRole` (localStorage) | `roleAPI.create()` |

### 6.3 Specific Page Changes

#### RolesTab.jsx
```diff
- const { customRoles, createCustomRole, deleteCustomRole, getMatrix } = usePermissionStore();
+ const [roles, setRoles] = useState([]);
+ const [loading, setLoading] = useState(true);

+ useEffect(() => {
+   accessControlAPI.getRoles()
+     .then(res => setRoles(res.data.data))
+     .finally(() => setLoading(false));
+ }, []);
```

#### PeopleTab.jsx
```diff
- import peopleMock from './peopleMock';
+ useEffect(() => {
+   accessControlAPI.getPeople({ q, role, page, pageSize })
+     .then(res => setPeople(res.data.data.items))
+ }, [q, role, page]);
```

#### User Access Editor
```diff
- const { saveUserOverrides, resetUserOverrides } = usePermissionStore();
+ const handleSave = async () => {
+   await accessControlAPI.updateUserOverrides(userId, { permissions: diffs });
+ };
```

---

## Phase 7: Frontend — Sidebar Permission Filtering

### 7.1 Update Sidebar (`src/components/Admin/Sidebar/AdminSidebar.jsx`)

The sidebar currently shows all items based on role. Add permission-based filtering:

```jsx
import { useEffectivePermission } from '@/hooks/useEffectivePermission';

function AdminSidebar() {
  const { can, isSectionEnabled } = useEffectivePermission();

  const menuSections = [
    {
      key: 'dashboard',
      title: 'Dashboard',
      items: [
        { label: 'Dashboard', icon: LayoutDashboard, path: '/admin/dashboard', feature: 'dashboard' },
      ],
    },
    {
      key: 'userManagement',
      title: 'User Management',
      items: [
        { label: 'Users', icon: Users, path: '/admin/users', feature: 'users' },
        { label: 'Assigned Users', icon: UserCheck, path: '/admin/assigned-users', feature: 'assignedUsers' },
        { label: 'Access Control', icon: ShieldCheck, path: '/admin/access-control', feature: 'accessControl' },
      ],
    },
    // ... other sections
  ];

  const filteredSections = menuSections
    .filter(section => isSectionEnabled(section.key))
    .map(section => ({
      ...section,
      items: section.items.filter(item => can(item.feature, 'view')),
    }))
    .filter(section => section.items.length > 0);

  // ... render filteredSections
}
```

---

## Phase 8: Frontend — Component-Level Gating

### 8.1 Add PermissionGate Usage in Admin Pages

In any component where actions should be hidden based on permissions:

```jsx
import { PermissionGate } from '@/components/common/PermissionGate';

// Hide "Add User" button from users who cannot create
<PermissionGate feature="users" action="create">
  <Button onClick={openAddModal}>Add User</Button>
</PermissionGate>

// Hide action column if user cannot do any of these
<PermissionGate feature="users" anyOf={['edit', 'delete']}>
  <ActionsColumn />
</PermissionGate>

// Hide export button
<PermissionGate feature="users" action="export">
  <Button>Export</Button>
</PermissionGate>
```

### 8.2 Inline Permission Checks

For menus and action items:

```jsx
const { can } = useEffectivePermission();

const menuItems = [
  can('users', 'view') && { icon: Eye, label: 'View', onClick: viewUser },
  can('users', 'edit') && { icon: Pencil, label: 'Edit', onClick: editUser },
  can('users', 'delete') && { icon: Trash2, label: 'Delete', onClick: deleteUser },
].filter(Boolean);
```

---

## Phase 9: Frontend — Role Switcher

### 9.1 Update Role Switcher

The existing `roleSwitcherAccordion.jsx` in the sidebar works but is tied to the old role model. Enhance it to use the new user shape:

```diff
- const roles = user?.roles || [];
+ const roles = user?.roleObjects?.map(r => ({
+   key: r.templateKey || 'custom',
+   id: r.id,
+   label: r.name,
+ })) || [];
```

### 9.2 Add Header Role Badge (like IndusTowers)

Show the current active role as a badge in the navbar:

```jsx
import { useAuth } from '@/context/AuthContext';

function RoleBadge() {
  const { user } = useAuth();
  const activeRole = user?.primaryRole || user?.role;

  return (
    <span className="px-2 py-1 text-xs font-medium rounded-full bg-primary-50 text-primary-700">
      {activeRole?.charAt(0).toUpperCase() + activeRole?.slice(1)}
    </span>
  );
}
```

---

## Phase 10: Migration Path

### Step-by-Step Rollout

| Step | Files to Change | Description |
|------|----------------|-------------|
| 1 | Backend | Add `AccessControl` controller with catalog, users, overrides, audit endpoints |
| 2 | `src/services/accessControlAPI.js` | Create the API service module |
| 3 | `src/context/AuthContext.jsx` | Add `normalizeUser()`, `effective` map, `switchRole()`, `refreshPermissions()` |
| 4 | `src/hooks/useEffectivePermission.js` | Create backend-driven permission hook |
| 5 | `src/components/common/PermissionGate.jsx` | Create gate component using `useEffectivePermission` |
| 6 | `src/context/PermissionContext.jsx` | Add backend data fetching (catalog, roles, people) |
| 7 | Admin pages (RolesTab, PeopleTab, etc.) | Replace localStorage calls with API calls |
| 8 | Sidebar (`AdminSidebar.jsx`) | Add permission filtering |
| 9 | Individual admin pages | Add `PermissionGate` and `can()` checks |
| 10 | Role Switcher | Update to use backend-driven role model |

### Feature Flag Strategy

Keep both systems running in parallel behind a flag:

```javascript
// In .env or a config
const USE_BACKEND_RBAC = import.meta.env.VITE_USE_BACKEND_RBAC === 'true';

// In components
const { can } = USE_BACKEND_RBAC
  ? useEffectivePermission()
  : { can: (f, a) => usePermission(f, a) };
```

---

## Appendix: Key File References

### IndusTowers (Reference Implementation)
| File | Purpose |
|------|---------|
| `src/hooks/usePermission.js` | Backend-driven permission check hook |
| `src/components/common/PermissionGate.jsx` | Declarative UI gate component |
| `src/app/router/ProtectedRoute.jsx` | Route guards (RoleRoute, AdminRoute, etc.) |
| `src/app/providers/AuthProvider.jsx` | User shape normalization, `refreshPermissions()`, `switchRole()` |
| `src/hooks/useActiveRolePermissions.js` | Super admin impersonation role data |
| `src/components/layout/RoleSwitcher.jsx` | Navbar role chip selector |
| `src/services/api/modules/accessControl.js` | AC API service module |
| `src/constants/roles.js` | Role constants and helpers |
| `src/app/router/AppRouter.jsx` | Full route tree with guards |

### Skill_Bridge (Existing Implementation)
| File | Purpose |
|------|---------|
| `src/access-control/catalog.js` | Client-side permission catalog |
| `src/access-control/resolver.js` | Three-layer permission resolver |
| `src/access-control/usePermission.js` | Current permission hook (localStorage) |
| `src/context/PermissionContext.jsx` | Current store (localStorage) |
| `src/components/ProtectedRoute.jsx` | Current route guards |
| `src/pages/Admin/AccessControl/` | Existing AC UI pages |
| `src/utils/api.js` | API client with `roleAPI`, `userRoleMappingAPI` |
| `src/context/AuthContext.jsx` | Current auth context |

---

## Data Flow Summary

```
Backend API                       Frontend
===========                       ========

POST /api/unified-auth/login ──>  AuthContext.login()
  └─ user.effective map            └─ normalizeUser()
                                    └─ store in localStorage + state

GET /api/unified-auth/me ──────>  AuthContext.refreshPermissions()
  └─ updated effective             └─ update user state

GET /api/AccessControl/people ──>  PeopleTab
GET /api/Role ─────────────────>  RolesTab
GET /api/Role/{id} ────────────>  RoleEditor
PUT /api/Role/{id}/permissions ──> Save role permissions
PUT /api/AccessControl/users/{id}/overrides ──> Save user overrides

Component Permission Check:
  useEffectivePermission().can('users', 'create')
    └─ reads user.effective['users']['create']
    └─ gates UI via PermissionGate or inline conditional
```
