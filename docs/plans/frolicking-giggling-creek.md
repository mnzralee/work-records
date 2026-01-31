# Local Testing: User Onchain Registration Module

> **Date**: 2026-01-29 (Session 10)
> **Developer**: dev-manazir
> **Objective**: Run all services locally and test the complete user registration flow with fresh users
> **Prerequisites**: Session 9 code changes already committed (security, resilience, fabricUserId fixes)

---

## Today's Plan: Local Testing & Verification

### Goal
Run the updated backend services locally and verify the complete user registration flow works end-to-end with fresh test users.

### Services to Run Locally

| Service | Port | Command |
|---------|------|---------|
| svc-admin | 3040 | `npm run dev --filter=svc-admin` |
| svc-auth | 3041 | `kubectl port-forward -n backend-dev-manazir svc/svc-auth 3041:80` |
| svc-kyc | 3042 | `kubectl port-forward -n backend-dev-manazir svc/svc-kyc 3042:80` |
| svc-wallet | 3043 | `kubectl port-forward -n backend-dev-manazir svc/svc-wallet 3043:80` |
| outbox-submitter | - | Run locally with `npx ts-node` or port-forward |
| Admin Frontend | 3001 | `cd gx-admin-frontend && npm run dev` |

### Testing Steps

#### Step 1: Start Infrastructure
```bash
# Terminal 1: Port-forward dependent services
kubectl port-forward -n backend-dev-manazir svc/svc-auth 3041:80 &
kubectl port-forward -n backend-dev-manazir svc/svc-kyc 3042:80 &
kubectl port-forward -n backend-dev-manazir svc/svc-wallet 3043:80 &

# Terminal 2: Run svc-admin locally (has our new code)
cd gx-protocol-backend && npm run dev --filter=svc-admin

# Terminal 3: Run admin frontend
cd gx-admin-frontend && npm run dev
```

#### Step 2: Create Fresh Test Users
Option A: Via Admin Portal UI
1. Navigate to http://localhost:3001/dashboard/kyc
2. Find a PENDING application
3. Review and approve → Should generate fabricUserId

Option B: Via Database Seed
```sql
-- Check for existing pending users
SELECT "profileId", email, status, "fabricUserId"
FROM "UserProfile"
WHERE status = 'APPROVED_PENDING_ONCHAIN';
```

#### Step 3: Test Batch Registration
1. Navigate to http://localhost:3001/dashboard/users
2. Click "Pending Onchain" tab
3. Verify users appear with their fabricUserId
4. Select user(s) and click "Register Selected"
5. Monitor outbox-submitter logs for processing

#### Step 4: Verify Results
```sql
-- Check user moved to ACTIVE
SELECT "profileId", email, status, "onchainStatus", "fabricUserId"
FROM "UserProfile"
WHERE "fabricUserId" IS NOT NULL;

-- Check outbox commands
SELECT id, "commandType", status, attempts, "fabricTxId"
FROM "OutboxCommand"
WHERE "commandType" = 'REGISTER_USER_ONCHAIN'
ORDER BY "createdAt" DESC;
```

### Verification Checklist
- [ ] svc-admin starts without errors locally
- [ ] fabricUserId generated at KYC approval (not null)
- [ ] Batch registration creates outbox command
- [ ] Outbox-submitter processes command (check logs)
- [ ] User status changes to ACTIVE
- [ ] Blockchain transaction confirmed (fabricTxId populated)

---

## Current State (Discovered)

### K8s Status (backend-dev-manazir)
| Service | Status | Issue |
|---------|--------|-------|
| svc-admin | ❌ CrashLoopBackOff | Prisma client not generated in Docker image |
| outbox-submitter | ✅ Running | Healthy, 18h uptime |
| svc-auth | ✅ Running | OK |
| svc-kyc | ✅ Running | OK |
| svc-wallet | ✅ Running | OK |
| postgres | ✅ Running | OK |

**svc-admin Crash Cause:**
```
Error: @prisma/client did not initialize yet. Please run "prisma generate"
```
→ Docker image needs rebuild. For now, run locally.

### Database Check Needed
```bash
# Check pending users
kubectl exec -n backend-dev-manazir postgres-0 -- psql -U gx_admin -d gxcoin -c \
  "SELECT COUNT(*) FROM \"UserProfile\" WHERE status = 'APPROVED_PENDING_ONCHAIN';"

# Check outbox queue
kubectl exec -n backend-dev-manazir postgres-0 -- psql -U gx_admin -d gxcoin -c \
  "SELECT \"commandType\", status, COUNT(*) FROM \"OutboxCommand\" GROUP BY \"commandType\", status;"
```

---

## Execution Plan

### Phase 1: Rebuild and Deploy svc-admin

The Dockerfile already includes `prisma generate` - the image just needs rebuilding with latest code.

**Step 1: Build Docker Image**
```bash
cd /home/dev-manazir/prod-blockchain/gx-protocol-backend

# First build the packages
npm run build

# Build Docker image
docker build -f apps/svc-admin/Dockerfile -t svc-admin:dev-manazir .
```

**Step 2: Push to Registry**
```bash
# Tag for your registry (check current deployment for registry URL)
kubectl get deployment svc-admin -n backend-dev-manazir -o jsonpath='{.spec.template.spec.containers[0].image}'

# Push to same registry
docker tag svc-admin:dev-manazir <registry>/svc-admin:dev-manazir
docker push <registry>/svc-admin:dev-manazir
```

**Step 3: Update Deployment**
```bash
# Update image and restart
kubectl set image deployment/svc-admin svc-admin=<registry>/svc-admin:dev-manazir -n backend-dev-manazir

# Or restart to pull fresh
kubectl rollout restart deployment/svc-admin -n backend-dev-manazir
```

**Step 4: Verify**
```bash
# Watch rollout
kubectl rollout status deployment/svc-admin -n backend-dev-manazir

# Check logs
kubectl logs -n backend-dev-manazir deploy/svc-admin --tail=20
```

### Phase 2: Verify Services Running
```bash
# Check all pods are healthy
kubectl get pods -n backend-dev-manazir

# Check svc-admin logs
kubectl logs -n backend-dev-manazir deploy/svc-admin --tail=20
```

### Phase 3: Check Database State
```bash
# Find correct database name and check pending users
kubectl exec -n backend-dev-manazir postgres-0 -- psql -U <user> -d <db> -c \
  "SELECT COUNT(*) FROM \"UserProfile\" WHERE status = 'APPROVED_PENDING_ONCHAIN';"
```

### Phase 4: Test Registration Flow
1. Port-forward svc-admin: `kubectl port-forward -n backend-dev-manazir svc/svc-admin 3040:80`
2. Start admin frontend: `cd gx-admin-frontend && npm run dev`
3. Navigate to http://localhost:3001/dashboard/users → Pending Onchain
4. Select user(s) → Click Register Selected
5. Monitor outbox-submitter logs

### Phase 5: Verify Results
1. Check user status changed to ACTIVE
2. Check fabricTxId populated in OutboxCommand
3. Check wallet balance (genesis tokens)

---

## Previous Session 9 Summary (Reference)

All these code changes are ALREADY COMMITTED:

---

## Executive Summary

This plan completes the **User Onchain Registration Module** with enterprise-grade improvements:

| Track | Focus | Priority | Agent Team |
|-------|-------|----------|------------|
| **Track A** | Code Quality Fixes | P0 | backend-impl |
| **Track B** | fabricUserId Pre-Generation | P0 CRITICAL | backend-impl |
| **Track C** | Projector Event Handling | P1 | chaincode-impl, backend-impl |
| **Track D** | Resilience Patterns | P0 CRITICAL | backend-impl |
| **Track E** | Security Hardening | P0 HIGH | backend-impl |
| **Track F** | Observability | P1 | backend-impl |
| **Track G** | Unit & Integration Tests | P1 | tester |
| **Track H** | E2E Playwright Testing | P2 | tester (Playwright MCP) |

---

## Multi-Agent Orchestration Architecture

### Department Hierarchy

```
                              ┌──────────────────────────────────┐
                              │      TIER 1: ORCHESTRATOR        │
                              │      (Main Claude - Opus)        │
                              │   Spawns supervisors, monitors   │
                              │   gates, synthesizes results     │
                              └────────────────┬─────────────────┘
                                               │
        ┌──────────────────────────────────────┼──────────────────────────────────────┐
        │                  │                   │                   │                  │
        ▼                  ▼                   ▼                   ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ SUPERVISOR-A  │  │ SUPERVISOR-B  │  │ SUPERVISOR-C  │  │ SUPERVISOR-D  │  │ SUPERVISOR-E  │
│ Code Quality  │  │ fabricUserId  │  │  Chaincode    │  │  Resilience   │  │   Security    │
│   (Sonnet)    │  │   (Sonnet)    │  │   (Sonnet)    │  │   (Sonnet)    │  │   (Sonnet)    │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                  │                   │                   │                  │
        │                  │                   │                   │                  │
    ┌───┴───┐          ┌───┴───┐           ┌───┴───┐           ┌───┴───┐          ┌───┴───┐
    │ TEAM  │          │ TEAM  │           │ TEAM  │           │ TEAM  │          │ TEAM  │
    │   A   │          │   B   │           │   C   │           │   D   │          │   E   │
    └───────┘          └───────┘           └───────┘           └───────┘          └───────┘


        ┌───────────────────────────────────────────────────────────────────────────┐
        │                                                                           │
        ▼                                     ▼                                     ▼
┌───────────────┐                     ┌───────────────┐                     ┌───────────────┐
│ SUPERVISOR-F  │                     │ SUPERVISOR-G  │                     │ SUPERVISOR-H  │
│ Observability │                     │    Testing    │                     │   E2E Tests   │
│   (Sonnet)    │                     │   (Sonnet)    │                     │   (Sonnet)    │
└───────┬───────┘                     └───────┬───────┘                     └───────┬───────┘
        │                                     │                                     │
    ┌───┴───┐                             ┌───┴───┐                             ┌───┴───┐
    │ TEAM  │                             │ TEAM  │                             │ TEAM  │
    │   F   │                             │   G   │                             │   H   │
    └───────┘                             └───────┘                             └───────┘
```

### Team Composition (Per Supervisor)

Each supervisor manages a team of 4 specialized agents:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SUPERVISOR TEAM STRUCTURE                         │
│                                                                          │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐      │
│  │  PROMPT-WRITER  │───▶│   IMPL-AGENT    │───▶│    DEBUGGER     │      │
│  │    (Haiku)      │    │   (Sonnet)      │    │    (Sonnet)     │      │
│  │  Fast context   │    │  Executes code  │    │  Error recovery │      │
│  │  generation     │    │    changes      │    │   & retry       │      │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘      │
│           │                                              │               │
│           │                                              │               │
│           ▼                                              ▼               │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │                    WORK-RECORDER (Haiku)                     │        │
│  │            Continuous documentation of progress              │        │
│  └─────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Supervisor Assignments

| Supervisor | Track | Model | Impl-Agent Type | Focus |
|------------|-------|-------|-----------------|-------|
| SUPERVISOR-A | Track A: Code Quality | Sonnet | backend-impl | outbox-submitter refactoring |
| SUPERVISOR-B | Track B: fabricUserId | Sonnet | backend-impl | KYC repo + outbox changes |
| SUPERVISOR-C | Track C: Chaincode | Sonnet | chaincode-impl + backend-impl | Go chaincode + projector |
| SUPERVISOR-D | Track D: Resilience | Sonnet | backend-impl | core-fabric + outbox patterns |
| SUPERVISOR-E | Track E: Security | Sonnet | backend-impl | SQL fix + rate limiting |
| SUPERVISOR-F | Track F: Observability | Sonnet | backend-impl | Logging + metrics + probes |
| SUPERVISOR-G | Track G: Testing | Sonnet | tester | Unit + integration tests |
| SUPERVISOR-H | Track H: E2E | Sonnet | tester (Playwright) | Browser automation testing |

### Parallel Execution Strategy

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                           PHASE 1: CRITICAL FIXES (PARALLEL)                           │
│                                                                                        │
│   ┌──────────────────────┐          ┌──────────────────────┐                          │
│   │    SUPERVISOR-D      │          │    SUPERVISOR-E      │                          │
│   │    (Resilience)      │  ║║║║║║  │    (Security)        │                          │
│   │    WP-D1 → WP-D5     │  ║║║║║║  │    WP-E1 → WP-E5     │                          │
│   └──────────────────────┘          └──────────────────────┘                          │
│                                                                                        │
│   ═══════════════════════════════ GATE 1 ═══════════════════════════════              │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                         PHASE 2: CORE LOGIC (PARALLEL)                                 │
│                                                                                        │
│   ┌──────────────────────┐          ┌──────────────────────┐                          │
│   │    SUPERVISOR-A      │          │    SUPERVISOR-B      │                          │
│   │    (Code Quality)    │  ║║║║║║  │    (fabricUserId)    │                          │
│   │    WP-A1 → WP-A5     │  ║║║║║║  │    WP-B1 → WP-B3     │                          │
│   └──────────────────────┘          └──────────────────────┘                          │
│                                                                                        │
│   ═══════════════════════════════ GATE 2 ═══════════════════════════════              │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                         PHASE 3: CHAINCODE (SEQUENTIAL)                                │
│                                                                                        │
│   ┌────────────────────────────────────────────────────────────────────┐              │
│   │                        SUPERVISOR-C                                 │              │
│   │                        (Chaincode)                                  │              │
│   │                        WP-C1 → WP-C3                                │              │
│   │    [chaincode-impl for Go] → [backend-impl for projector]          │              │
│   └────────────────────────────────────────────────────────────────────┘              │
│                                                                                        │
│   ═══════════════════════════════ GATE 3 ═══════════════════════════════              │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                         PHASE 4: OBSERVABILITY (PARALLEL)                              │
│                                                                                        │
│   ┌──────────────────────────────────────────────────────────────────┐                │
│   │                        SUPERVISOR-F                               │                │
│   │                        (Observability)                            │                │
│   │                        WP-F1 → WP-F5                              │                │
│   └──────────────────────────────────────────────────────────────────┘                │
│                                                                                        │
│   ═══════════════════════════════ GATE 4 ═══════════════════════════════              │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                           PHASE 5: TESTING (PARALLEL)                                  │
│                                                                                        │
│   ┌──────────────────────┐          ┌──────────────────────┐                          │
│   │    SUPERVISOR-G      │          │    SUPERVISOR-H      │                          │
│   │    (Unit/Integration)│  ║║║║║║  │    (E2E Playwright)  │                          │
│   │    WP-G1 → WP-G3     │  ║║║║║║  │    WP-H1 → WP-H5     │                          │
│   └──────────────────────┘          └──────────────────────┘                          │
│                                                                                        │
│   ═══════════════════════════════ GATE 5 ═══════════════════════════════              │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### Per-Work-Package Execution Protocol

For each WP, supervisors follow this 5-step protocol:

```
┌────────────────────────────────────────────────────────────────────────┐
│ STEP 1: PROMPT-WRITER (Haiku, ~30 seconds)                             │
│                                                                        │
│ Supervisor → prompt-writer:                                            │
│ "Generate implementation prompt for WP-D1: Route submissions through   │
│  circuit breaker. Include: fabric-client.ts lines 267-270,            │
│  outbox-submitter.ts line 801, existing circuit breaker pattern."     │
│                                                                        │
│ Output: Context-rich prompt with:                                      │
│ - Exact file paths and line numbers                                   │
│ - Current code snippets                                               │
│ - Expected changes with examples                                       │
│ - Acceptance criteria                                                  │
│ - Verification commands                                                │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ STEP 2: IMPL-AGENT (Sonnet, ~2-5 minutes)                              │
│                                                                        │
│ Supervisor → impl-agent:                                               │
│ [Uses prompt from prompt-writer]                                       │
│                                                                        │
│ Actions:                                                               │
│ - Read specified files                                                 │
│ - Apply code changes                                                   │
│ - Run verification command                                             │
│ - Report success/failure                                               │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                            ┌───────┴───────┐
                         SUCCESS          FAILURE
                            │                │
                            │                ▼
                            │    ┌───────────────────────────────────────┐
                            │    │ STEP 3: DEBUGGER (Sonnet, if needed)  │
                            │    │                                       │
                            │    │ Diagnose root cause:                  │
                            │    │ - Read error output                   │
                            │    │ - Analyze stack trace                 │
                            │    │ - Identify fix                        │
                            │    │ - Return to STEP 1 with new context   │
                            │    │                                       │
                            │    │ Max 3 retry cycles per WP             │
                            │    └───────────────────────────────────────┘
                            │                │
                            └────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ STEP 4: VERIFICATION (By Supervisor)                                   │
│                                                                        │
│ Supervisor verifies:                                                   │
│ - Build passes: npm run build --filter=<service>                       │
│ - Lint passes: npm run lint --filter=<service>                         │
│ - Type checks pass                                                     │
│                                                                        │
│ If verification fails → Back to STEP 3 (debugger)                      │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ STEP 5: WORK-RECORDER (Haiku, ~15 seconds)                             │
│                                                                        │
│ Supervisor → work-recorder:                                            │
│ "Log WP-D1 completion to work record:                                  │
│  - Status: COMPLETE                                                    │
│  - Files modified: fabric-client.ts, outbox-submitter/index.ts        │
│  - Verification: Build passed, lint passed                             │
│  - Commit: [pending/done]"                                             │
│                                                                        │
│ Output: Updated docs/workrecords/work-record-2026-01-28.md             │
└────────────────────────────────────────────────────────────────────────┘
```

### Agent Spawning Reference (Orchestrator Commands)

**Phase 1 - Spawn 2 Supervisors in Parallel:**
```typescript
// Spawn SUPERVISOR-D and SUPERVISOR-E simultaneously
Task({
  prompt: `SUPERVISOR-D: Resilience Track Lead.

  Your team: prompt-writer (Haiku), backend-impl (Sonnet), debugger (Sonnet), work-recorder (Haiku)

  Execute Work Packages: WP-D1, WP-D2, WP-D3, WP-D4, WP-D5

  For each WP:
  1. Spawn prompt-writer to generate context-rich implementation prompt
  2. Spawn backend-impl with the generated prompt
  3. If failure: spawn debugger, then retry with new prompt
  4. Verify build passes
  5. Spawn work-recorder to log progress

  Files: packages/core-fabric/src/fabric-client.ts, workers/outbox-submitter/src/index.ts

  Gate Criteria: All 5 WPs complete, npm run build passes`,
  subagent_type: "general-purpose",
  model: "sonnet"
});

Task({
  prompt: `SUPERVISOR-E: Security Track Lead.

  Your team: prompt-writer (Haiku), backend-impl (Sonnet), debugger (Sonnet), work-recorder (Haiku)

  Execute Work Packages: WP-E1, WP-E2, WP-E3, WP-E4, WP-E5

  [Same protocol as above]

  Files: workers/outbox-submitter/src/index.ts, apps/svc-admin/src/app.ts

  Gate Criteria: All 5 WPs complete, SQL injection fixed, rate limiting added`,
  subagent_type: "general-purpose",
  model: "sonnet"
});
```

**Phase 2 - After Gate 1 passes, spawn 2 more:**
```typescript
Task({ prompt: "SUPERVISOR-A: Code Quality Track...", ... });
Task({ prompt: "SUPERVISOR-B: fabricUserId Track...", ... });
```

### Communication Flow

```
┌─────────────┐                                        ┌─────────────┐
│ ORCHESTRATOR│                                        │ WORK RECORD │
│   (Opus)    │◄──────────────────────────────────────▶│    FILE     │
└──────┬──────┘                                        └─────────────┘
       │
       │ Spawn supervisor with track assignment
       │
       ▼
┌─────────────┐     Spawn with task context     ┌─────────────────┐
│ SUPERVISOR  │────────────────────────────────▶│  PROMPT-WRITER  │
│  (Sonnet)   │◄────────────────────────────────│     (Haiku)     │
└──────┬──────┘     Return optimized prompt     └─────────────────┘
       │
       │ Spawn with optimized prompt
       │
       ▼
┌─────────────┐     On failure, spawn          ┌─────────────────┐
│ IMPL-AGENT  │────────────────────────────────▶│    DEBUGGER     │
│  (Sonnet)   │◄────────────────────────────────│    (Sonnet)     │
└──────┬──────┘     Return diagnosis           └─────────────────┘
       │
       │ On success, spawn
       │
       ▼
┌─────────────────┐
│  WORK-RECORDER  │───────▶ Update work record
│     (Haiku)     │
└─────────────────┘
```

### Concurrency Limits

| Tier | Max Concurrent Agents | Rationale |
|------|----------------------|-----------|
| Supervisors | 2-3 | Balance parallelism with context coherence |
| prompt-writers | 4 | Fast (Haiku), low cost, high throughput |
| impl-agents | 2-3 | Prevent conflicting file edits |
| debuggers | 1-2 | On-demand only when failures occur |
| work-recorders | 2 | Fast (Haiku), non-conflicting writes |

### Error Recovery Protocol

```
IF impl-agent fails:
  ┌─────────────────────────────────────────────────────────────────┐
  │ RETRY CYCLE (Max 3 attempts per WP)                             │
  │                                                                 │
  │ Attempt 1: Original prompt                                      │
  │     └─► Failure                                                 │
  │             └─► debugger diagnoses, prompt-writer refines       │
  │                                                                 │
  │ Attempt 2: Refined prompt with error context                    │
  │     └─► Failure                                                 │
  │             └─► debugger with full trace, prompt-writer         │
  │                 generates minimal targeted fix                  │
  │                                                                 │
  │ Attempt 3: Minimal fix prompt                                   │
  │     └─► Failure                                                 │
  │             └─► ESCALATE to Orchestrator                        │
  │                 Orchestrator spawns Opus architect for          │
  │                 deep analysis or defers WP to next phase        │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Critical Issues Identified

### Issue 1: Race Condition in fabricUserId Generation (CRITICAL)
```
Current Flow (BROKEN):
1. Admin clicks "Register Selected"
2. OutboxCommand created (no fabricUserId yet)
3. Worker picks command, generates fabricUserId
4. If retry occurs → NEW fabricUserId generated → SPLIT IDENTITY ON BLOCKCHAIN
```

**Root Cause**: `enrichRegisterUserPayload()` in `outbox-submitter/src/index.ts:711-763` generates fabricUserId at submission time, not at approval time.

### Issue 2: Fabric Single Event Per Transaction Limitation
```
Current Flow:
1. IdentityContract:CreateUser emits UserCreated event
2. CreateUser calls genesis distribution
3. TokenomicsContract emits TreasuryAllocationEvent
4. TreasuryAllocationEvent OVERWRITES UserCreated (Fabric limitation)
5. Projector never receives UserCreated → can't update user status
```

**Root Cause**: Hyperledger Fabric only allows ONE event per transaction (last SetEvent wins).

### Issue 3: Code Quality Audit Findings (15 Total)
| Severity | Count | Key Issues |
|----------|-------|------------|
| CRITICAL | 1 | Inconsistent error handling in REGISTER_USER_ONCHAIN handler |
| HIGH | 3 | Missing transaction rollback, duplicate lock acquisition, hardcoded constants |
| MEDIUM | 6 | Duplicate code, missing logging, inconsistent null checks |
| LOW | 5 | Magic numbers, missing types, code organization |

### Issue 4: Enterprise Resilience Gaps (Resilience Audit)
| Severity | Issue | Location |
|----------|-------|----------|
| CRITICAL | `submitTransactionWithOptions` BYPASSES circuit breaker | fabric-client.ts:267-270 |
| HIGH | No exponential backoff - retry storms under outage | outbox-submitter:466-496 |
| HIGH | REGISTER_USER_ONCHAIN not idempotent on retry | outbox-submitter:728-729 |
| MEDIUM | Graceful shutdown doesn't drain in-flight commands | outbox-submitter:1949-1958 |
| MEDIUM | No dedicated DLQ status or alerting | outbox-submitter:665-670 |
| MEDIUM | evaluateTransaction calls unbounded (no timeout) | outbox-submitter:1729,1783 |

### Issue 5: Security Vulnerabilities (Security Audit)
| Severity | Issue | Location |
|----------|-------|----------|
| CRITICAL | SQL injection via string interpolation (lockTimeoutSeconds) | outbox-submitter:530 |
| HIGH | Zero rate limiting on all admin endpoints | svc-admin/app.ts |
| MEDIUM | Missing IP/User-Agent in audit entries | all user-management use cases |
| MEDIUM | No input sanitization before blockchain payload | batch-register-onchain.use-case.ts |
| LOW | 10MB body size limit too permissive | svc-admin/app.ts:70 |
| LOW | permissionMiddleware optional fallback (fail-open) | user-management.routes.ts |

### Issue 6: Observability Gaps (Observability Audit)
| Severity | Issue | Location |
|----------|-------|----------|
| MEDIUM | Custom logger duplicates @gx/core-logger | both workers |
| MEDIUM | No correlation ID propagation | both workers |
| MEDIUM | projector_lag_blocks metric declared but never updated | projector:116-119 |
| MEDIUM | Single /health endpoint conflates liveness/readiness | both workers |
| LOW | No alerting hooks for DLQ events | outbox-submitter |
| LOW | No per-command-type label on outbox metrics | outbox-submitter |

---

## Track A: Code Quality Fixes

**Priority**: P0
**Dependencies**: None
**Files**: `workers/outbox-submitter/src/index.ts`

### WP-A1: Fix CRITICAL - Consistent Error Handling

**File**: `workers/outbox-submitter/src/index.ts` (lines 350-425)

**Current Issue**: Mix of throwing errors vs returning early, inconsistent logging patterns.

**Fix**:
```typescript
// Standardize error handling pattern
try {
  // ... operation
} catch (error) {
  logger.error(`[REGISTER_USER_ONCHAIN] Failed: ${command.id}`, { error, userId });
  await this.markCommandFailed(command.id, error.message);
  throw error; // Consistent: always throw for retry logic
}
```

### WP-A2: Fix HIGH - Duplicate Lock Acquisition

**File**: `workers/outbox-submitter/src/index.ts` (lines 180-220)

**Current Issue**: Lock acquired in `processCommand()` AND in individual handlers.

**Fix**: Remove redundant lock acquisition from handlers, keep only in main `processCommand()`.

### WP-A3: Fix HIGH - Extract Hardcoded Constants

**File**: `workers/outbox-submitter/src/index.ts`

**Fix**: Create constants file:
```typescript
// workers/outbox-submitter/src/constants.ts
export const WORKER_CONFIG = {
  POLL_INTERVAL_MS: 5000,
  MAX_RETRIES: 3,
  LOCK_TIMEOUT_MS: 300000,
  BATCH_SIZE: 10,
};
```

### WP-A4: Fix MEDIUM - Consolidate Duplicate Code

**Files**:
- `enrichRegisterUserPayload()` lines 711-763
- `handlePostCommitSideEffects()` lines 1232-1694

**Fix**: Extract shared utilities:
```typescript
// workers/outbox-submitter/src/utils/user-enrichment.ts
export async function fetchUserProfileData(prisma: PrismaClient, profileId: string): Promise<UserProfileData>;
export function mapToFabricUserData(profile: UserProfileData): FabricUserPayload;
```

### WP-A5: Fix MEDIUM - Add Missing Logging

Add structured logging at key decision points:
- Before Fabric submission
- After successful commit
- On retry attempts

---

## Track B: fabricUserId Pre-Generation at KYC Approval

**Priority**: P0 CRITICAL (Fixes race condition)
**Dependencies**: None
**Files**:
- `apps/svc-admin/src/infrastructure/repositories/prisma-kyc-application.repository.ts`
- `workers/outbox-submitter/src/index.ts`

### WP-B1: Generate fabricUserId at KYC Approval

**File**: `prisma-kyc-application.repository.ts` (method `approveApplication()` lines 260-304)

**Current Code**:
```typescript
async approveApplication(applicationId: string): Promise<KycApplication> {
  return this.prisma.$transaction(async (tx) => {
    // ... approval logic
    await tx.userProfile.update({
      where: { profileId: application.profileId },
      data: { status: 'APPROVED_PENDING_ONCHAIN' },
    });
  });
}
```

**New Code**:
```typescript
import { generateFabricUserId } from '@gx/core-fabric';

async approveApplication(applicationId: string): Promise<KycApplication> {
  return this.prisma.$transaction(async (tx) => {
    // ... approval logic

    // Generate fabricUserId at approval time (BEFORE any registration attempt)
    const fabricUserId = generateFabricUserId();

    await tx.userProfile.update({
      where: { profileId: application.profileId },
      data: {
        status: 'APPROVED_PENDING_ONCHAIN',
        fabricUserId, // Store NOW, use LATER
      },
    });
  });
}
```

### WP-B2: Modify Outbox Submitter to Use Existing fabricUserId

**File**: `workers/outbox-submitter/src/index.ts` (function `enrichRegisterUserPayload()` lines 711-763)

**Current Code**:
```typescript
const fabricUserId = generateFabricUserId(); // Generated at submission time (RACE CONDITION)
```

**New Code**:
```typescript
// Use existing fabricUserId from UserProfile (generated at approval)
const userProfile = await this.prisma.userProfile.findUnique({
  where: { profileId: payload.profileId },
  select: { fabricUserId: true },
});

if (!userProfile?.fabricUserId) {
  throw new Error(`fabricUserId not found for profile ${payload.profileId}. Was KYC approved correctly?`);
}

const fabricUserId = userProfile.fabricUserId; // Use existing, don't generate
```

### WP-B3: Add Idempotency Check

**File**: `workers/outbox-submitter/src/index.ts`

Before submitting to Fabric, check if user already exists:
```typescript
// Idempotency: Check if user already registered on blockchain
const existingUser = await this.fabricClient.query('IdentityContract:GetUser', [fabricUserId]);
if (existingUser) {
  logger.info(`User ${fabricUserId} already exists on blockchain, skipping registration`);
  await this.markCommandCommitted(command.id);
  return;
}
```

---

## Track C: Projector Event Handling (Chaincode Fix)

**Priority**: P1
**Dependencies**: Track B complete
**Files**:
- `gx-coin-fabric/chaincode/gxtv3/contracts/identity_contract.go`
- `gx-coin-fabric/chaincode/gxtv3/contracts/tokenomics_contract.go`
- `workers/projector/src/index.ts`

### WP-C1: Create Composite Event in Chaincode

**Problem**: Fabric allows only ONE event per transaction. UserCreated gets overwritten by TreasuryAllocationEvent.

**Solution**: Create composite `UserRegistrationComplete` event containing all data.

**File**: `identity_contract.go` (CreateUser function)

**New Event Structure**:
```go
type UserRegistrationCompleteEvent struct {
    EventType       string                    `json:"eventType"`
    User            UserCreatedData           `json:"user"`
    GenesisTransfer *GenesisTransferData      `json:"genesisTransfer,omitempty"`
    TreasuryUpdate  *TreasuryAllocationData   `json:"treasuryUpdate,omitempty"`
    Timestamp       int64                     `json:"timestamp"`
}

// In CreateUser, after genesis distribution completes:
event := UserRegistrationCompleteEvent{
    EventType: "UserRegistrationComplete",
    User: UserCreatedData{
        FabricUserId: user.FabricUserId,
        ProfileId:    user.ProfileId,
        Status:       user.Status,
    },
    GenesisTransfer: &GenesisTransferData{
        Amount:    genesisAmount,
        ToUserId:  user.FabricUserId,
    },
    TreasuryUpdate: &TreasuryAllocationData{
        NewBalance: treasuryBalance,
    },
    Timestamp: txTimestamp,
}

eventBytes, _ := json.Marshal(event)
ctx.GetStub().SetEvent("UserRegistrationComplete", eventBytes)
```

### WP-C2: Update Projector to Handle Composite Event

**File**: `workers/projector/src/index.ts`

**Add New Handler**:
```typescript
async handleUserRegistrationComplete(event: UserRegistrationCompleteEvent): Promise<void> {
  const { user, genesisTransfer, treasuryUpdate } = event;

  await this.prisma.$transaction(async (tx) => {
    // 1. Update UserProfile status to ACTIVE
    await tx.userProfile.update({
      where: { fabricUserId: user.fabricUserId },
      data: {
        status: 'ACTIVE',
        onchainStatus: 'ACTIVE',
        updatedAt: new Date(),
      },
    });

    // 2. Create/update Wallet with genesis balance
    if (genesisTransfer) {
      await tx.wallet.upsert({
        where: { profileId: user.profileId },
        create: {
          profileId: user.profileId,
          balance: genesisTransfer.amount,
          onchainBalance: genesisTransfer.amount,
        },
        update: {
          balance: genesisTransfer.amount,
          onchainBalance: genesisTransfer.amount,
        },
      });
    }

    // 3. Update treasury pool balance
    if (treasuryUpdate) {
      await tx.pool.update({
        where: { poolType: 'TREASURY' },
        data: { balance: treasuryUpdate.newBalance },
      });
    }
  });

  logger.info(`[PROJECTOR] User registration complete: ${user.fabricUserId}`);
}
```

### WP-C3: Remove Post-Commit Side Effects from Outbox Submitter

**File**: `workers/outbox-submitter/src/index.ts` (lines 1232-1694)

**Change**: Remove direct database updates from `handlePostCommitSideEffects()` for REGISTER_USER_ONCHAIN.

**Rationale**: Projector now handles this via blockchain events, ensuring consistency.

**Keep**: Outbox submitter still marks command as COMMITTED, but doesn't update UserProfile/Wallet directly.

---

## Track D: Resilience Patterns (CRITICAL)

**Priority**: P0 CRITICAL
**Dependencies**: None
**Files**: `workers/outbox-submitter/src/index.ts`, `packages/core-fabric/src/fabric-client.ts`

### WP-D1: CRITICAL - Route All Submissions Through Circuit Breaker

**Problem**: `submitTransactionWithOptions()` BYPASSES the circuit breaker entirely (fabric-client.ts:267-270).

**File**: `packages/core-fabric/src/fabric-client.ts`

**Fix**:
```typescript
// Add circuit breaker wrapper for options-based calls
this.circuitBreakerWithOptions = new CircuitBreaker(
  this.submitTxInternalWithOptions.bind(this),
  {
    timeout: 120000,
    errorThresholdPercentage: 50,
    resetTimeout: 30000,
    volumeThreshold: 5,
    name: 'fabric-gateway-with-options',
  }
);

async submitTransactionWithOptions(options: SubmitTransactionOptions): Promise<TransactionResult> {
  if (!this.isConnected) throw new Error('Not connected');
  return this.circuitBreakerWithOptions.fire(options);
}
```

### WP-D2: HIGH - Add Exponential Backoff with Jitter

**Problem**: Failed commands retry immediately every 100ms poll cycle, creating retry storms.

**File**: `workers/outbox-submitter/src/index.ts` (findAndLockCommands query)

**Fix**: Add backoff delay to the claim query:
```sql
-- Only pick up failed commands after exponential backoff delay
AND (
  status != 'FAILED'
  OR "updatedAt" < NOW() - INTERVAL '1 second' * POWER(2, LEAST(attempts, 6))
)
```

### WP-D3: Add Graceful Shutdown with Drain

**Problem**: SIGTERM immediately disconnects, severing in-flight Fabric transactions.

**File**: `workers/outbox-submitter/src/index.ts` (lines 1949-1958)

**Fix**:
```typescript
private activeCommands = 0;
private shutdownResolve?: () => void;

async stop(): Promise<void> {
  this.isRunning = false;
  if (this.pollTimer) clearTimeout(this.pollTimer);

  // Wait for in-flight commands (max 25s within K8s 30s grace period)
  if (this.activeCommands > 0) {
    await new Promise<void>((resolve) => {
      this.shutdownResolve = resolve;
      setTimeout(resolve, 25000);
    });
  }
  // ... disconnect clients
}
```

### WP-D4: Add Dead Letter Queue Status

**Problem**: Exhausted commands stay in FAILED status, silently excluded from metrics.

**File**: `db/prisma/schema.prisma` (OutboxStatus enum)

**Fix**: Add `DEAD_LETTERED` status:
```prisma
enum OutboxStatus {
  PENDING
  LOCKED
  COMMITTED
  FAILED
  DEAD_LETTERED  // NEW: Commands that exhausted all retries
}
```

Then transition commands to DEAD_LETTERED when max retries exhausted, with alerting webhook.

### WP-D5: Add Timeout to evaluateTransaction Calls

**Problem**: Balance sync queries have no timeout, can block indefinitely.

**File**: `workers/outbox-submitter/src/index.ts` (lines 1729, 1783)

**Fix**:
```typescript
const EVAL_TIMEOUT_MS = 30000;
const balanceBytes = await Promise.race([
  fabricClient.evaluateTransaction('TokenomicsContract', 'GetBalance', fabricUserId),
  new Promise((_, reject) => setTimeout(() => reject(new Error('Timeout')), EVAL_TIMEOUT_MS))
]);
```

---

## Track E: Security Hardening (HIGH)

**Priority**: P0 HIGH
**Dependencies**: None
**Files**: `workers/outbox-submitter/src/index.ts`, `apps/svc-admin/src/app.ts`

### WP-E1: CRITICAL - Fix SQL Injection in findAndLockCommands

**Problem**: `lockTimeoutSeconds` is string-interpolated, not parameterized (line 530).

**File**: `workers/outbox-submitter/src/index.ts`

**Current** (VULNERABLE):
```typescript
`... AND "lockedAt" < NOW() - INTERVAL '${lockTimeoutSeconds} seconds' ...`
```

**Fix** (use computed timestamp parameter):
```typescript
const lockCutoff = new Date(Date.now() - this.config.lockTimeout);
// Use $4 parameter instead of interpolation
`... AND "lockedAt" < $4 ...`
`, this.config.maxRetries, this.config.batchSize, this.config.workerId, lockCutoff);
```

### WP-E2: HIGH - Add Rate Limiting to Admin Endpoints

**Problem**: Zero rate limiting on any admin endpoint including destructive operations.

**File**: `apps/svc-admin/src/app.ts`

**Fix**:
```typescript
import rateLimit from 'express-rate-limit';

// Global rate limiter
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 500,                   // 500 requests per window per IP
  message: { error: 'Rate Limited', code: 'RATE_LIMITED' },
  standardHeaders: true,
});
app.use(globalLimiter);

// Destructive action limiter (freeze, ban, batch-register)
const destructiveLimiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 10,               // 10 destructive ops per minute
});
```

### WP-E3: MEDIUM - Capture IP and User-Agent in Audit Logs

**Problem**: Audit entries missing request origin metadata for forensics.

**Files**: All use cases in `apps/svc-admin/src/application/use-cases/user-management/`

**Fix**: Pass request context to use cases:
```typescript
// Controller
const result = await useCase.execute(dto, {
  adminId,
  ipAddress: req.ip,
  userAgent: req.headers['user-agent'],
});

// Use case - include in audit
await auditLogRepository.create({
  action: 'USER_KYC_APPROVED',
  ipAddress: context.ipAddress,
  userAgent: context.userAgent,
  // ...
});
```

### WP-E4: LOW - Reduce Body Size Limit

**File**: `apps/svc-admin/src/app.ts` (line 70)

**Fix**:
```typescript
app.use(express.json({ limit: '256kb' }));  // Down from 10mb
```

### WP-E5: LOW - Fail Closed on Missing permissionMiddleware

**File**: `apps/svc-admin/src/interface/routes/user-management.routes.ts`

**Fix**: Remove else fallback, require permission middleware:
```typescript
const banPermission = options.permissionMiddleware?.('user:ban');
if (!banPermission) {
  throw new Error('permissionMiddleware required for destructive routes');
}
router.post('/:id/ban', banPermission, controller.banUser);
```

---

## Track F: Observability

**Priority**: P1
**Dependencies**: None
**Files**: `workers/outbox-submitter/src/index.ts`, `workers/projector/src/index.ts`

### WP-F1: Adopt @gx/core-logger with Correlation IDs

**Problem**: Custom log() methods duplicate shared library, no correlation ID propagation.

**Files**: Both workers

**Fix**:
```typescript
import { logger } from '@gx/core-logger';

// In processCommand:
const cmdLogger = logger.child({
  correlationId: command.requestId,
  commandType: command.commandType
});
cmdLogger.info('Processing command');
```

### WP-F2: Fix Dead projector_lag_blocks Metric

**Problem**: Metric defined but never updated (projector/src/index.ts:116-119).

**File**: `workers/projector/src/index.ts`

**Fix**:
```typescript
// After updating lastProcessedBlock:
const chainTip = await this.fabricClient.getBlockHeight();
metrics.projectionLag.set(Number(chainTip - this.lastProcessedBlock));
```

### WP-F3: Separate Liveness and Readiness Probes

**Problem**: Single /health endpoint conflates liveness and readiness.

**Files**: Both workers

**Fix**: Add `/ready` endpoint:
```typescript
} else if (req.url === '/ready') {
  const dbAlive = await this.checkDatabaseConnectivity();
  const fabricAlive = fabricClients.size > 0;
  const lagAcceptable = this.projectionLag < MAX_ACCEPTABLE_LAG;
  const isReady = dbAlive && fabricAlive && lagAcceptable;
  res.statusCode = isReady ? 200 : 503;
  res.end(JSON.stringify({ ready: isReady, dbAlive, fabricAlive, lagAcceptable }));
}
```

### WP-F4: Add DLQ Alerting Webhook

**Problem**: Commands reaching DLQ have no alerting, silent data integrity risk.

**File**: `workers/outbox-submitter/src/index.ts`

**Fix**:
```typescript
if (isMaxRetries) {
  await this.fireAlertWebhook('DLQ', {
    commandId: command.id,
    commandType: command.commandType,
    attempts,
    payload: command.payload
  });
}

private async fireAlertWebhook(type: string, data: any): Promise<void> {
  const webhookUrl = process.env.ALERT_WEBHOOK_URL;
  if (!webhookUrl) return;
  await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ type, ...data, timestamp: new Date().toISOString() }),
  });
}
```

### WP-F5: Add Per-Command-Type Metric Labels

**Problem**: outbox_commands_processed_total only has status label, not command_type.

**File**: `workers/outbox-submitter/src/index.ts`

**Fix**:
```typescript
// Extend counter with command_type label
metrics.commandsProcessed.inc({ status: 'success', command_type: command.commandType });
```

---

## Track G: Unit & Integration Tests

**Priority**: P1
**Dependencies**: Tracks A-F complete
**Files**: New test files

### WP-G1: Unit Tests for fabricUserId Pre-Generation

**File**: `apps/svc-admin/src/__tests__/unit/approve-kyc-application.unit.test.ts`

```typescript
describe('approveApplication', () => {
  it('should generate and store fabricUserId on approval', async () => {
    const result = await repository.approveApplication(applicationId);

    const userProfile = await prisma.userProfile.findUnique({
      where: { profileId: result.profileId },
    });

    expect(userProfile.fabricUserId).toBeDefined();
    expect(userProfile.fabricUserId).toMatch(/^USR-[A-Z0-9]+$/);
    expect(userProfile.status).toBe('APPROVED_PENDING_ONCHAIN');
  });
});
```

### WP-G2: Unit Tests for Outbox Submitter Idempotency

**File**: `workers/outbox-submitter/src/__tests__/register-user.unit.test.ts`

```typescript
describe('REGISTER_USER_ONCHAIN handler', () => {
  it('should use existing fabricUserId from UserProfile', async () => {
    // Setup: User with pre-generated fabricUserId
    const profileId = 'test-profile';
    const existingFabricUserId = 'USR-EXISTING123';

    await prisma.userProfile.update({
      where: { profileId },
      data: { fabricUserId: existingFabricUserId },
    });

    // Execute
    await handler.processCommand(command);

    // Verify: Used existing ID, didn't generate new
    expect(fabricClientMock.invoke).toHaveBeenCalledWith(
      'IdentityContract:CreateUser',
      expect.objectContaining({ fabricUserId: existingFabricUserId })
    );
  });

  it('should skip registration if user already exists on blockchain', async () => {
    // Setup: User already on blockchain
    fabricClientMock.query.mockResolvedValue({ exists: true });

    // Execute
    await handler.processCommand(command);

    // Verify: No invoke, command marked committed
    expect(fabricClientMock.invoke).not.toHaveBeenCalled();
    expect(command.status).toBe('COMMITTED');
  });
});
```

### WP-G3: Integration Tests for Full Registration Flow

**File**: `workers/outbox-submitter/src/__tests__/integration/registration-flow.integration.test.ts`

```typescript
describe('User Registration Flow (Integration)', () => {
  it('should complete full flow: KYC approval → Outbox → Fabric → Projector', async () => {
    // 1. Approve KYC application
    const approval = await kycRepository.approveApplication(applicationId);
    expect(approval.userProfile.fabricUserId).toBeDefined();

    // 2. Create outbox command
    const command = await outboxRepository.createCommand({
      type: 'REGISTER_USER_ONCHAIN',
      payload: { profileId: approval.profileId },
    });

    // 3. Process command (submits to Fabric)
    await outboxWorker.processCommand(command);

    // 4. Verify blockchain state
    const blockchainUser = await fabricClient.query('IdentityContract:GetUser', [approval.userProfile.fabricUserId]);
    expect(blockchainUser.status).toBe('Active');

    // 5. Simulate projector event handling
    await projector.handleUserRegistrationComplete(mockEvent);

    // 6. Verify final database state
    const finalProfile = await prisma.userProfile.findUnique({
      where: { profileId: approval.profileId },
    });
    expect(finalProfile.status).toBe('ACTIVE');
    expect(finalProfile.onchainStatus).toBe('ACTIVE');
  });
});
```

---

## Track H: E2E Playwright MCP Testing

**Priority**: P2
**Dependencies**: All other tracks complete
**Tool**: Playwright MCP browser automation

### WP-H1: Test Case 1 - Happy Path Single User Registration

```
Test Steps:
1. browser_navigate → http://localhost:3001/dashboard/users
2. browser_click → "Pending Onchain" tab
3. Verify → Users with status APPROVED_PENDING_ONCHAIN visible
4. browser_click → Checkbox for first user
5. browser_click → "Register Selected" button
6. browser_wait_for → Success toast/notification
7. browser_network_requests → Verify POST /api/v1/admin/users/batch-register returned 200
8. Wait 10 seconds for worker processing
9. browser_click → "Active" tab
10. Verify → Registered user now appears in Active tab with fabricUserId

Expected Result: User moves from "Pending Onchain" to "Active" tab
```

### WP-H2: Test Case 2 - Batch Registration (Multiple Users)

```
Test Steps:
1. browser_navigate → http://localhost:3001/dashboard/users
2. browser_click → "Pending Onchain" tab
3. browser_click → "Select All" checkbox
4. Verify → Multiple users selected
5. browser_click → "Register Selected" button
6. browser_wait_for → Progress indicator
7. Wait for batch processing complete
8. browser_click → "Active" tab
9. Verify → All selected users now in Active tab

Expected Result: Batch registration processes all selected users
```

### WP-H3: Test Case 3 - Idempotency Verification

```
Test Steps:
1. Complete WP-E1 (register a user)
2. Manually create duplicate OutboxCommand via database
3. Check worker logs → Should skip with "already exists" message
4. Verify → No duplicate user on blockchain
5. Verify → No database corruption

Expected Result: Duplicate commands are safely skipped
```

### WP-H4: Test Case 4 - Error Handling

```
Test Steps:
1. Stop Fabric network (simulate outage)
2. browser_navigate → http://localhost:3001/dashboard/users
3. browser_click → "Pending Onchain" tab
4. browser_click → Register a user
5. Verify → Error message displayed
6. Verify → User remains in "Pending Onchain" (not lost)
7. Start Fabric network
8. Verify → Retry succeeds on next worker poll

Expected Result: Graceful error handling, no data loss
```

### WP-H5: Test Case 5 - Genesis Token Verification

```
Test Steps:
1. Complete WP-E1 (register a user)
2. Query database: SELECT balance FROM "Wallet" WHERE profileId = '<userId>'
3. Verify → Balance equals genesis amount (configured value)
4. Query chaincode: TokenomicsContract:GetBalance
5. Verify → Onchain balance matches database

Expected Result: Genesis tokens distributed and synced correctly
```

---

## Execution Timeline with Agent Spawning

### Orchestrator (Main Claude) Execution Flow

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ PHASE 1: CRITICAL FIXES (Spawn 2 Supervisors in Parallel)                              │
│                                                                                        │
│ Orchestrator spawns:                                                                   │
│                                                                                        │
│ Task("SUPERVISOR-D: Resilience Track Lead...")  ──┐                                   │
│                                                   │ PARALLEL                           │
│ Task("SUPERVISOR-E: Security Track Lead...")  ────┘                                   │
│                                                                                        │
│   SUPERVISOR-D Team:                    SUPERVISOR-E Team:                            │
│   ├── prompt-writer (Haiku)             ├── prompt-writer (Haiku)                     │
│   ├── backend-impl (Sonnet)             ├── backend-impl (Sonnet)                     │
│   ├── debugger (Sonnet, on-demand)      ├── debugger (Sonnet, on-demand)              │
│   └── work-recorder (Haiku)             └── work-recorder (Haiku)                     │
│                                                                                        │
│   WP-D1: Circuit breaker fix            WP-E1: SQL injection fix                      │
│   WP-D2: Exponential backoff            WP-E2: Rate limiting                          │
│   WP-D3: Graceful shutdown              WP-E3: Audit log IP/UA                        │
│   WP-D4: DLQ status                     WP-E4: Body size limit                        │
│   WP-D5: Timeout handling               WP-E5: Fail-closed routes                     │
│                                                                                        │
│   ═══════════════════════════════ GATE 1 ═══════════════════════════════              │
│   Orchestrator verifies: npm run build passes, security tests pass                    │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ PHASE 2: CORE LOGIC (Spawn 2 Supervisors in Parallel)                                  │
│                                                                                        │
│ Task("SUPERVISOR-A: Code Quality Track Lead...")  ──┐                                 │
│                                                     │ PARALLEL                         │
│ Task("SUPERVISOR-B: fabricUserId Track Lead...")  ──┘                                 │
│                                                                                        │
│   SUPERVISOR-A Team:                    SUPERVISOR-B Team:                            │
│   ├── prompt-writer (Haiku)             ├── prompt-writer (Haiku)                     │
│   ├── backend-impl (Sonnet)             ├── backend-impl (Sonnet)                     │
│   ├── debugger (Sonnet, on-demand)      ├── debugger (Sonnet, on-demand)              │
│   └── work-recorder (Haiku)             └── work-recorder (Haiku)                     │
│                                                                                        │
│   WP-A1: Error handling                 WP-B1: fabricUserId at approval               │
│   WP-A2: Remove duplicate locks         WP-B2: Use existing fabricUserId              │
│   WP-A3: Extract constants              WP-B3: Add idempotency check                  │
│   WP-A4: Consolidate code                                                             │
│   WP-A5: Add logging                                                                  │
│                                                                                        │
│   ═══════════════════════════════ GATE 2 ═══════════════════════════════              │
│   Orchestrator verifies: npm run build passes, race condition tests pass              │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ PHASE 3: CHAINCODE (Spawn 1 Supervisor - Sequential Go + TS work)                      │
│                                                                                        │
│ Task("SUPERVISOR-C: Chaincode Track Lead...")                                         │
│                                                                                        │
│   SUPERVISOR-C Team:                                                                  │
│   ├── prompt-writer (Haiku)                                                           │
│   ├── chaincode-impl (Sonnet) → For WP-C1 (Go chaincode)                              │
│   ├── backend-impl (Sonnet) → For WP-C2, WP-C3 (projector TypeScript)                 │
│   ├── debugger (Sonnet, on-demand)                                                    │
│   └── work-recorder (Haiku)                                                           │
│                                                                                        │
│   WP-C1: Create composite event in Go chaincode                                       │
│   WP-C2: Update projector event handler                                               │
│   WP-C3: Remove outbox-submitter post-commit side effects                             │
│                                                                                        │
│   ═══════════════════════════════ GATE 3 ═══════════════════════════════              │
│   Orchestrator verifies: go build passes, chaincode tests pass                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ PHASE 4: OBSERVABILITY (Spawn 1 Supervisor)                                            │
│                                                                                        │
│ Task("SUPERVISOR-F: Observability Track Lead...")                                     │
│                                                                                        │
│   SUPERVISOR-F Team:                                                                  │
│   ├── prompt-writer (Haiku)                                                           │
│   ├── backend-impl (Sonnet)                                                           │
│   ├── debugger (Sonnet, on-demand)                                                    │
│   └── work-recorder (Haiku)                                                           │
│                                                                                        │
│   WP-F1: Adopt @gx/core-logger with correlation IDs                                   │
│   WP-F2: Fix projector_lag_blocks metric                                              │
│   WP-F3: Separate liveness/readiness probes                                           │
│   WP-F4: Add DLQ alerting webhook                                                     │
│   WP-F5: Add per-command-type metric labels                                           │
│                                                                                        │
│   ═══════════════════════════════ GATE 4 ═══════════════════════════════              │
│   Orchestrator verifies: Health endpoints respond correctly                           │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ PHASE 5: TESTING (Spawn 2 Supervisors in Parallel)                                     │
│                                                                                        │
│ Task("SUPERVISOR-G: Testing Track Lead...")  ──────┐                                  │
│                                                    │ PARALLEL                          │
│ Task("SUPERVISOR-H: E2E Track Lead...")  ──────────┘                                  │
│                                                                                        │
│   SUPERVISOR-G Team:                    SUPERVISOR-H Team:                            │
│   ├── prompt-writer (Haiku)             ├── prompt-writer (Haiku)                     │
│   ├── tester (Sonnet)                   ├── tester (Sonnet + Playwright MCP)          │
│   ├── debugger (Sonnet, on-demand)      ├── debugger (Sonnet, on-demand)              │
│   └── work-recorder (Haiku)             └── work-recorder (Haiku)                     │
│                                                                                        │
│   WP-G1: Unit tests fabricUserId        WP-H1: E2E happy path                         │
│   WP-G2: Unit tests idempotency         WP-H2: E2E batch registration                 │
│   WP-G3: Integration tests              WP-H3: E2E idempotency                        │
│                                         WP-H4: E2E error handling                     │
│                                         WP-H5: E2E genesis tokens                     │
│                                                                                        │
│   ═══════════════════════════════ GATE 5 ═══════════════════════════════              │
│   Orchestrator verifies: All tests pass, E2E scenarios complete                       │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ PHASE 6: FINAL VERIFICATION & COMMIT                                                   │
│                                                                                        │
│ Orchestrator (Main Claude):                                                           │
│ 1. Run full build: npm run build                                                       │
│ 2. Run all tests: npm test                                                            │
│ 3. Stage and commit all changes using /commit skill                                    │
│ 4. Update work record with final status                                               │
│                                                                                        │
│   ════════════════════════ MODULE-4 COMPLETE ═════════════════════════                │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### Total Agent Count Per Phase

| Phase | Supervisors | prompt-writers | impl-agents | debuggers | work-recorders | Total Active |
|-------|-------------|----------------|-------------|-----------|----------------|--------------|
| 1 | 2 | 2-4 | 2-4 | 0-2 | 2 | 8-14 |
| 2 | 2 | 2-4 | 2-4 | 0-2 | 2 | 8-14 |
| 3 | 1 | 1-2 | 1-2 | 0-1 | 1 | 4-6 |
| 4 | 1 | 1-2 | 1-2 | 0-1 | 1 | 4-6 |
| 5 | 2 | 2-4 | 2-4 | 0-2 | 2 | 8-14 |
| **Peak** | **2** | **4** | **4** | **2** | **2** | **14** |

---

## Files to Modify

### Track A (Code Quality)
- `workers/outbox-submitter/src/index.ts`
- `workers/outbox-submitter/src/constants.ts` (NEW)
- `workers/outbox-submitter/src/utils/user-enrichment.ts` (NEW)

### Track B (fabricUserId Pre-Generation)
- `apps/svc-admin/src/infrastructure/repositories/prisma-kyc-application.repository.ts`
- `workers/outbox-submitter/src/index.ts`

### Track C (Projector Event Handling)
- `gx-coin-fabric/chaincode/gxtv3/contracts/identity_contract.go`
- `gx-coin-fabric/chaincode/gxtv3/contracts/tokenomics_contract.go`
- `workers/projector/src/index.ts`

### Track D (Resilience Patterns)
- `packages/core-fabric/src/fabric-client.ts` (circuit breaker fix)
- `workers/outbox-submitter/src/index.ts` (backoff, shutdown, timeouts)
- `db/prisma/schema.prisma` (DEAD_LETTERED status)

### Track E (Security Hardening)
- `workers/outbox-submitter/src/index.ts` (SQL injection fix)
- `apps/svc-admin/src/app.ts` (rate limiting, body limit)
- `apps/svc-admin/src/interface/routes/user-management.routes.ts` (fail closed)
- `apps/svc-admin/src/application/use-cases/user-management/*.ts` (audit context)

### Track F (Observability)
- `workers/outbox-submitter/src/index.ts` (logging, metrics)
- `workers/projector/src/index.ts` (logging, metrics, readiness)

### Track G (Tests)
- `apps/svc-admin/src/__tests__/unit/approve-kyc-application.unit.test.ts` (NEW)
- `workers/outbox-submitter/src/__tests__/register-user.unit.test.ts` (NEW)
- `workers/outbox-submitter/src/__tests__/integration/registration-flow.integration.test.ts` (NEW)

---

## Per-Work-Package Protocol

See **Multi-Agent Orchestration Architecture** section above for the detailed 5-step protocol:
1. **PROMPT-WRITER** (Haiku) → Generate context-rich prompt
2. **IMPL-AGENT** (Sonnet) → Execute with optimized prompt
3. **DEBUGGER** (Sonnet) → Error recovery if needed (max 3 retries)
4. **VERIFICATION** → Build/lint check by supervisor
5. **WORK-RECORDER** (Haiku) → Log to work record

---

## Quality Gates

### Gate 1: After Tracks D & E (Security & Resilience)
```bash
cd gx-protocol-backend
npm run build  # All packages compile with security fixes
npm run lint   # No linting errors

# Verify SQL injection fix
grep -n "queryRawUnsafe" workers/outbox-submitter/src/index.ts
# Should show parameterized lockCutoff, not interpolated lockTimeoutSeconds
```

### Gate 2: After Tracks A & B (Code Quality & Race Condition Fix)
```bash
cd gx-protocol-backend
npm run build  # All packages compile
npm run lint   # No linting errors

# Verify fabricUserId is stored at approval
grep -n "fabricUserId" apps/svc-admin/src/infrastructure/repositories/prisma-kyc-application.repository.ts
```

### Gate 3: After Track C (Chaincode & Projector)
```bash
cd gx-coin-fabric/chaincode/gxtv3
go build ./...  # Chaincode compiles
go test ./...   # Chaincode tests pass

cd gx-protocol-backend
npm run build --filter=projector
```

### Gate 4: After Track F (Observability)
```bash
# Verify logger adoption
grep -n "@gx/core-logger" workers/outbox-submitter/src/index.ts
grep -n "@gx/core-logger" workers/projector/src/index.ts

# Verify /ready endpoint exists
grep -n "/ready" workers/outbox-submitter/src/index.ts
```

### Gate 5: After Track G (Tests)
```bash
cd gx-protocol-backend
npm test -- --filter=svc-admin
npm test -- --filter=outbox-submitter
```

### Gate 6: After Track H (E2E)
```
All Playwright MCP E2E scenarios pass
Work record updated with final status
```

---

## Success Criteria

| Metric | Target |
|--------|--------|
| **Security** | |
| SQL Injection | ✅ Fixed (parameterized query) |
| Rate Limiting | ✅ Implemented (global + destructive) |
| Audit Logging | ✅ IP/User-Agent captured |
| **Resilience** | |
| Circuit Breaker | ✅ All Fabric calls protected |
| Exponential Backoff | ✅ Retry storms eliminated |
| Graceful Shutdown | ✅ In-flight commands drained |
| Dead Letter Queue | ✅ DEAD_LETTERED status + alerting |
| **Core Logic** | |
| Code Quality Issues | 0/15 remaining |
| Race Condition | ✅ Eliminated |
| fabricUserId Generation | ✅ At KYC approval |
| Projector Event Handling | ✅ Composite event |
| **Observability** | |
| Structured Logging | ✅ @gx/core-logger with correlation IDs |
| Metrics | ✅ All gauges functional, command_type labels |
| Health Probes | ✅ Separate liveness/readiness |
| **Testing** | |
| Unit Test Coverage | ✅ 80%+ |
| Integration Tests | ✅ All pass |
| E2E Tests | ✅ 5/5 scenarios pass |
| **Builds** | |
| Backend Build | ✅ 0 errors |
| Chaincode Build | ✅ 0 errors |

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Chaincode deployment fails | Test in DevNet first, rollback plan ready |
| Existing users missing fabricUserId | Migration script to backfill |
| Projector not deployed | Verify K8s deployment before Track C |
| E2E tests flaky | Use explicit waits, retry mechanism |
| Rate limiter blocks legitimate traffic | Start with high limits, tune based on metrics |
| Circuit breaker too aggressive | Use conservative thresholds (50% error, 5 volume) |
| Schema migration for DEAD_LETTERED | Apply during low-traffic window |

---

## Commit Strategy

```bash
# Track D (Resilience) - FIRST
git commit -m "fix(fabric): route all submissions through circuit breaker"
git commit -m "feat(outbox): add exponential backoff with jitter for retries"
git commit -m "feat(outbox): implement graceful shutdown with drain"
git commit -m "feat(db): add DEAD_LETTERED status to OutboxStatus enum"
git commit -m "fix(outbox): add timeout to evaluateTransaction calls"

# Track E (Security)
git commit -m "security(outbox): fix SQL injection in findAndLockCommands"
git commit -m "security(admin): add rate limiting to all endpoints"
git commit -m "feat(admin): capture IP and User-Agent in audit logs"
git commit -m "security(admin): reduce body size limit to 256kb"
git commit -m "security(admin): fail closed on missing permissionMiddleware"

# Track A (Code Quality)
git commit -m "refactor(outbox): standardize error handling patterns"
git commit -m "refactor(outbox): extract constants and remove duplicates"

# Track B (fabricUserId)
git commit -m "feat(kyc): generate fabricUserId at KYC approval"
git commit -m "fix(outbox): use pre-generated fabricUserId, add idempotency"

# Track C (Chaincode & Projector)
git commit -m "feat(chaincode): create composite UserRegistrationComplete event"
git commit -m "feat(projector): handle UserRegistrationComplete event"

# Track F (Observability)
git commit -m "refactor(workers): adopt @gx/core-logger with correlation IDs"
git commit -m "fix(projector): update projector_lag_blocks metric"
git commit -m "feat(workers): add separate readiness endpoint"
git commit -m "feat(outbox): add DLQ alerting webhook"
git commit -m "feat(outbox): add command_type label to metrics"

# Track G (Tests)
git commit -m "test(admin): add unit tests for KYC approval fabricUserId"
git commit -m "test(outbox): add integration tests for registration flow"

# Track H (E2E)
git commit -m "docs(workrecords): add E2E test results for user registration"
```

---

## MODULE-4 Completion Status After This Session

| Component | Before | After |
|-----------|--------|-------|
| **Security** | | |
| SQL Injection Vulnerability | ❌ Present | ✅ Fixed |
| Rate Limiting | ❌ None | ✅ Implemented |
| Audit Trail | ⚠️ Partial | ✅ Complete |
| **Resilience** | | |
| Circuit Breaker | ⚠️ Bypassed | ✅ All paths protected |
| Retry Logic | ❌ Retry storms | ✅ Exponential backoff |
| Graceful Shutdown | ❌ Abrupt | ✅ Drains in-flight |
| Dead Letter Queue | ❌ Silent | ✅ Status + Alerting |
| **Core Logic** | | |
| Code Quality | 15 issues | ✅ 0 issues |
| fabricUserId Race Condition | ❌ Present | ✅ Fixed |
| Projector Integration | ❌ Broken | ✅ Working |
| **Observability** | | |
| Structured Logging | ⚠️ Custom | ✅ @gx/core-logger |
| Correlation IDs | ❌ None | ✅ Implemented |
| Metrics | ⚠️ Incomplete | ✅ All functional |
| Health Probes | ⚠️ Combined | ✅ Separate liveness/readiness |
| **Testing** | | |
| Unit Tests | Minimal | ✅ 80%+ coverage |
| E2E Tests | None | ✅ 5 scenarios |
| **MODULE-4** | **85%** | **100%** |

---

## Enterprise Best Practices Checklist

This plan addresses the following enterprise patterns:

### Resilience
- [x] Circuit breaker for all external calls
- [x] Exponential backoff with jitter
- [x] Dead letter queue with alerting
- [x] Graceful shutdown
- [x] Idempotent operations
- [x] Timeout handling

### Security
- [x] Input validation (Zod)
- [x] Parameterized queries
- [x] Rate limiting
- [x] Audit logging with full context
- [x] Fail-closed authorization

### Observability
- [x] Structured logging with correlation IDs
- [x] Prometheus metrics with proper labels
- [x] Separate liveness/readiness probes
- [x] Alerting for critical failures

### Testing
- [x] Unit tests for business logic
- [x] Integration tests for cross-component flows
- [x] E2E tests for user journeys

---

## Notes

1. **Backward Compatibility**: Existing users without fabricUserId need migration
2. **Chaincode Deployment**: Requires peer restart after update
3. **Projector Must Be Running**: Deploy before Track C verification
4. **DevNet Only**: All changes target DevNet, not TestNet/MainNet
5. **Rate Limiter Tuning**: Start conservative, adjust based on production metrics
6. **Alerting Webhook**: Configure ALERT_WEBHOOK_URL environment variable
