# Plan: Simplify User Status Routing Flow

## Objective

Clean up the routing logic based on `UserStatus` (user account status):
- **Dashboard users** (ACTIVE, FROZEN, SUSPENDED, CLOSED) → `/dashboard`
- **Onboarding users** (REGISTERED, PENDING_ADMIN_APPROVAL, APPROVED_PENDING_ONCHAIN, DENIED) → `/onboarding`

**Important Distinction:**
- **UserStatus** = User account status (REGISTERED, ACTIVE, etc.) → Used for main routing
- **kycStatus** = KYC progress (NONE, KYC_PENDING, KYC_APPROVED, etc.) → Used within onboarding
- **KYCApplicationStatus** = Application record status (DRAFT, RESUBMISSION_REQUIRED, etc.) → Application-level

Current state: Complex, scattered status checks across 15+ files with verbose conditionals.
Goal: Simple, centralized, easy-to-understand routing.

---

## Multi-Agent Orchestration Structure

```
                    ┌─────────────────────────┐
                    │      ORCHESTRATOR       │
                    │     (Main Claude)       │
                    └───────────┬─────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            │                   │                   │
            ▼                   ▼                   ▼
    ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
    │  TRACK A      │   │  TRACK B      │   │  TRACK C      │
    │  Utility      │   │  Layouts      │   │  Verify       │
    │  (Sonnet)     │   │  (Sonnet)     │   │  (Sonnet)     │
    └───────────────┘   └───────────────┘   └───────────────┘
         │                    │                   │
         ▼                    ▼                   ▼
    Create utility      Update 3 files      Build + Test
    user-status.ts      - landing page      - npm run build
                        - client layout     - Manual verification
                        - onboarding layout
```

### Agent Assignments

| Track | Agent Type | Files | Dependencies |
|-------|------------|-------|--------------|
| **A** | frontend-impl | `lib/utils/user-status.ts` (CREATE) | None |
| **B** | frontend-impl | 3 layout files (MODIFY) | Track A complete |
| **C** | tester/reviewer | Build verification | Tracks A+B complete |

**Execution Strategy:**
- Track A runs first (creates the utility)
- Track B runs after Track A (uses the utility)
- Track C runs after Track B (verifies everything)

---

## Current Problems

1. **Scattered checks**: Status checked in multiple layouts independently
2. **Verbose conditionals**: Long `if (status === 'X' || status === 'Y' || ...)` chains
3. **Indirect routing**: `/` → `/dashboard` → `/onboarding` (unnecessary hops)
4. **No centralized logic**: Same routing decision made in multiple places

---

## Implementation Plan

### Step 1: Create Simple Status Utility

**File:** `gx-wallet-frontend/lib/utils/user-status.ts` (NEW)

```typescript
// Simple status grouping - no over-engineering
// NOTE: These are USER STATUSES (account status), not KYC application statuses

// Statuses that need onboarding/KYC completion (redirect to /onboarding)
const ONBOARDING_STATUSES = [
  'REGISTERED',
  'PENDING_ADMIN_APPROVAL',
  'APPROVED_PENDING_ONCHAIN',
  'DENIED'
] as const

// Statuses that have dashboard access (redirect to /dashboard)
const DASHBOARD_STATUSES = ['ACTIVE', 'FROZEN', 'SUSPENDED', 'CLOSED'] as const

export function needsOnboarding(status: string | undefined): boolean {
  return ONBOARDING_STATUSES.includes(status as any)
}

export function hasDashboardAccess(status: string | undefined): boolean {
  return DASHBOARD_STATUSES.includes(status as any)
}

export function getRedirectPath(status: string | undefined): '/dashboard' | '/onboarding' | '/login' {
  if (!status) return '/login'
  if (needsOnboarding(status)) return '/onboarding'
  return '/dashboard'
}
```

### Step 2: Simplify Root Page

**File:** `gx-wallet-frontend/app/(landing)/page.tsx`

Current: Always redirects to `/dashboard`
```typescript
export default function LandingPage() {
  redirect('/dashboard');
  return null;
}
```

New: Server-side redirect based on status
```typescript
import { getServerSession } from 'next-auth'
import { redirect } from 'next/navigation'
import { authOptions } from '@/app/api/auth/[...nextauth]/auth-options'
import { getRedirectPath } from '@/lib/utils/user-status'

export default async function LandingPage() {
  const session = await getServerSession(authOptions)

  if (!session) {
    redirect('/login')
    return null
  }

  redirect(getRedirectPath(session.user?.status))
  return null
}
```

### Step 3: Simplify Client Layout

**File:** `gx-wallet-frontend/app/(root)/(client)/layout.tsx`

Current: ~100 lines of complex auth checks with multiple status comparisons
New: Simple check using utility

Replace the verbose `needsOnboarding` logic (lines ~40-47):
```typescript
// OLD (verbose)
const needsOnboarding = !isOnVerificationPage && session?.user?.status && (
    session.user.status === 'REGISTERED' ||
    session.user.status === 'PENDING_ADMIN_APPROVAL' ||
    session.user.status === 'APPROVED_PENDING_ONCHAIN' ||
    session.user.status === 'DENIED' ||
    session.user.status === 'FROZEN' ||
    session.user.status === 'SUSPENDED'
)
```

```typescript
// NEW (simple)
import { needsOnboarding } from '@/lib/utils/user-status'

// In component:
if (!isOnVerificationPage && needsOnboarding(session?.user?.status)) {
  redirect('/onboarding')
}
```

### Step 4: Simplify Onboarding Layout

**File:** `gx-wallet-frontend/app/onboarding/layout.tsx`

Current: Checks only ACTIVE status
```typescript
if (status === 'authenticated' && session?.user?.status === 'ACTIVE') {
  router.push('/dashboard');
  return;
}
```

New: Use utility for consistency
```typescript
import { hasDashboardAccess } from '@/lib/utils/user-status'

// In component:
if (status === 'authenticated' && hasDashboardAccess(session?.user?.status)) {
  router.push('/dashboard')
  return
}
```

---

## Files to Modify

| File | Action | Changes |
|------|--------|---------|
| `lib/utils/user-status.ts` | CREATE | Status utility with `needsOnboarding()`, `hasDashboardAccess()`, `getRedirectPath()` |
| `app/(landing)/page.tsx` | MODIFY | Server-side redirect based on status |
| `app/(root)/(client)/layout.tsx` | MODIFY | Replace verbose conditionals with utility call |
| `app/onboarding/layout.tsx` | MODIFY | Replace ACTIVE check with utility call |

---

## Verification

1. **Test routing from `/`:**
   - Logged out → `/login`
   - REGISTERED → `/onboarding`
   - PENDING_ADMIN_APPROVAL → `/onboarding`
   - APPROVED_PENDING_ONCHAIN → `/onboarding`
   - DENIED → `/onboarding`
   - ACTIVE → `/dashboard`
   - FROZEN → `/dashboard`
   - SUSPENDED → `/dashboard`
   - CLOSED → `/dashboard`

2. **Test direct access:**
   - REGISTERED user accessing `/dashboard` → redirected to `/onboarding`
   - ACTIVE user accessing `/onboarding` → redirected to `/dashboard`

3. **Build verification:**
   ```bash
   cd gx-wallet-frontend
   npm run build
   ```

---

## Summary

**Before:** Complex, scattered, 15+ files with duplicated logic
**After:** 1 utility file + 3 simple modifications

| Metric | Value |
|--------|-------|
| Total files | 4 (1 create, 3 modify) |
| New code | ~25 lines |
| Agents | 3 (Track A, B, C) |
| Execution | Sequential (A → B → C) |
| Complexity | Low |

### Agent Execution Order

1. **Track A Agent** (frontend-impl): Create `lib/utils/user-status.ts`
2. **Track B Agent** (frontend-impl): Update 3 layout files to use utility
3. **Track C Agent** (tester): `npm run build` + routing verification

### Work Recording

Orchestrator will update work record (`docs/workrecords/work-record-2026-01-27.md`) with:
- Session 7: User Status Routing Simplification
- Track progress and agent results
- Final verification status
