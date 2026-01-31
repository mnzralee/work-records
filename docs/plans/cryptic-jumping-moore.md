# Admin KYC Approval Flow - Implementation Plan

**Date:** 2026-01-26
**Work Record:** `docs/workrecords/work-record-2026-01-26.md`

## Executive Summary

The KYC approval workflow is **85% complete**. Both frontend and backend are implemented. The remaining work is:
1. **Verify** yesterday's fixes compile and work
2. **Test** end-to-end flow
3. **Add** audit logging for KYC operations (minor gap)

---

## Current State

| Component | Status | Notes |
|-----------|--------|-------|
| Admin Frontend | COMPLETE | All pages, dialogs, hooks implemented |
| svc-admin Backend | COMPLETE | Endpoints exist, needs testing |
| PrismaUserProfileRepository | JUST ADDED | Created yesterday, needs verification |
| Data Model | COMPLETE | UserProfile, KYCApplication, KYCVerification |

---

## Implementation Phases

### Phase 0: Initialize Work Record (5 min)

**Goal:** Create today's work record for documentation

```bash
# Create work record for 2026-01-26
# Location: /home/dev-manazir/prod-blockchain/docs/workrecords/work-record-2026-01-26.md
```

**Template:**
- Session: Admin KYC Approval Flow Testing
- Objective: Verify and test complete KYC approval workflow
- Update throughout session with findings

---

### Phase 1: Verify Yesterday's Code (15 min)

**Goal:** Ensure TypeScript compiles and svc-admin starts

```bash
# 1. Check TypeScript compilation
cd /home/dev-manazir/prod-blockchain/gx-protocol-backend
npx tsc --noEmit -p apps/svc-admin/tsconfig.json

# 2. Start database port-forward
kubectl port-forward -n devnet svc/postgres-devnet 5432:5432 &

# 3. Start svc-admin locally
export DATABASE_URL="postgresql://gx_admin:rIvz0o0Ib1EIV2RHgp2pONW2CgWgRBWG@localhost:5432/gx_devnet?schema=public"
npx nx serve svc-admin
```

**Files to check if errors:**
- `apps/svc-admin/src/infrastructure/repositories/prisma-user-profile.repository.ts`
- `apps/svc-admin/src/infrastructure/repositories/index.ts`

---

### Phase 2: Backend API Testing (20 min)

**Goal:** Verify all user management endpoints work

```bash
# Login
TOKEN=$(curl -s -X POST http://localhost:3033/api/v1/admin/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"superowner","password":"Gx5up3r0wn3r2026!"}' | jq -r '.accessToken')

# Test endpoints
curl -s http://localhost:3033/api/v1/admin/users -H "Authorization: Bearer $TOKEN" | jq
curl -s "http://localhost:3033/api/v1/admin/users?status=PENDING_ADMIN_APPROVAL" -H "Authorization: Bearer $TOKEN" | jq
curl -s http://localhost:3033/api/v1/admin/users/pending-onchain -H "Authorization: Bearer $TOKEN" | jq
curl -s http://localhost:3033/api/v1/admin/users/frozen -H "Authorization: Bearer $TOKEN" | jq
```

**Expected Results:**
- Users list returns with pagination
- Status filter works
- Empty lists are OK (means no users in that status)

---

### Phase 3: Frontend Testing (20 min)

**Goal:** Verify admin portal displays users correctly

```bash
# Start frontend
cd /home/dev-manazir/prod-blockchain/gx-admin-frontend
npm run dev
```

**Browser Testing at http://localhost:3000:**
1. Login as superowner
2. Navigate to Dashboard > Users
3. Verify tabs: All Users, Pending Approval, Registration Queue, Active, Frozen
4. Click a user to view details
5. Check Documents tab for KYC documents

---

### Phase 4: Create Test User for E2E Flow (30 min)

**Goal:** Have a user in PENDING_ADMIN_APPROVAL status to test approval

**Option A:** Use existing user from wallet registration
**Option B:** Seed a test user directly

```sql
-- Check if any users exist in pending status
SELECT "profileId", email, status FROM "UserProfile"
WHERE status = 'PENDING_ADMIN_APPROVAL' LIMIT 5;
```

If no pending users, we can:
1. Complete a registration via wallet frontend
2. OR update an existing user's status for testing

---

### Phase 5: Test Complete Approval Flow (30 min)

**Test Case 1: Approve KYC**
1. Go to Users > Pending Approval
2. Click user row > View Details
3. Review Documents tab
4. Click "Approve KYC" button
5. Verify: Status changes to APPROVED_PENDING_ONCHAIN, Fabric ID generated

**Test Case 2: Deny KYC**
1. Find another pending user (or reset test user)
2. Click "Deny KYC"
3. Enter reason (min 10 chars)
4. Optionally toggle "Request Resubmission" and select fields
5. Verify: Status changes to DENIED, reason stored

**Test Case 3: Batch Register (if approved users exist)**
1. Go to Registration Queue tab
2. Select users with checkboxes
3. Click "Register on Blockchain"
4. Verify: OutboxCommand created

---

### Phase 6: Add Audit Logging (Optional - 20 min)

**Gap:** KYC operations not logged to audit trail

**File:** `apps/svc-admin/src/services/user-management.service.ts`

Add audit log calls after:
- `approveUser()` - Event: `KYC_APPROVED`
- `denyUser()` - Event: `KYC_REJECTED`
- `freezeUser()` - Event: `ACCOUNT_FROZEN`
- `unfreezeUser()` - Event: `ACCOUNT_UNFROZEN`

---

## Critical Files

| File | Purpose |
|------|---------|
| `svc-admin/src/infrastructure/repositories/prisma-user-profile.repository.ts` | NEW - User queries |
| `svc-admin/src/services/user-management.service.ts` | Business logic |
| `svc-admin/src/routes/admin.routes.ts` | Route definitions |
| `gx-admin-frontend/src/hooks/use-users.ts` | API hooks |
| `gx-admin-frontend/src/app/(main)/dashboard/users/page.tsx` | Users list |
| `gx-admin-frontend/src/app/(main)/dashboard/users/[id]/page.tsx` | User detail |

---

## Verification Checklist

### Backend
- [ ] TypeScript compiles without errors
- [ ] svc-admin starts successfully
- [ ] GET /users returns user list
- [ ] GET /users?status=PENDING_ADMIN_APPROVAL filters correctly
- [ ] POST /users/:id/approve works
- [ ] POST /users/:id/deny works

### Frontend
- [ ] Login works
- [ ] Users page loads all tabs
- [ ] User detail page shows profile and documents
- [ ] Approve dialog submits successfully
- [ ] Deny dialog validates and submits

### E2E
- [ ] Approve flow: PENDING_ADMIN_APPROVAL → APPROVED_PENDING_ONCHAIN
- [ ] Deny flow: PENDING_ADMIN_APPROVAL → DENIED
- [ ] Fabric ID generated on approval

---

## Estimated Time: 2-3 hours

| Phase | Time |
|-------|------|
| Verify code | 15 min |
| Backend API testing | 20 min |
| Frontend testing | 20 min |
| Create test user | 30 min |
| E2E flow testing | 30 min |
| Audit logging | 20 min |
| Buffer for issues | 30 min |
