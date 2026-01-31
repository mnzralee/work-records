# GX Protocol: Admin Users Page - Onchain User Management

> **Date**: 2026-01-27
> **Feature**: Admin Dashboard Users Page Redesign
> **Status**: PLANNING

---

## Executive Summary

Redesign the admin `/dashboard/users` page to focus on **onchain users only** (users with blockchain accounts). Users in KYC process stages are managed via the KYC Review page.

### Key Changes

1. **Scope**: Only show users with blockchain accounts (APPROVED_PENDING_ONCHAIN and beyond)
2. **Tabs**: All, Pending Onchain, Active, Suspended, Locked, Deactivated, Banned
3. **Actions**: Suspend/Lock/Deactivate/Ban with blockchain synchronization
4. **Batch Registration**: Register approved users to Hyperledger Fabric

---

## Status Definitions (UPDATED)

### UserProfileStatus Enum (Prisma)

```prisma
enum UserProfileStatus {
  // Pre-Onchain (KYC Flow - managed in KYC Page)
  REGISTERED                  // Initial registration complete
  PENDING_ADMIN_APPROVAL      // KYC submitted, awaiting review
  APPROVED_PENDING_ONCHAIN    // KYC approved, waiting blockchain registration

  // Onchain Statuses (managed in Users Page - SYNC with blockchain)
  ACTIVE                      // Normal operations - full access
  SUSPENDED                   // Admin-initiated temporary (receive-only, other features locked)
  LOCKED                      // Security incident (password breach, etc.) - receive-only
  DEACTIVATED                 // User-initiated deactivation from system
  BANNED                      // Permanently banned by admin for ToS breach
}
```

### Status Details

| Status | Initiated By | Restrictions | Reversible | Chaincode Status |
|--------|--------------|--------------|------------|------------------|
| `ACTIVE` | System (after registration) | None - full access | N/A | `"Active"` |
| `SUSPENDED` | Admin | Receive-only, other features locked | Yes (by admin) | `"Suspended"` |
| `LOCKED` | System/Admin | Receive-only, security lockdown | Yes (by admin after verification) | `"Locked"` |
| `DEACTIVATED` | User | Account dormant, can reactivate | Yes (by user) | `"Deactivated"` |
| `BANNED` | Admin | Permanent - no access | **No** (terminal) | `"Banned"` |

### Blockchain Sync Requirements

**Onchain statuses MUST be in sync** - when admin changes status in database, chaincode must be called to update on-chain status.

| Action | Offchain (DB) | Onchain (Chaincode) |
|--------|---------------|---------------------|
| Suspend User | status → SUSPENDED | `SetUserStatus(userId, "Suspended")` |
| Lock User | status → LOCKED | `SetUserStatus(userId, "Locked")` |
| User Deactivates | status → DEACTIVATED | `SetUserStatus(userId, "Deactivated")` |
| Ban User | status → BANNED | `SetUserStatus(userId, "Banned")` |
| Unsuspend | status → ACTIVE | `SetUserStatus(userId, "Active")` |
| Unlock | status → ACTIVE | `SetUserStatus(userId, "Active")` |
| Reactivate | status → ACTIVE | `SetUserStatus(userId, "Active")` |

---

## Chaincode Changes Required

### Current Chaincode Status (identity_types.go)

```go
type User struct {
    Status string `json:"status"` // Currently: "Active", "Frozen"
}
```

### New Chaincode Statuses Needed

```go
const (
    UserStatusActive      = "Active"      // Full access
    UserStatusSuspended   = "Suspended"   // Receive-only (admin temp)
    UserStatusLocked      = "Locked"      // Receive-only (security)
    UserStatusDeactivated = "Deactivated" // User-initiated dormant
    UserStatusBanned      = "Banned"      // Permanent ban
)
```

### New Chaincode Function Needed

**Replace** `FreezeWallet`/`UnfreezeWallet` with generic `SetUserStatus`:

```go
// SetUserStatus - Update user's account status
// Access: gx_super_admin only
func (s *IdentityContract) SetUserStatus(
    ctx contractapi.TransactionContextInterface,
    userID string,
    newStatus string,
    reason string,
) error
```

**Status Transition Rules in Chaincode:**
- `Active` → `Suspended`, `Locked`, `Deactivated`, `Banned`
- `Suspended` → `Active`, `Banned`
- `Locked` → `Active`, `Banned`
- `Deactivated` → `Active`, `Banned`
- `Banned` → (no transitions - terminal)

### Transaction Restrictions by Status

| Status | Can Send | Can Receive | Can Login | Can View |
|--------|----------|-------------|-----------|----------|
| Active | ✅ | ✅ | ✅ | ✅ |
| Suspended | ❌ | ✅ | ✅ | ✅ |
| Locked | ❌ | ✅ | ❌ | ❌ |
| Deactivated | ❌ | ✅ | ✅ | ✅ |
| Banned | ❌ | ❌ | ❌ | ❌ |

### Existing Endpoints (svc-admin)

| Endpoint | Method | Status |
|----------|--------|--------|
| `GET /users` | List users | ✅ Exists |
| `GET /users/:id` | Get user | ✅ Exists |
| `POST /users/:id/freeze` | Freeze user | ⚠️ Rename to suspend |
| `POST /users/:id/unfreeze` | Unfreeze user | ⚠️ Rename to unsuspend |
| `POST /users/batch-register` | Batch register | ✅ Exists |
| `GET /users/pending-onchain` | List pending | ✅ Exists |
| `POST /users/:id/suspend` | Suspend user | ❌ NEEDED |
| `POST /users/:id/lock` | Lock user (security) | ❌ NEEDED |
| `POST /users/:id/unlock` | Unlock user | ❌ NEEDED |
| `POST /users/:id/ban` | Ban user | ❌ NEEDED |
| `POST /users/:id/deactivate` | User deactivates | ❌ NEEDED (wallet API) |
| `POST /users/:id/reactivate` | User reactivates | ❌ NEEDED (wallet API) |

---

## Implementation Plan

### Track 0: Chaincode - New User Status System

#### WP-0.1: Update User Status Constants

**File:** `gx-coin-fabric/chaincode/gxtv3/consts.go`

**Add:**
```go
// User Status Constants
const (
    UserStatusActive      = "Active"
    UserStatusSuspended   = "Suspended"
    UserStatusLocked      = "Locked"
    UserStatusDeactivated = "Deactivated"
    UserStatusBanned      = "Banned"
)
```

#### WP-0.2: Add SetUserStatus Function

**File:** `gx-coin-fabric/chaincode/gxtv3/identity_contract.go`

**New Function:**
```go
// SetUserStatus updates a user's account status
// @param userID - Fabric user ID
// @param newStatus - Target status (Active, Suspended, Locked, Deactivated, Banned)
// @param reason - Reason for status change (required for non-Active)
// Access: gx_super_admin only
func (s *IdentityContract) SetUserStatus(
    ctx contractapi.TransactionContextInterface,
    userID string,
    newStatus string,
    reason string,
) error {
    // 1. Check admin role
    // 2. Validate status is allowed
    // 3. Load user, validate transition is allowed
    // 4. Update status
    // 5. Emit UserStatusChanged event
}
```

#### WP-0.3: Update Transfer Validation

**File:** `gx-coin-fabric/chaincode/gxtv3/tokenomics_contract.go`

**Update `Transfer` function to check status:**
```go
// In Transfer():
if sender.Status == UserStatusSuspended ||
   sender.Status == UserStatusLocked ||
   sender.Status == UserStatusDeactivated {
    return fmt.Errorf("account is restricted: %s", sender.Status)
}
if sender.Status == UserStatusBanned {
    return fmt.Errorf("account is banned")
}
// Receiver can still receive for Suspended/Locked/Deactivated
if receiver.Status == UserStatusBanned {
    return fmt.Errorf("recipient account is banned")
}
```

---

### Track A: Backend - Status Management Endpoints

#### WP-A1: Update Prisma Schema

**File:** `db/prisma/schema/identity/enums.prisma`

**Change:**
```prisma
enum UserProfileStatus {
  REGISTERED
  PENDING_ADMIN_APPROVAL
  APPROVED_PENDING_ONCHAIN
  ACTIVE
  SUSPENDED   // Admin temp - receive only
  LOCKED      // Security - receive only
  DEACTIVATED // User initiated
  BANNED      // Permanent admin ban
}
```

**Remove:** `FROZEN`, `CLOSED`

#### WP-A2: Add Suspend/Unsuspend Endpoints

**Files:**
- `apps/svc-admin/src/application/use-cases/user-management/suspend-user.use-case.ts` (NEW)
- `apps/svc-admin/src/application/use-cases/user-management/unsuspend-user.use-case.ts` (NEW)

**Endpoint:** `POST /api/v1/admin/users/:id/suspend`
```typescript
{
  reason: 'COMPLIANCE_REVIEW' | 'SUSPICIOUS_ACTIVITY' | 'PENDING_INVESTIGATION' | 'COURT_ORDER',
  notes?: string
}
```

**Endpoint:** `POST /api/v1/admin/users/:id/unsuspend`
```typescript
{
  notes?: string
}
```

**Logic:**
1. Validate user status transition is allowed
2. Update UserProfile.status
3. Create outbox command: `SET_USER_STATUS` → chaincode `SetUserStatus()`

#### WP-A3: Add Lock/Unlock Endpoints

**Files:**
- `apps/svc-admin/src/application/use-cases/user-management/lock-user.use-case.ts` (NEW)
- `apps/svc-admin/src/application/use-cases/user-management/unlock-user.use-case.ts` (NEW)

**Endpoint:** `POST /api/v1/admin/users/:id/lock`
```typescript
{
  reason: 'PASSWORD_BREACH' | 'SUSPICIOUS_LOGIN' | 'DEVICE_COMPROMISED' | 'USER_REQUEST',
  notes?: string
}
```

**Endpoint:** `POST /api/v1/admin/users/:id/unlock`
```typescript
{
  verificationMethod: 'IDENTITY_VERIFIED' | 'DEVICE_RESET' | 'ADMIN_OVERRIDE',
  notes?: string
}
```

#### WP-A4: Add Ban Endpoint

**File:** `apps/svc-admin/src/application/use-cases/user-management/ban-user.use-case.ts` (NEW)

**Endpoint:** `POST /api/v1/admin/users/:id/ban`
```typescript
{
  reason: 'FRAUD' | 'SANCTIONS_VIOLATION' | 'TERMS_VIOLATION' | 'COURT_ORDER' | 'ILLEGAL_ACTIVITY',
  notes: string, // Required
  evidence?: string[] // Optional: document IDs
}
```

**Note:** No unban endpoint - BANNED is terminal.

#### WP-A5: Add Deactivate/Reactivate Endpoints (Wallet Frontend)

**Files:**
- `apps/svc-wallet/src/application/use-cases/deactivate-account.use-case.ts` (NEW)
- `apps/svc-wallet/src/application/use-cases/reactivate-account.use-case.ts` (NEW)

**Wallet Endpoint:** `POST /api/v1/wallet/account/deactivate`
**Wallet Endpoint:** `POST /api/v1/wallet/account/reactivate`

User-initiated - requires authentication and confirmation.

#### WP-A6: Update User Listing Filters

**Files:**
- `apps/svc-admin/src/application/use-cases/user-management/list-users.use-case.ts`
- `apps/svc-admin/src/infrastructure/repositories/prisma-user-profile.repository.ts`

**Changes:**
1. Add status filter for all new statuses
2. Exclude REGISTERED, PENDING_ADMIN_APPROVAL, DENIED from users endpoint
3. Add counts per status for tabs

---

### Track B: Admin Frontend - Users Page Redesign

#### WP-B1: Update Users Page Tab Structure

**File:** `gx-admin-frontend/src/app/(main)/dashboard/users/page.tsx`

**New Tabs:**
```typescript
const TABS = [
  { id: 'all', label: 'All Users', filter: ['APPROVED_PENDING_ONCHAIN', 'ACTIVE', 'SUSPENDED', 'LOCKED', 'DEACTIVATED', 'BANNED'] },
  { id: 'pending-onchain', label: 'Pending Onchain', filter: ['APPROVED_PENDING_ONCHAIN'], badge: true },
  { id: 'active', label: 'Active', filter: ['ACTIVE'] },
  { id: 'suspended', label: 'Suspended', filter: ['SUSPENDED'] },
  { id: 'locked', label: 'Locked', filter: ['LOCKED'] },
  { id: 'deactivated', label: 'Deactivated', filter: ['DEACTIVATED'] },
  { id: 'banned', label: 'Banned', filter: ['BANNED'] },
];
```

#### WP-B2: Enhance Pending Onchain Tab with Batch Registration

**File:** `gx-admin-frontend/src/app/(main)/dashboard/users/_components/pending-onchain-panel.tsx`

**Users Shown:** `APPROVED_PENDING_ONCHAIN` status (KYC approved, awaiting blockchain registration)
**Note:** These users do NOT have fabricId yet - it's generated during batch registration.

**Features:**
1. Checkbox selection for individual users
2. "Select All" checkbox in header
3. "Register Selected" button (when items selected)
4. "Register All" button
5. Loading states during batch operation
6. Success/error notifications
7. Progress indicator for batch operations
8. After registration: fabricId is generated and displayed

**Batch Registration Process:**
- Calls backend endpoint to generate fabricId using `generateFabricUserId(country, dob, gender, '0')`
- Creates CREATE_USER outbox command for each user
- Outbox submitter processes commands → blockchain registration
- Projector updates user status to ACTIVE with fabricId visible

#### WP-B3: Update User Actions for New Statuses

**Files:**
- `gx-admin-frontend/src/app/(main)/dashboard/users/_components/user-action-dialogs.tsx`
- `gx-admin-frontend/src/app/(main)/dashboard/users/_components/users-table.tsx`

**New/Updated Dialogs:**
1. **SuspendUserDialog** - Reason dropdown, notes
2. **UnsuspendUserDialog** - Confirmation, notes
3. **LockUserDialog** - Security reason dropdown, notes
4. **UnlockUserDialog** - Verification method, notes
5. **BanUserDialog** - Reason dropdown, **required** notes, confirmation warning

**Action Menu by Status:**
| User Status | Available Actions |
|-------------|-------------------|
| APPROVED_PENDING_ONCHAIN | Register Onchain |
| ACTIVE | Suspend, Lock, Ban, View Details |
| SUSPENDED | Unsuspend, Ban, View Details |
| LOCKED | Unlock, Ban, View Details |
| DEACTIVATED | Ban, View Details |
| BANNED | View Details only (no actions) |

#### WP-B4: Update User Status Badge

**File:** `gx-admin-frontend/src/app/(main)/dashboard/users/_components/user-status-badge.tsx`

**Status Colors:**
| Status | Color | Icon | Description |
|--------|-------|------|-------------|
| APPROVED_PENDING_ONCHAIN | Blue | Clock | Awaiting blockchain |
| ACTIVE | Green | CheckCircle | Full access |
| SUSPENDED | Orange | Pause | Admin temp restriction |
| LOCKED | Red/Orange | Lock | Security lockdown |
| DEACTIVATED | Gray | UserMinus | User-initiated dormant |
| BANNED | Red | Ban | Permanently banned |

#### WP-B5: Update API Hooks

**File:** `gx-admin-frontend/src/hooks/use-users.ts`

**New Hooks:**
```typescript
useSuspendUser()    // POST /users/:id/suspend
useUnsuspendUser()  // POST /users/:id/unsuspend
useLockUser()       // POST /users/:id/lock
useUnlockUser()     // POST /users/:id/unlock
useBanUser()        // POST /users/:id/ban
```

**Update Existing:**
- Add status filter to `useUsers()`
- Add status counts endpoint
- Update mutation query invalidation

#### WP-B6: Update Types

**File:** `gx-admin-frontend/src/types/user.ts`

**Updates:**
```typescript
type UserStatus =
  | 'REGISTERED'
  | 'PENDING_ADMIN_APPROVAL'
  | 'APPROVED_PENDING_ONCHAIN'
  | 'ACTIVE'
  | 'SUSPENDED'
  | 'LOCKED'
  | 'DEACTIVATED'
  | 'BANNED';

type SuspendReason = 'COMPLIANCE_REVIEW' | 'SUSPICIOUS_ACTIVITY' | 'PENDING_INVESTIGATION' | 'COURT_ORDER';
type LockReason = 'PASSWORD_BREACH' | 'SUSPICIOUS_LOGIN' | 'DEVICE_COMPROMISED' | 'USER_REQUEST';
type BanReason = 'FRAUD' | 'SANCTIONS_VIOLATION' | 'TERMS_VIOLATION' | 'COURT_ORDER' | 'ILLEGAL_ACTIVITY';
```

---

### Track C: KYC Page Enhancement - Show FabricId

#### WP-C1: Add FabricId Column to Approved Tab

**File:** `gx-admin-frontend/src/app/(main)/dashboard/kyc/_components/kyc-applications-table.tsx`

**Changes:**
1. Add `fabricUserId` column to table (only for APPROVED status)
2. Show "Pending Registration" badge if no fabricId yet (APPROVED_PENDING_ONCHAIN status)
3. Show formatted fabricId when available (after batch registration)

**IMPORTANT: FabricId Generation Flow:**
```
KYC Approval (svc-admin)
├── Status: PENDING_ADMIN_APPROVAL → APPROVED_PENDING_ONCHAIN
├── fabricUserId: NULL (not generated yet!)
└── User appears in "Pending Onchain" tab

Batch Registration (svc-admin or svc-identity)
├── Uses: generateFabricUserId(country, dob, gender, accountType)
├── Format: "CC CCC AANNNN TCCCC NNNN" (e.g., "US A3F HBF934 0XYZW 1234")
├── fabricUserId: GENERATED HERE
├── Creates: CREATE_USER outbox command
└── onchainStatus: PENDING

Blockchain Processing (chaincode + projector)
├── Chaincode creates user on blockchain
├── Projector updates: status = ACTIVE, onchainStatus = ACTIVE
└── User appears in "Active" tab with fabricId visible
```

**FabricId Generator Location:** `packages/core-fabric/src/id-generator.ts`

---

## Status Transition Rules

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              USERS PAGE SCOPE                                    │
│                    (Onchain Users - APPROVED_PENDING_ONCHAIN and beyond)        │
└─────────────────────────────────────────────────────────────────────────────────┘

APPROVED_PENDING_ONCHAIN ──[Batch Register]──► ACTIVE
                                                   │
                    ┌──────────────────────────────┼──────────────────────────────┐
                    │                              │                              │
                    ▼                              ▼                              ▼
              [Suspend]                       [Lock]                          [Ban]
                    │                              │                              │
                    ▼                              ▼                              ▼
              SUSPENDED                        LOCKED                         BANNED
           (admin temp)                    (security)                      (terminal)
                    │                              │
         ┌─────────┼─────────┐          ┌─────────┼─────────┐
         │                   │          │                   │
    [Unsuspend]           [Ban]     [Unlock]             [Ban]
         │                   │          │                   │
         ▼                   ▼          ▼                   ▼
      ACTIVE              BANNED     ACTIVE              BANNED


                          DEACTIVATED (user-initiated)
                                │
                     ┌──────────┼──────────┐
                     │                     │
               [Reactivate]             [Ban]
                     │                     │
                     ▼                     ▼
                  ACTIVE                BANNED


                    ╔═══════════════════════════════════════╗
                    ║  BANNED = TERMINAL (No way out)       ║
                    ╚═══════════════════════════════════════╝
```

### Transition Matrix

| From → To | ACTIVE | SUSPENDED | LOCKED | DEACTIVATED | BANNED |
|-----------|--------|-----------|--------|-------------|--------|
| **ACTIVE** | - | Admin | Admin/System | User | Admin |
| **SUSPENDED** | Admin | - | ❌ | ❌ | Admin |
| **LOCKED** | Admin | ❌ | - | ❌ | Admin |
| **DEACTIVATED** | User | ❌ | ❌ | - | Admin |
| **BANNED** | ❌ | ❌ | ❌ | ❌ | - |

---

## Blockchain Synchronization

### Outbox Commands

| Command | Chaincode Function | Payload |
|---------|-------------------|---------|
| `SET_USER_STATUS` | `IdentityContract:SetUserStatus` | `{ userId, newStatus, reason }` |

### Status Sync Flow

```
Admin Action (Frontend)
       │
       ▼
svc-admin Use Case
       │
       ├─► Update PostgreSQL (UserProfile.status)
       │
       └─► Create OutboxCommand { type: SET_USER_STATUS, payload: {...} }
                     │
                     ▼
            Outbox Submitter Worker
                     │
                     ▼
            Fabric Gateway → SetUserStatus(userId, status, reason)
                     │
                     ▼
            Chaincode emits UserStatusChanged event
                     │
                     ▼
            Projector Worker → Confirm sync
```

### Event Processing

| Chaincode Event | Projector Action |
|-----------------|------------------|
| `UserStatusChanged` | Update UserProfile.onchainStatus, verify sync |
| `UserCreated` | Update UserProfile.fabricUserId, set ACTIVE |

---

## Critical Files

### Chaincode (gx-coin-fabric)
```
gx-coin-fabric/chaincode/gxtv3/
├── consts.go (ADD user status constants)
├── identity_contract.go (ADD SetUserStatus function)
├── identity_types.go (UPDATE User struct)
└── tokenomics_contract.go (UPDATE Transfer to check status)
```

### Backend (svc-admin)
```
gx-protocol-backend/
├── db/prisma/schema/identity/enums.prisma (UPDATE UserProfileStatus)
└── apps/svc-admin/src/
    ├── application/
    │   ├── dto/user-management.dto.ts (ADD new DTOs)
    │   └── use-cases/user-management/
    │       ├── suspend-user.use-case.ts (NEW)
    │       ├── unsuspend-user.use-case.ts (NEW)
    │       ├── lock-user.use-case.ts (NEW)
    │       ├── unlock-user.use-case.ts (NEW)
    │       └── ban-user.use-case.ts (NEW)
    ├── interface/
    │   ├── routes/user-management.routes.ts (ADD endpoints)
    │   └── controllers/user-management.controller.ts (ADD handlers)
    └── shared/di/container.ts (WIRE new use cases)
```

### Admin Frontend
```
gx-admin-frontend/src/
├── app/(main)/dashboard/users/
│   ├── page.tsx (MODIFY - new tab structure)
│   └── _components/
│       ├── pending-onchain-panel.tsx (ENHANCE - batch registration)
│       ├── user-action-dialogs.tsx (ADD suspend/lock/ban dialogs)
│       ├── users-table.tsx (MODIFY - action menu per status)
│       └── user-status-badge.tsx (UPDATE colors/icons)
├── hooks/use-users.ts (ADD new mutations)
└── types/user.ts (UPDATE status types)
```

---

## Verification Plan

### E2E Tests (Playwright MCP)

1. **Test: Batch Registration**
   - Navigate to Users → Pending Onchain tab
   - Verify users shown have NO fabricId (status: APPROVED_PENDING_ONCHAIN)
   - Select multiple users via checkboxes
   - Click "Register Selected"
   - Verify backend generates fabricId for each user (format: "CC CCC AANNNN TCCCC NNNN")
   - Verify users move to Active tab after blockchain registration
   - Verify fabricId is now displayed in the Active users table

2. **Test: Suspend/Unsuspend User**
   - Navigate to Users → Active tab
   - Click Suspend on a user
   - Fill reason: "COMPLIANCE_REVIEW"
   - Verify status → SUSPENDED
   - Verify user in Suspended tab
   - Unsuspend and verify → ACTIVE

3. **Test: Lock/Unlock User**
   - Navigate to Users → Active tab
   - Click Lock on a user
   - Fill reason: "PASSWORD_BREACH"
   - Verify status → LOCKED
   - Unlock with verification
   - Verify → ACTIVE

4. **Test: Ban User (Permanent)**
   - Navigate to Users → Suspended tab
   - Click Ban on a user
   - Fill reason and **required** notes
   - Confirm warning dialog
   - Verify status → BANNED
   - **Verify NO actions available**

5. **Test: Blockchain Sync**
   - Suspend a user via admin
   - Query chaincode to verify on-chain status = "Suspended"
   - Verify user cannot send tokens (receive still works)

### Manual Verification

- [ ] All 7 tabs show correct user counts
- [ ] Action menus show appropriate options per status
- [ ] Batch registration completes with progress indicator
- [ ] Blockchain sync occurs for all status changes
- [ ] Status badges show correct colors/icons
- [ ] Banned users cannot perform ANY actions
- [ ] Suspended/Locked users can still receive coins

---

## Work Package Summary

| Track | WP | Description | Priority | Effort |
|-------|-----|-------------|----------|--------|
| **0** | WP-0.1 | Chaincode status constants | P0 | Low |
| **0** | WP-0.2 | SetUserStatus chaincode function | P0 | Medium |
| **0** | WP-0.3 | Update Transfer validation | P0 | Low |
| **A** | WP-A1 | Update Prisma schema | P0 | Low |
| **A** | WP-A2 | Suspend/Unsuspend endpoints | P1 | Medium |
| **A** | WP-A3 | Lock/Unlock endpoints | P1 | Medium |
| **A** | WP-A4 | Ban endpoint | P1 | Medium |
| **A** | WP-A5 | Deactivate/Reactivate (wallet) | P2 | Medium |
| **A** | WP-A6 | Update user listing filters | P2 | Low |
| **B** | WP-B1 | Update tab structure | P1 | Low |
| **B** | WP-B2 | Batch registration panel | P1 | Medium |
| **B** | WP-B3 | Action dialogs | P1 | High |
| **B** | WP-B4 | Status badges | P2 | Low |
| **B** | WP-B5 | API hooks | P1 | Medium |
| **B** | WP-B6 | Update types | P1 | Low |
| **C** | WP-C1 | FabricId in KYC approved tab | P3 | Low |

**Total:** 16 work packages across 4 tracks

---

## Execution Order

```
Phase 1: Foundation (BLOCKING)
├── Track 0 (Chaincode) - FIRST
│   └── WP-0.1 → WP-0.2 → WP-0.3
│
└── Track A.1 (Schema)
    └── WP-A1 (Prisma update)

Phase 2: Backend Implementation
└── Track A (svc-admin)
    └── WP-A2 → WP-A3 → WP-A4 → WP-A6
    └── WP-A5 (wallet - can be parallel)

Phase 3: Frontend Implementation (after Track A)
├── Track B (Admin Frontend)
│   └── WP-B6 → WP-B5 → WP-B1 → WP-B2 → WP-B3 → WP-B4
│
└── Track C (KYC Page) - Parallel
    └── WP-C1

Phase 4: Verification
└── E2E Tests → Manual QA → Documentation
```

---

## Open Questions (Resolved)

1. ✅ **Onchain statuses defined**: ACTIVE, SUSPENDED, LOCKED, DEACTIVATED, BANNED
2. ✅ **Suspended vs Locked**: Suspended = admin temp, Locked = security incident
3. ✅ **Can banned users be unbanned?** → No, BANNED is terminal
4. ✅ **Batch registration selection?** → Individual checkboxes + "Register All" button
5. ✅ **Blockchain sync required?** → Yes, all status changes sync via SetUserStatus chaincode
6. ✅ **Transaction restrictions?** → Suspended/Locked/Deactivated can receive only; Banned cannot transact at all

---

## Multi-Agent Orchestration Architecture

This implementation follows the enterprise multi-agent orchestration pattern defined in CLAUDE.md v2.3.0.

### Agent Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           TIER 1: ORCHESTRATOR                                   │
│                         Main Claude (Opus 4.5)                                   │
│                                                                                  │
│  Responsibilities:                                                               │
│  • Parse plan and assign tracks to supervisors                                  │
│  • Monitor supervisor progress                                                   │
│  • Resolve cross-track dependencies                                             │
│  • Final verification and approval                                              │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│  SUPERVISOR-0       │   │  SUPERVISOR-A       │   │  SUPERVISOR-B       │
│  Chaincode Track    │   │  Backend Track      │   │  Frontend Track     │
│  (Sonnet)           │   │  (Sonnet)           │   │  (Sonnet)           │
│                     │   │                     │   │                     │
│  WP-0.1, WP-0.2,   │   │  WP-A1 → WP-A6     │   │  WP-B1 → WP-B6     │
│  WP-0.3            │   │                     │   │  + WP-C1           │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
          │                           │                           │
          ▼                           ▼                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        TIER 3: IMPLEMENTATION TEAMS                              │
│                                                                                  │
│  Each Supervisor spawns:                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ prompt-writer│→│ impl-agent   │→│ tester       │→│ reviewer     │        │
│  │ (Haiku)      │  │ (Sonnet)     │  │ (Sonnet)     │  │ (Sonnet)     │        │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘        │
│                                                                                  │
│  Support Agents (as needed):                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                          │
│  │ debugger     │  │ security     │  │ work-recorder│                          │
│  │ (Sonnet)     │  │ (Opus)       │  │ (Haiku)      │                          │
│  └──────────────┘  └──────────────┘  └──────────────┘                          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Agent Definitions

| Agent | Model | Role | Tools |
|-------|-------|------|-------|
| **orchestrator** | Opus | Main coordinator, track assignment | Read, Task |
| **supervisor** | Sonnet | Track lead, WP sequencing | Read, Grep, Glob, Task |
| **prompt-writer** | Haiku | Generate context-optimized prompts | Read, Grep, Glob |
| **chaincode-impl** | Sonnet | Go chaincode implementation | Read, Edit, Write, Bash |
| **backend-impl** | Sonnet | TypeScript backend implementation | Read, Edit, Write, Bash |
| **frontend-impl** | Sonnet | React/Next.js frontend implementation | Read, Edit, Write, Bash |
| **tester** | Sonnet | Write and run tests | Read, Edit, Bash |
| **reviewer** | Sonnet | Code review and quality checks | Read, Grep, Glob |
| **debugger** | Sonnet | Error diagnosis and fixing | Read, Grep, Bash |
| **security** | Opus | Security audit (chaincode + API) | Read, Grep, Glob |
| **work-recorder** | Haiku | Continuous documentation | Read, Edit, Write, Glob |

---

## Execution Protocol

### Per-Work-Package Flow

```
SUPERVISOR receives WP assignment
       │
       ▼
1. PROMPT-WRITER (Haiku, fast)
   │
   │  Input: WP summary, file paths, conventions from CLAUDE.md
   │  Output: Optimized prompt for implementation agent
   │
       ▼
2. IMPL-AGENT (chaincode-impl / backend-impl / frontend-impl)
   │
   │  Input: Optimized prompt from prompt-writer
   │  Action: Implement the work package
   │  Output: Code changes
   │
   ├─► IF FAILURE ─► DEBUGGER ─► PROMPT-WRITER (retry) ─► IMPL-AGENT
   │
       ▼
3. TESTER (if applicable)
   │
   │  Action: Write unit/integration tests
   │  Output: Test files, test results
   │
       ▼
4. REVIEWER
   │
   │  Action: Code review, quality check
   │  Output: Approval or feedback for fixes
   │
       ▼
5. WORK-RECORDER
   │
   │  Action: Log WP completion to work record
   │  Output: Updated docs/workrecords/work-record-YYYY-MM-DD.md
   │
       ▼
SUPERVISOR marks WP complete, proceeds to next WP
```

### Error Recovery Protocol

```
IF impl-agent fails:
   │
   ├─► 1. DEBUGGER diagnoses root cause
   │       - Reads error output
   │       - Investigates related files
   │       - Produces diagnosis report
   │
   ├─► 2. PROMPT-WRITER generates refined prompt
   │       - Incorporates diagnosis
   │       - Adds specific fixes
   │       - References correct patterns
   │
   └─► 3. IMPL-AGENT retries with refined prompt
           - Max 3 retries per WP
           - Escalate to orchestrator if still failing
```

---

## Track Assignments with Agent Mapping

### Track 0: Chaincode (SUPERVISOR-0)

| WP | Description | Agent | Files |
|----|-------------|-------|-------|
| WP-0.1 | User status constants | chaincode-impl | `gx-coin-fabric/chaincode/gxtv3/consts.go` |
| WP-0.2 | SetUserStatus function | chaincode-impl | `gx-coin-fabric/chaincode/gxtv3/identity_contract.go` |
| WP-0.3 | Update Transfer validation | chaincode-impl | `gx-coin-fabric/chaincode/gxtv3/tokenomics_contract.go` |

**Post-Track Validation:**
- security agent: Audit access control, state transitions
- tester agent: Add chaincode unit tests

### Track A: Backend (SUPERVISOR-A)

| WP | Description | Agent | Files |
|----|-------------|-------|-------|
| WP-A1 | Update Prisma schema | backend-impl | `db/prisma/schema/identity/enums.prisma` |
| WP-A2 | Suspend/Unsuspend endpoints | backend-impl | `apps/svc-admin/src/.../suspend-user.use-case.ts` |
| WP-A3 | Lock/Unlock endpoints | backend-impl | `apps/svc-admin/src/.../lock-user.use-case.ts` |
| WP-A4 | Ban endpoint | backend-impl | `apps/svc-admin/src/.../ban-user.use-case.ts` |
| WP-A5 | Deactivate/Reactivate (wallet) | backend-impl | `apps/svc-wallet/src/.../deactivate-account.use-case.ts` |
| WP-A6 | Update user listing filters | backend-impl | `apps/svc-admin/src/.../list-users.use-case.ts` |

**Post-Track Validation:**
- tester agent: Unit tests for use cases
- security agent: Audit endpoint authorization
- reviewer agent: Clean Architecture compliance

### Track B: Frontend (SUPERVISOR-B)

| WP | Description | Agent | Files |
|----|-------------|-------|-------|
| WP-B1 | Update tab structure | frontend-impl | `gx-admin-frontend/src/app/(main)/dashboard/users/page.tsx` |
| WP-B2 | Batch registration panel | frontend-impl | `.../_components/pending-onchain-panel.tsx` |
| WP-B3 | Action dialogs | frontend-impl | `.../_components/user-action-dialogs.tsx` |
| WP-B4 | Status badges | frontend-impl | `.../_components/user-status-badge.tsx` |
| WP-B5 | API hooks | frontend-impl | `gx-admin-frontend/src/hooks/use-users.ts` |
| WP-B6 | Update types | frontend-impl | `gx-admin-frontend/src/types/user.ts` |
| WP-C1 | FabricId in KYC tab | frontend-impl | `.../_components/kyc-applications-table.tsx` |

**Post-Track Validation:**
- tester agent: Component tests (if applicable)
- reviewer agent: UI/UX consistency, accessibility

---

## Parallel Execution Strategy

### Phase 1: Foundation (BLOCKING - Sequential)

```
ORCHESTRATOR
    │
    ├─► SUPERVISOR-0 (Track 0: Chaincode)
    │       └── WP-0.1 → WP-0.2 → WP-0.3 (sequential)
    │
    └─► SUPERVISOR-A (Track A: Schema only)
            └── WP-A1 (can run in parallel with Track 0)
```

### Phase 2: Backend Implementation (After Phase 1)

```
ORCHESTRATOR
    │
    └─► SUPERVISOR-A (Track A: Backend)
            └── WP-A2 → WP-A3 → WP-A4 (sequential, share patterns)
                    │
                    └── WP-A5 (parallel - different service)
                    └── WP-A6 (sequential after A2-A4)
```

### Phase 3: Frontend Implementation (After Phase 2)

```
ORCHESTRATOR
    │
    └─► SUPERVISOR-B (Track B: Frontend)
            │
            ├── WP-B6 → WP-B5 (types before hooks)
            │
            └── After B5:
                ├── WP-B1 → WP-B2 (page structure, then panel)
                ├── WP-B3 (dialogs - can parallel with B1/B2)
                ├── WP-B4 (badges - can parallel)
                └── WP-C1 (KYC table - parallel with B track)
```

### Phase 4: Verification (After All Tracks)

```
ORCHESTRATOR
    │
    ├─► SECURITY agent: Full audit across all tracks
    │
    ├─► TESTER agent: E2E tests via Playwright MCP
    │       └── Test: Batch registration
    │       └── Test: Suspend/Unsuspend flow
    │       └── Test: Lock/Unlock flow
    │       └── Test: Ban (permanent)
    │       └── Test: Blockchain sync
    │
    ├─► REVIEWER agent: Final code review
    │
    └─► WORK-RECORDER agent: Final session documentation
```

---

## Prompt-Writer Examples

### Example: WP-0.2 SetUserStatus

**Supervisor → Prompt-Writer Input:**
```
Generate a prompt for chaincode-impl agent to implement WP-0.2:
- File: gx-coin-fabric/chaincode/gxtv3/identity_contract.go
- Task: Add SetUserStatus function
- Context: New user status system (Active, Suspended, Locked, Deactivated, Banned)
- Reference: Status constants from WP-0.1 in consts.go
- Access: gx_super_admin only
```

**Prompt-Writer Output (optimized for chaincode-impl):**
```
## Task: Implement SetUserStatus chaincode function

### File
`gx-coin-fabric/chaincode/gxtv3/identity_contract.go`

### Context
We're adding a new user status management system. Status constants are defined in `consts.go`:
- UserStatusActive, UserStatusSuspended, UserStatusLocked, UserStatusDeactivated, UserStatusBanned

### Requirements
1. Add function signature:
   func (s *IdentityContract) SetUserStatus(
       ctx contractapi.TransactionContextInterface,
       userID string,
       newStatus string,
       reason string,
   ) error

2. Implementation steps:
   - Check caller has gx_super_admin role using checkAttribute()
   - Validate newStatus is one of the allowed constants
   - Load user from state using composite key
   - Validate transition is allowed (see transition matrix in plan)
   - Update user.Status
   - Put user back to state
   - Emit UserStatusChanged event

3. Follow existing patterns in identity_contract.go (see FreezeWallet for reference)

### Acceptance Criteria
- [ ] Function compiles without errors
- [ ] Access control enforced
- [ ] Invalid transitions rejected with clear error
- [ ] Event emitted on success
```

---

## Work Recording Integration

### Continuous Documentation

The work-recorder agent updates `docs/workrecords/work-record-YYYY-MM-DD.md` after each major milestone:

```markdown
## Session XX: Admin Users Page Implementation

### Phase 1 Completed: Foundation
- WP-0.1: ✅ User status constants added to consts.go
- WP-0.2: ✅ SetUserStatus function implemented
- WP-0.3: ✅ Transfer validation updated
- WP-A1: ✅ Prisma schema updated

### Phase 2 In Progress: Backend
- WP-A2: 🔄 Suspend/Unsuspend endpoints (in progress)
...
```

### Session Summary Template

At session end, work-recorder generates:
```markdown
## Session Summary

**Work Packages Completed:** 10/16
**Tracks:**
- Track 0 (Chaincode): ✅ Complete
- Track A (Backend): ✅ Complete
- Track B (Frontend): 🔄 4/7 complete

**Blockers:** None
**Next Session:** Complete WP-B3, WP-B4, WP-C1, then verification
```

---

## Spawning Commands Reference

### Spawn Supervisor
```
Task(
  prompt="You are SUPERVISOR-0 for Track 0 (Chaincode). Execute WP-0.1 through WP-0.3 sequentially.
         For each WP:
         1. Spawn prompt-writer to generate optimized prompt
         2. Spawn chaincode-impl with the prompt
         3. On failure: spawn debugger, then retry
         4. Log completion to work-recorder
         Report back when track complete.",
  subagent_type="supervisor",
  model="sonnet"
)
```

### Spawn Implementation Agent (via Supervisor)
```
Task(
  prompt="<optimized prompt from prompt-writer>",
  subagent_type="backend-impl",
  model="sonnet"
)
```

### Spawn Parallel Verification
```
# In single message, spawn multiple validators
Task(prompt="Security audit for chaincode...", subagent_type="security", model="opus")
Task(prompt="Run E2E tests...", subagent_type="tester", model="sonnet")
Task(prompt="Code review...", subagent_type="reviewer", model="sonnet")
```

---

## Success Criteria

### Track Completion Gates

| Track | Gate | Verification |
|-------|------|--------------|
| Track 0 | Chaincode compiles | `go build ./...` |
| Track 0 | Unit tests pass | `go test ./...` |
| Track A | Schema applies | `prisma db push` |
| Track A | Services start | `npm run dev` |
| Track B | Types compile | `npm run type-check` |
| Track B | UI renders | Visual verification |
| All | E2E tests pass | Playwright MCP tests |

### Final Verification Checklist

- [ ] All 16 WPs completed
- [ ] Chaincode compiles and deploys
- [ ] Backend services start without errors
- [ ] Frontend types compile
- [ ] E2E tests pass (5 test scenarios)
- [ ] Security audit passed
- [ ] Code review approved
- [ ] Work record updated
- [ ] Commits made per /commit skill

---

## Risk Mitigation

### Dependency Risks

| Risk | Mitigation |
|------|------------|
| Chaincode changes break existing | Run all chaincode tests before Phase 2 |
| Prisma migration fails | Test schema change in dev namespace first |
| Frontend type mismatches | Update types (WP-B6) before hooks (WP-B5) |
| Blockchain sync latency | Add retry logic in outbox submitter |

### Agent Failure Handling

| Scenario | Action |
|----------|--------|
| impl-agent fails | debugger → prompt-writer → retry (max 3) |
| Test failures | debugger to fix, re-run tests |
| Security issues found | Immediate fix before proceeding |
| Supervisor timeout | Orchestrator re-spawns with state checkpoint |
