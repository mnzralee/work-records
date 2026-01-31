# Admin Frontend Testing Plan

## Objective
Verify all admin frontend features are functioning correctly through a 3-phase approach:
1. **Code Review** - Verify implementation completeness
2. **Backend API Testing** - Test endpoints with curl
3. **Live Testing** - Manual browser testing (after phases 1-2 pass)

---

## Phase 1: Code Review

### 1.1 API Hooks to Verify
| Hook File | Features | Status |
|-----------|----------|--------|
| `src/hooks/use-users.ts` | User CRUD, approve, deny, freeze, batch register | |
| `src/hooks/use-admins.ts` | Admin CRUD, role assignment, password reset, MFA | |
| `src/hooks/use-government.ts` | Treasury operations, accounts, funds | |
| `src/hooks/use-supply.ts` | Supply status, pools, country allocations | |
| `src/hooks/use-approvals.ts` | Multi-sig workflows, voting | |
| `src/hooks/use-rbac.ts` | Roles, permissions, matrix | |
| `src/hooks/use-audit-logs.ts` | Audit logging, activity stats | |
| `src/hooks/use-transactions.ts` | Transaction listing, filters | |
| `src/hooks/use-dashboard-stats.ts` | Dashboard metrics | |

### 1.2 Auth Flow to Verify
- `src/stores/auth.ts` - Login, MFA, token refresh logic
- `src/lib/api.ts` - Axios interceptors, error handling
- `src/middleware.ts` - Route protection

### 1.3 Dashboard Pages to Verify
- `/dashboard` - Stats, widgets, recent activity
- `/dashboard/users` - User table, filters, actions
- `/dashboard/admins` - Admin table, create form
- `/dashboard/treasury` - Treasury list, detail
- `/dashboard/government-accounts` - Account hierarchy
- `/dashboard/supply` - Supply status, pools
- `/dashboard/approvals` - Approval list, voting
- `/dashboard/roles` - Roles, permission matrix
- `/dashboard/audit` - Audit logs, filters

---

## Phase 2: Backend API Testing

### 2.1 Pre-requisites
```bash
# Start port-forwards (in separate terminals)
kubectl port-forward -n backend-devnet svc/svc-admin 3033:80
kubectl port-forward -n backend-devnet svc/svc-government 3034:80
```

### 2.2 API Tests to Run
```bash
# 1. Health checks
curl -s http://localhost:3033/health
curl -s http://localhost:3034/health

# 2. Login and get token
TOKEN=$(curl -s -X POST http://localhost:3033/api/v1/admin/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"superowner","password":"Gx5up3r0wn3r2026!"}' \
  | jq -r '.accessToken')

# 3. Test authenticated endpoints
curl -s http://localhost:3033/api/v1/admin/users -H "Authorization: Bearer $TOKEN"
curl -s http://localhost:3033/api/v1/admin/admins -H "Authorization: Bearer $TOKEN"
curl -s http://localhost:3033/api/v1/admin/dashboard/stats -H "Authorization: Bearer $TOKEN"
curl -s http://localhost:3033/api/v1/admin/supply/status -H "Authorization: Bearer $TOKEN"
curl -s http://localhost:3033/api/v1/admin/approvals -H "Authorization: Bearer $TOKEN"
curl -s http://localhost:3033/api/v1/admin/rbac/roles -H "Authorization: Bearer $TOKEN"

# 4. Government API
curl -s http://localhost:3034/api/v1/government/treasury -H "Authorization: Bearer $TOKEN"
```

---

## Phase 3: Live Browser Testing

### 3.1 Pre-requisites
```bash
cd /home/dev-manazir/prod-blockchain/gx-admin-frontend
npm run dev
# Frontend at http://localhost:3000
```

### 3.2 Test Credentials
- **Username:** superowner
- **Password:** Gx5up3r0wn3r2026!

### 3.3 Testing Checklist

#### Authentication
- [ ] Login page loads
- [ ] Login with credentials
- [ ] MFA verification (if enabled)
- [ ] Dashboard redirect
- [ ] Logout works

#### Dashboard
- [ ] Stats cards load
- [ ] Supply meter widget
- [ ] Blockchain status widget
- [ ] Recent activity

#### User Management
- [ ] User list with pagination
- [ ] Status filters (pending, active, frozen)
- [ ] Search functionality
- [ ] User detail page
- [ ] Approve/deny actions
- [ ] Freeze/unfreeze actions

#### Admin Management
- [ ] Admin list
- [ ] Create admin form
- [ ] Admin detail
- [ ] Role assignment

#### Treasury/Government
- [ ] Treasury list
- [ ] Treasury detail
- [ ] Account hierarchy
- [ ] Balance sync

#### Supply
- [ ] Supply status
- [ ] Pool statuses (6 pools)
- [ ] Country allocations

#### Approvals
- [ ] Approval list
- [ ] Pending count badge
- [ ] Create approval
- [ ] Vote on approval

#### RBAC
- [ ] Roles list
- [ ] Permission matrix
- [ ] My permissions

#### Audit
- [ ] Audit log table
- [ ] Filters work
- [ ] Activity stats

---

## Critical Paths

1. **Auth Flow:** Login -> Dashboard -> Logout
2. **User Approval:** View pending -> Approve -> Verify active
3. **Treasury:** List -> Detail -> Sync balance
4. **Approvals:** Create -> Vote -> Verify execution

---

## Key Files
- `src/lib/api.ts` - API client configuration
- `src/stores/auth.ts` - Auth state management
- `src/hooks/use-*.ts` - Feature API hooks
- `next.config.mjs` - API proxy configuration
