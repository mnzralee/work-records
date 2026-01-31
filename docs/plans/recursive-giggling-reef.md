# Plan: Wallet Frontend Dashboard Fix + Enterprise State Management

## Multi-Agent Orchestration Plan

```
TIER 1: ORCHESTRATOR (Main Claude - Opus)
├── Parse this plan document
├── Spawn supervisors for each track
│
├── TIER 2: SUPERVISORS (Track Leads - Sonnet)
│   ├── supervisor-api-routes (Track A: Fix 404 Errors)
│   ├── supervisor-state-mgmt (Track B: TanStack Query Setup)
│   └── supervisor-verify (Track C: E2E Testing)
│
└── TIER 3: SUPPORT TEAMS (Per Supervisor)
    ├── frontend-impl (Sonnet) → Execute code changes
    ├── tester (Sonnet) → Verify functionality
    ├── reviewer (Sonnet) → Code quality check
    └── work-recorder (Haiku) → Document progress
```

---

## Track Assignment Summary

| Track | Supervisor | Agents | Work Packages |
|-------|------------|--------|---------------|
| **A** | supervisor-api-routes | frontend-impl | WP-A1: Fix dashboard route, WP-A2: Fix transactions route, WP-A3: Fix notifications route, WP-A4: Fix users/me route |
| **B** | supervisor-state-mgmt | frontend-impl | WP-B1: Install TanStack Query, WP-B2: Create QueryClient, WP-B3: Create hooks, WP-B4: Refactor dashboard |
| **C** | supervisor-verify | tester, reviewer | WP-C1: Port-forward setup, WP-C2: E2E verification, WP-C3: Code review |

---

## Problem Summary

### Issue 1: 404 Errors
The wallet frontend dashboard is broken with multiple 404 errors:
- `GET /api/wallet/dashboard` - returns 404
- `GET /api/wallet/transactions` - returns 404
- `GET /api/users/me` - returns 404
- `GET /api/notifications/unread-count` - returns 404

**Root Cause**: Frontend API routes use `identityClient` (port 3041 = svc-auth) to call wallet endpoints, but:
1. svc-auth only handles authentication - no wallet endpoints
2. The decomposed `svc-wallet` (port 3043) has the correct endpoints
3. The endpoint paths are different: `/wallet/balance` vs `/wallets/${userId}/balance`

## Current State

### Backend Services (backend-dev-manazir namespace)

| Service | Status | Port | Purpose |
|---------|--------|------|---------|
| svc-auth | Running | 3041 | Authentication only |
| svc-wallet | Running | 3043 | Dashboard, balance, transactions, notifications |
| svc-identity | Running | N/A | User profiles (monolith) |
| svc-kyc | Running | 3042 | KYC operations |
| svc-trust | Running | 3044 | Relationships |

### Frontend Configuration

`.env.local` defines:
- `AUTH_API_URL=http://localhost:3041` (svc-auth)
- `WALLET_API_URL=http://localhost:3043` (svc-wallet)

`backendClient.ts` exports:
- `authClient` (port 3041)
- `walletClient` (port 3043)
- `identityClient` = alias to authClient (deprecated)

## Reference Files (Read-Only)

| File | Purpose |
|------|---------|
| `gx-wallet-frontend/lib/backendClient.ts` | Backend client definitions |
| `gx-protocol-backend/apps/svc-wallet/src/interface/routes/index.ts` | svc-wallet route definitions |
| `gx-protocol-backend/apps/svc-wallet/src/interface/controllers/index.ts` | Response format reference |

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Response format mismatch | Transform responses in frontend API routes |
| JWT token incompatibility | Verify JWT secret matches across services |
| svc-wallet missing data | Fallback to default empty state (already implemented) |
| TanStack Query memory leak | Ensure proper cleanup in hooks |

---

## Part 2: Enterprise State Management (Eliminate Skeleton Flash)

### Current Problem

The dashboard currently uses manual `useState`/`useEffect` pattern that causes:
- Skeleton flash on every page mount
- Full reload on every navigation
- No caching of fetched data
- Unnecessary API calls for unchanged data

```typescript
// Current anti-pattern in dashboard/page.tsx
const fetchDashboardData = useCallback(async () => {
  setIsLoading(true)  // <-- Always shows skeleton!
  // ... fetch data
  setIsLoading(false)
}, [session])
```

### Solution: TanStack Query (React Query v5)

[TanStack Query](https://tanstack.com/query/latest) is the enterprise standard for server state management in 2026, powering 80% of new React apps per the [2025 State of JS survey](https://refine.dev/blog/react-query-vs-tanstack-query-vs-swr-2025/).

**Key Features**:
- **Stale-While-Revalidate**: Show cached data immediately, refresh in background
- **Smart Caching**: Deduplicated queries, configurable cache time
- **Background Refetching**: Only when data is stale or window regains focus
- **Optimistic Updates**: Instant UI feedback
- **DevTools**: Built-in debugging

### Implementation Strategy

#### Phase 5: Install TanStack Query (2 min)

```bash
cd gx-wallet-frontend
npm install @tanstack/react-query @tanstack/react-query-devtools
```

#### Phase 6: Setup Query Provider (5 min)

Create `/lib/queryClient.ts`:
```typescript
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30 * 1000,        // Data fresh for 30 seconds
      gcTime: 5 * 60 * 1000,       // Cache for 5 minutes (was cacheTime)
      refetchOnWindowFocus: true,  // Refetch when tab regains focus
      refetchOnMount: false,       // Don't refetch if data is fresh
      retry: 1,                    // Retry failed requests once
    },
  },
})
```

Update `/app/layout.tsx`:
```typescript
import { QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { queryClient } from '@/lib/queryClient'

export default function RootLayout({ children }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && <ReactQueryDevtools />}
    </QueryClientProvider>
  )
}
```

#### Phase 7: Create Query Hooks (10 min)

Create `/lib/hooks/useDashboardQueries.ts`:
```typescript
import { useQuery } from '@tanstack/react-query'
import apiClient from '@/lib/apiClient'

// Dashboard data query - cached, background refresh
export function useDashboardData() {
  return useQuery({
    queryKey: ['dashboard'],
    queryFn: async () => {
      const [userData, walletRes, txRes] = await Promise.all([
        apiClient('/users/me'),
        apiClient('/wallet/dashboard'),
        apiClient('/wallet/transactions')
      ])
      return { user: userData.user, wallet: walletRes, transactions: txRes.transactions }
    },
    staleTime: 30 * 1000,  // Fresh for 30 seconds
  })
}

// Wallet balance only (for quick updates)
export function useWalletBalance() {
  return useQuery({
    queryKey: ['wallet', 'balance'],
    queryFn: () => apiClient('/wallet/dashboard'),
    staleTime: 10 * 1000,  // Fresh for 10 seconds
  })
}

// Transactions (paginated)
export function useTransactions(page = 1) {
  return useQuery({
    queryKey: ['transactions', page],
    queryFn: () => apiClient(`/wallet/transactions?page=${page}`),
    staleTime: 60 * 1000,  // Fresh for 1 minute
  })
}
```

#### Phase 8: Refactor Dashboard (15 min)

Update `/app/(root)/(client)/dashboard/page.tsx`:
```typescript
'use client'
import { useDashboardData } from '@/lib/hooks/useDashboardQueries'

const Dashboard = () => {
  const { data, isLoading, isFetching } = useDashboardData()

  // First load only shows skeleton
  if (isLoading) {
    return <DashboardSkeleton />
  }

  // Subsequent loads show cached data + subtle refresh indicator
  return (
    <div className="min-h-screen ...">
      {isFetching && <RefreshIndicator />}  {/* Small spinner in corner */}
      <DashboardHeader user={data.user} />
      <BalanceCard walletData={data.wallet} />
      {/* ... */}
    </div>
  )
}
```

**Key Difference**:
- `isLoading` = true only on FIRST load (no cached data)
- `isFetching` = true when refreshing in background (has cached data)

### Optional: Real-Time Updates with WebSocket

The project already has `socket.io-client` installed. For real-time balance updates:

```typescript
// lib/hooks/useRealtimeBalance.ts
import { useEffect } from 'react'
import { useQueryClient } from '@tanstack/react-query'
import { io } from 'socket.io-client'

export function useRealtimeBalance() {
  const queryClient = useQueryClient()

  useEffect(() => {
    const socket = io(process.env.NEXT_PUBLIC_WS_URL)

    socket.on('balance:updated', (data) => {
      // Invalidate cache - triggers background refetch
      queryClient.invalidateQueries({ queryKey: ['wallet', 'balance'] })
    })

    socket.on('transaction:new', (data) => {
      queryClient.invalidateQueries({ queryKey: ['transactions'] })
    })

    return () => socket.disconnect()
  }, [queryClient])
}
```

### State Management Architecture (2026 Best Practices)

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND STATE LAYERS                     │
├─────────────────────────────────────────────────────────────┤
│  Server State (TanStack Query)                              │
│  - Dashboard data, transactions, user profile               │
│  - Cached, background refresh, stale-while-revalidate       │
├─────────────────────────────────────────────────────────────┤
│  URL State (Next.js router)                                 │
│  - Filters, pagination, search params                       │
│  - Shareable, bookmarkable                                  │
├─────────────────────────────────────────────────────────────┤
│  Client State (useState/Zustand)                            │
│  - UI state: modals, toasts, sidebar open                   │
│  - Form state: React Hook Form                              │
├─────────────────────────────────────────────────────────────┤
│  Real-Time (WebSocket/SSE)                                  │
│  - Push notifications                                       │
│  - Balance updates                                          │
│  - Invalidates TanStack Query cache                         │
└─────────────────────────────────────────────────────────────┘
```

Sources:
- [TanStack Query Official](https://tanstack.com/query/latest)
- [React Query vs SWR 2025 Comparison](https://refine.dev/blog/react-query-vs-tanstack-query-vs-swr-2025/)
- [React State Management 2025](https://www.developerway.com/posts/react-state-management-2025)

---

---

## Track A: Fix API Routes (supervisor-api-routes)

**Agent**: `frontend-impl`
**Execution**: Sequential (each WP depends on previous)

### WP-A1: Fix Dashboard Route

**File**: `gx-wallet-frontend/app/api/wallet/dashboard/route.ts`

**Current (broken)**:
```typescript
import { identityClient } from '@/lib/backendClient'
const response = await identityClient.get(`/wallets/${userId}/balance`, {...})
```

**Target**:
```typescript
import { walletClient } from '@/lib/backendClient'
const response = await walletClient.get('/wallet/dashboard', {...})
```

**Changes**:
1. Replace `identityClient` → `walletClient`
2. Replace `/wallets/${userId}/balance` → `/wallet/dashboard`
3. Update response transformation to match svc-wallet format

### WP-A2: Fix Transactions Route

**File**: `gx-wallet-frontend/app/api/wallet/transactions/route.ts`

**Changes**:
1. Replace `identityClient` → `walletClient`
2. Replace `/wallets/${userId}/transactions` → `/wallet/transactions`

### WP-A3: Fix Notifications Route

**File**: `gx-wallet-frontend/app/api/notifications/unread-count/route.ts`

**Changes**:
1. Replace `identityClient` → `walletClient`
2. Replace `/wallet/notifications/count` → `/notifications/count`

### WP-A4: Fix Users/Me Route

**File**: `gx-wallet-frontend/app/api/users/me/route.ts`

**Changes**:
1. Remove backend call entirely
2. Return user data from NextAuth session
3. Session already contains: id, email, name, firstName, lastName, status

---

## Track B: State Management (supervisor-state-mgmt)

**Agent**: `frontend-impl`
**Execution**: Sequential (WP-B1 must complete first)

### WP-B1: Install TanStack Query

**Command**:
```bash
cd gx-wallet-frontend && npm install @tanstack/react-query @tanstack/react-query-devtools
```

### WP-B2: Create Query Client

**File**: `gx-wallet-frontend/lib/queryClient.ts` (CREATE)

```typescript
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30 * 1000,        // Fresh for 30 seconds
      gcTime: 5 * 60 * 1000,       // Cache for 5 minutes
      refetchOnWindowFocus: true,
      refetchOnMount: false,       // Don't refetch if data is fresh
      retry: 1,
    },
  },
})
```

**File**: `gx-wallet-frontend/app/providers.tsx` (CREATE)

```typescript
'use client'
import { QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { queryClient } from '@/lib/queryClient'

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && <ReactQueryDevtools />}
    </QueryClientProvider>
  )
}
```

**File**: `gx-wallet-frontend/app/layout.tsx` (MODIFY)

Add `<Providers>` wrapper around children.

### WP-B3: Create Dashboard Query Hooks

**File**: `gx-wallet-frontend/lib/hooks/useDashboardQueries.ts` (CREATE)

```typescript
import { useQuery } from '@tanstack/react-query'
import apiClient from '@/lib/apiClient'

export function useDashboardData() {
  return useQuery({
    queryKey: ['dashboard'],
    queryFn: async () => {
      const [userData, walletRes, txRes] = await Promise.all([
        apiClient('/users/me'),
        apiClient('/wallet/dashboard'),
        apiClient('/wallet/transactions')
      ])
      return {
        user: userData.user,
        wallet: walletRes,
        transactions: txRes.transactions || []
      }
    },
    staleTime: 30 * 1000,
  })
}

export function useWalletBalance() {
  return useQuery({
    queryKey: ['wallet', 'balance'],
    queryFn: () => apiClient('/wallet/dashboard'),
    staleTime: 10 * 1000,
  })
}

export function useTransactions(page = 1) {
  return useQuery({
    queryKey: ['transactions', page],
    queryFn: () => apiClient(`/wallet/transactions?page=${page}`),
    staleTime: 60 * 1000,
  })
}

export function useNotificationCount() {
  return useQuery({
    queryKey: ['notifications', 'count'],
    queryFn: () => apiClient('/notifications/unread-count'),
    staleTime: 30 * 1000,
  })
}
```

### WP-B4: Refactor Dashboard Page

**File**: `gx-wallet-frontend/app/(root)/(client)/dashboard/page.tsx` (MODIFY)

**Key Changes**:
1. Replace `useState`/`useEffect` data fetching with `useDashboardData()`
2. Use `isLoading` only for first load (no cached data)
3. Use `isFetching` for subtle background refresh indicator
4. Remove manual `fetchDashboardData` callback

**Before**:
```typescript
const [isLoading, setIsLoading] = useState(true)
const fetchDashboardData = useCallback(async () => {
  setIsLoading(true)  // Always shows skeleton
  // fetch...
  setIsLoading(false)
}, [session])
```

**After**:
```typescript
const { data, isLoading, isFetching } = useDashboardData()

if (isLoading) return <DashboardSkeleton />  // Only on first load

return (
  <div>
    {isFetching && <RefreshIndicator />}  // Subtle spinner
    <BalanceCard walletData={data.wallet} />
    ...
  </div>
)
```

---

## Track C: Verification (supervisor-verify)

**Agents**: `tester`, `reviewer`
**Execution**: Parallel after Track A + B complete

### WP-C1: Port-Forward Setup

**Agent**: Infrastructure check (manual or debugger)

```bash
# Kill existing port-forwards
pkill -f "port-forward.*304"

# Start fresh port-forwards to backend-dev-manazir
kubectl port-forward -n backend-dev-manazir svc/svc-auth 3041:80 &
kubectl port-forward -n backend-dev-manazir svc/svc-wallet 3043:80 &

# Verify services are healthy
curl http://localhost:3041/health
curl http://localhost:3043/health
```

### WP-C2: E2E Verification

**Agent**: `tester`

**Test Plan**:
1. Start frontend: `cd gx-wallet-frontend && npm run dev`
2. Open browser: http://localhost:3000 (or 3002)
3. Login with: `manazir.dev@gmail.com`
4. Verify dashboard loads WITHOUT skeleton flash on subsequent visits
5. Check Network tab:
   - `/api/wallet/dashboard` → 200
   - `/api/wallet/transactions` → 200
   - `/api/notifications/unread-count` → 200
   - `/api/users/me` → 200
6. Navigate away and back - verify cached data shows immediately
7. Verify background refresh indicator appears briefly

**Playwright Verification**:
```typescript
// Use browser_snapshot to verify dashboard renders
await mcp__playwright__browser_navigate({ url: 'http://localhost:3000/dashboard' })
await mcp__playwright__browser_snapshot()
```

### WP-C3: Code Review

**Agent**: `reviewer`

**Review Checklist**:
- [ ] No TypeScript errors
- [ ] Correct client used (walletClient vs identityClient)
- [ ] Proper error handling in API routes
- [ ] TanStack Query hooks follow best practices
- [ ] No memory leaks (query cleanup)
- [ ] Responsive design preserved

---

## Execution Protocol

```
Orchestrator (Main Claude)
│
├── Phase 1: Infrastructure Setup
│   └── WP-C1: Verify port-forwards working
│
├── Phase 2: Track A (API Routes)
│   ├── supervisor-api-routes spawns frontend-impl
│   ├── Execute: WP-A1 → WP-A2 → WP-A3 → WP-A4 (sequential)
│   └── On error: debugger → retry
│
├── Phase 3: Track B (State Management)
│   ├── supervisor-state-mgmt spawns frontend-impl
│   ├── Execute: WP-B1 → WP-B2 → WP-B3 → WP-B4 (sequential)
│   └── On error: debugger → retry
│
├── Phase 4: Track C (Verification) - PARALLEL
│   ├── supervisor-verify spawns tester + reviewer
│   ├── WP-C2: E2E testing (tester)
│   └── WP-C3: Code review (reviewer)
│
└── Phase 5: Documentation
    └── work-recorder updates docs/workrecords/work-record-2026-01-26.md
```

---

## Files Summary

| File | Action | Track | Agent |
|------|--------|-------|-------|
| `app/api/wallet/dashboard/route.ts` | Modify | A | frontend-impl |
| `app/api/wallet/transactions/route.ts` | Modify | A | frontend-impl |
| `app/api/notifications/unread-count/route.ts` | Modify | A | frontend-impl |
| `app/api/users/me/route.ts` | Modify | A | frontend-impl |
| `lib/queryClient.ts` | Create | B | frontend-impl |
| `app/providers.tsx` | Create | B | frontend-impl |
| `app/layout.tsx` | Modify | B | frontend-impl |
| `lib/hooks/useDashboardQueries.ts` | Create | B | frontend-impl |
| `app/(root)/(client)/dashboard/page.tsx` | Modify | B | frontend-impl |
| `docs/workrecords/work-record-2026-01-26.md` | Update | - | work-recorder |
