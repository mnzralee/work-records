# Phase 7: svc-admin Clean Architecture Migration Plan

**Date:** 2026-01-20
**Parallel Work:** This runs alongside the svc-government migration (handled by another agent)
**Complexity:** HIGH (14 controllers, 13 services, ~115 use cases)

---

## Executive Summary

Refactor `svc-admin` from legacy pattern (controller → service) to Clean Architecture following the **established patterns from svc-auth and svc-tokenomics** (NOT svc-government which has pattern deviations).

**Location:** `/home/dev-manazir/prod-blockchain/gx-protocol-backend/apps/svc-admin/`

---

## Pattern Alignment (CRITICAL)

**Reference Services (Correct Patterns):**
- `apps/svc-auth/` - Gold standard for Clean Architecture
- `apps/svc-tokenomics/` - Advanced DI container pattern

**Pattern Deviations to Avoid (from svc-government):**
1. ❌ Static readonly error properties → Use constructor-based
2. ❌ Direct route exports → Use factory functions
3. ❌ Missing `timestamp` and `details` in errors → Include these fields

---

## Naming Conventions (Strict Alignment with svc-auth/svc-tokenomics)

| Aspect | Convention | Example |
|--------|------------|---------|
| **File Names** | kebab-case with suffix | `get-admin.use-case.ts` |
| **Repository Ports** | `I<Entity>Repository` | `IAdminUserRepository` |
| **Port File Names** | `<entity>.repository.port.ts` | `admin-user.repository.port.ts` |
| **Repository Impl** | `prisma-<entity>.repository.ts` | `prisma-admin-user.repository.ts` |
| **Use Case Classes** | `<Verb><Entity>UseCase` | `LoginUseCase`, `GetAdminUseCase` |
| **Use Case Files** | `<verb-entity>.use-case.ts` | `login.use-case.ts` |
| **Controller Classes** | `<Entity>Controller` | `AuthController` |
| **Controller Files** | `<entity>.controller.ts` | `auth.controller.ts` |
| **Route Factories** | `create<Entity>Routes(controller)` | `createAuthRoutes(authController)` |
| **Route Files** | `<entity>.routes.ts` | `auth.routes.ts` |

---

## Target Structure

```
src/
├── domain/
│   └── errors/index.ts              # ~40 domain-specific errors
├── application/
│   ├── use-cases/
│   │   ├── auth/                    # 15 use cases
│   │   ├── rbac/                    # 12 use cases
│   │   ├── admin/                   # 15 use cases
│   │   ├── approval/                # 7 use cases
│   │   ├── user-management/         # 9 use cases
│   │   ├── entity-accounts/         # 17 use cases (3 sub-modules)
│   │   ├── audit/                   # 8 use cases
│   │   ├── dashboard/               # 6 use cases
│   │   ├── deployment/              # 8 use cases
│   │   ├── session/                 # 12 use cases
│   │   ├── notification/            # 9 use cases
│   │   └── reconciliation/          # 6 use cases
│   ├── ports/repositories/          # 12 repository interfaces
│   └── dto/
│       ├── request/                 # Zod-validated DTOs
│       └── response/                # Response DTOs
├── infrastructure/
│   ├── repositories/                # Prisma implementations
│   └── services/                    # External service adapters (Fabric, K8s)
├── interface/
│   ├── controllers/                 # 12 THIN controllers
│   └── routes/                      # Express route factories
└── shared/di/
    └── container.ts                 # Dependency injection
```

---

## Feature Module Organization (10 Modules)

| Module | Controllers | Use Cases | Priority |
|--------|-------------|-----------|----------|
| **auth** | admin-auth | 15 | CRITICAL |
| **rbac** | rbac | 12 | HIGH |
| **admin** | admin | 15 | HIGH |
| **approval** | approval | 7 | MEDIUM |
| **user-management** | user-management | 9 | MEDIUM |
| **entity-accounts** | entity-accounts | 17 | MEDIUM |
| **audit** | audit | 8 | MEDIUM |
| **dashboard** | dashboard | 6 | LOW |
| **deployment** | deployment | 8 | MEDIUM |
| **session** | session, device | 12 | MEDIUM |
| **notification** | notification | 9 | LOW |
| **reconciliation** | reconciliation | 6 | LOW |

---

## Implementation Phases

### Phase 7.1: Foundation (Domain Errors)
**Files to create:**
- `src/domain/errors/index.ts` - ~40 domain errors

**Base Error Pattern (from svc-auth/svc-tokenomics - CORRECT):**
```typescript
// domain/errors/index.ts
export abstract class DomainError extends Error {
  public readonly code: string;
  public readonly statusCode: number;
  public readonly timestamp: string;
  public readonly details?: Record<string, unknown>;

  constructor(
    code: string,
    statusCode: number,
    message: string,
    details?: Record<string, unknown>
  ) {
    super(message);
    this.name = this.constructor.name;
    this.code = code;
    this.statusCode = statusCode;
    this.timestamp = new Date().toISOString();
    this.details = details;
    Error.captureStackTrace(this, this.constructor);
  }
}

// Example concrete error
export class InvalidCredentialsError extends DomainError {
  constructor(details?: Record<string, unknown>) {
    super('INVALID_CREDENTIALS', 401, 'Invalid username or password', details);
  }
}
```

**Error Categories:**
- Auth: InvalidCredentialsError, AccountLockedError, MfaRequiredError, SessionExpiredError, etc.
- RBAC: PermissionNotFoundError, RoleNotFoundError, InsufficientPrivilegesError
- Approval: ApprovalNotFoundError, ApprovalExpiredError, SelfApprovalNotAllowedError
- User Management: UserNotFoundError, UserAlreadyFrozenError, InvalidFreezeReasonError
- Entity Accounts: BusinessAccountNotFoundError, GovernmentAccountNotFoundError, NPOAccountNotFoundError
- Deployment: DeploymentNotFoundError, InvalidPromotionPathError, HealthCheckFailedError
- Session/Audit/Notification: SessionNotFoundError, AuditLogNotFoundError, etc.

### Phase 7.2: Repository Ports
**Files to create in `application/ports/repositories/`:**
- `admin-user.repository.port.ts` - Admin user CRUD
- `admin-session.repository.port.ts` - Session management
- `rbac.repository.port.ts` - Permissions and roles
- `approval.repository.port.ts` - Approval requests
- `user-profile.repository.port.ts` - User management
- `entity-account.repository.port.ts` - Business/Government/NPO
- `audit-log.repository.port.ts` - Audit queries
- `deployment.repository.port.ts` - Deployment records
- `notification.repository.port.ts` - Notifications & webhooks
- `reconciliation.repository.port.ts` - Balance reconciliation
- `dashboard.repository.port.ts` - Aggregation queries
- `system-parameter.repository.port.ts` - System config
- `index.ts` - Export all

### Phase 7.3: DTOs (Request/Response)
**Request DTOs (with Zod validation):**
```
dto/request/
├── auth/           # login, mfa-verify, password-change, mfa-enable
├── rbac/           # grant-permission, revoke-permission, update-role
├── approval/       # create-approval, vote-approval, list-approvals
├── user-management/# list-users, approve-user, deny-user, freeze-user
├── entity-accounts/# list-business, list-government, list-npo
├── deployment/     # create-deployment, rollback
├── audit/          # query-logs
├── session/        # query-sessions, update-device
├── notification/   # create-webhook, update-webhook
├── reconciliation/ # run-reconciliation, resolve-discrepancy
└── index.ts
```

**Response DTOs:**
```
dto/response/
├── auth/           # login, session, mfa-setup, profile
├── rbac/           # permission, role, permission-matrix
├── approval/       # approval
├── user-management/# user
├── entity-accounts/# business, government, npo
├── deployment/     # deployment
├── audit/          # audit-log
├── dashboard/      # stats
├── session/        # session
├── notification/   # notification, webhook
├── reconciliation/ # reconciliation
└── index.ts
```

### Phase 7.4: Use Cases (By Module)

**Auth Module (15 use cases)** - CRITICAL PATH
- login, verify-mfa, refresh-token, logout, logout-all-sessions
- get-active-sessions, revoke-session, change-password
- setup-mfa, enable-mfa, disable-mfa
- get-profile, validate-session, update-session-activity

**RBAC Module (12 use cases)** - HIGH PRIORITY
- get-all-permissions, get-permission-by-code, get-permissions-by-category
- get-all-roles, get-role-details, get-permission-matrix
- get-admin-permissions, grant-permission, revoke-permission
- update-admin-role, bulk-update-permissions, check-permission

**Admin Module (15 use cases)**
- bootstrap-system, initialize-country-data, update-system-parameter
- get-system-parameter, pause-system, resume-system
- appoint-admin, activate-treasury, get-system-status
- list-admins, get-admin-by-id, get-supply-status
- get-pool-status, get-public-supply, get-country-allocations

**Remaining modules follow same pattern...**

### Phase 7.5: Infrastructure Repositories
**Files to create in `infrastructure/repositories/`:**
- `prisma-admin-user.repository.ts`
- `prisma-admin-session.repository.ts`
- `prisma-rbac.repository.ts`
- `prisma-approval.repository.ts`
- `prisma-user-profile.repository.ts`
- `prisma-entity-account.repository.ts`
- `prisma-audit-log.repository.ts`
- `prisma-deployment.repository.ts`
- `prisma-notification.repository.ts`
- `prisma-reconciliation.repository.ts`
- `prisma-dashboard.repository.ts`
- `prisma-system-parameter.repository.ts`
- `index.ts`

### Phase 7.6: Interface Layer (Controllers & Routes)

**Controller Pattern (from svc-auth - CORRECT):**
```typescript
// interface/controllers/auth.controller.ts
export class AuthController {
  constructor(
    private readonly loginUseCase: LoginUseCase,
    private readonly logoutUseCase: LogoutUseCase,
    private readonly refreshTokenUseCase: RefreshTokenUseCase,
  ) {}

  // Arrow function methods (preserve `this` context)
  login = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const dto = req.body as LoginRequestDto;
      const result = await this.loginUseCase.execute(dto);

      res.status(200).json({
        success: true,
        data: result,
      });
    } catch (error) {
      next(error); // Delegate to error middleware
    }
  };

  logout = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      await this.logoutUseCase.execute({ sessionId: req.user.sessionId });
      res.status(200).json({ success: true });
    } catch (error) {
      next(error);
    }
  };
}
```

**Controllers (THIN - HTTP concerns only):**
- `auth.controller.ts`
- `rbac.controller.ts`
- `admin.controller.ts`
- `approval.controller.ts`
- `user-management.controller.ts`
- `entity-accounts.controller.ts`
- `audit.controller.ts`
- `dashboard.controller.ts`
- `deployment.controller.ts`
- `session.controller.ts`
- `notification.controller.ts`
- `reconciliation.controller.ts`
- `health.controller.ts` (infrastructure only)

**Route Factories:**
- One route factory per controller
- Apply auth middleware via `@gx/core-http`
- Apply permission checks via RBAC middleware

### Phase 7.7: DI Container & Integration

**DI Container Pattern (from svc-tokenomics - CORRECT):**
```typescript
// shared/di/container.ts
export interface Container {
  // Repositories
  adminUserRepository: PrismaAdminUserRepository;
  adminSessionRepository: PrismaAdminSessionRepository;
  rbacRepository: PrismaRbacRepository;
  // ... more repositories

  // Use Cases
  loginUseCase: LoginUseCase;
  logoutUseCase: LogoutUseCase;
  // ... more use cases

  // Controllers
  authController: AuthController;
  rbacController: RbacController;
  // ... more controllers
}

let containerInstance: Container | null = null;

export function createContainer(): Container {
  // ========================================
  // Infrastructure Layer - Repositories
  // ========================================
  const adminUserRepository = new PrismaAdminUserRepository();
  const adminSessionRepository = new PrismaAdminSessionRepository();

  // ========================================
  // Application Layer - Use Cases
  // ========================================
  const loginUseCase = new LoginUseCase(adminUserRepository, adminSessionRepository);

  // ========================================
  // Interface Layer - Controllers
  // ========================================
  const authController = new AuthController(loginUseCase, logoutUseCase);

  return {
    adminUserRepository,
    adminSessionRepository,
    loginUseCase,
    authController,
    // ... all dependencies
  };
}

export function getContainer(): Container {
  if (!containerInstance) {
    containerInstance = createContainer();
  }
  return containerInstance;
}

export function resetContainer(): void {
  containerInstance = null;
}
```

**Route Factory Pattern (from svc-auth - CORRECT):**
```typescript
// interface/routes/auth.routes.ts
import { Router } from 'express';
import { AuthController } from '../controllers';
import { validateBody } from '@gx/core-http';
import { loginRequestSchema } from '../../application/dto/request';

export function createAuthRoutes(controller: AuthController): Router {
  const router = Router();

  router.post('/login', validateBody(loginRequestSchema), controller.login);
  router.post('/logout', authMiddleware, controller.logout);
  router.post('/refresh', validateBody(refreshTokenSchema), controller.refresh);

  return router;
}
```

**App.ts Integration Pattern (from svc-auth - CORRECT):**
```typescript
// app.ts
import { createContainer } from './shared/di';
import { createAuthRoutes, createRbacRoutes } from './interface/routes';

export function createApp(): Application {
  const app = express();

  // Middleware
  app.use(helmet());
  app.use(cors());
  app.use(express.json());

  // Create container (dependency injection)
  const container = createContainer();

  // Mount routes with injected controllers
  app.use('/api/v1/admin/auth', createAuthRoutes(container.authController));
  app.use('/api/v1/admin/rbac', createRbacRoutes(container.rbacController));
  // ... more routes

  // Error handling
  app.use(errorHandler);

  return app;
}
```

**Files to create:**
- `shared/di/container.ts` - Wire all dependencies
- `shared/di/index.ts` - Export container functions
- Update `app.ts` to use new routes from container
- Register routes on `/api/v1/admin/*` (maintain compatibility)

### Phase 7.8: Remove Legacy Directories
After verification:
- Remove `src/controllers/` (old)
- Remove `src/services/` (old)
- Remove `src/routes/` (old)
- Keep `src/types/` → migrate to `dto/` if needed

---

## Critical Files Reference

| File | Purpose |
|------|---------|
| `apps/svc-tokenomics/src/shared/di/container.ts` | DI pattern reference |
| `apps/svc-tokenomics/src/domain/errors/index.ts` | Error pattern reference |
| `apps/svc-admin/src/services/admin-auth.service.ts` | Auth logic to extract (846 LOC) |
| `apps/svc-admin/src/controllers/rbac.controller.ts` | RBAC logic to refactor (689 LOC) |
| `apps/svc-admin/src/controllers/entity-accounts.controller.ts` | Largest feature scope |

---

## Implementation Order (Recommended)

1. **Domain Errors** (foundation for all modules)
2. **Auth Module** (blocking - all other modules need auth)
3. **RBAC Module** (blocking - permissions affect all operations)
4. **Admin Module** (system bootstrap, status)
5. **Approval Module** (multi-sig workflow)
6. **User Management Module** (KYC approval)
7. **Entity Accounts Module** (3 sub-modules)
8. **Session/Audit/Dashboard** (observability)
9. **Deployment/Notification/Reconciliation** (specialized)
10. **DI Container & Integration**
11. **Remove Legacy Code**

---

## API Compatibility

**Maintain existing routes:**
- `/api/v1/admin/auth/*` - Authentication
- `/api/v1/admin/rbac/*` - Role management
- `/api/v1/admin/approvals/*` - Approvals
- `/api/v1/admin/users/*` - User management
- `/api/v1/admin/audit/*` - Audit logs
- `/api/v1/admin/deployments/*` - Deployments
- `/api/v1/admin/dashboard/*` - Statistics
- `/api/v1/admin/sessions/*` - Session management
- `/api/v1/admin/notifications/*` - Notifications
- `/api/v1/admin/reconciliation/*` - Reconciliation
- `/api/v1/admin/webhooks/*` - Webhooks
- `/api/v1/public/*` - Public supply info

**gx-admin-frontend compatibility:**
- No frontend changes required
- Backend maintains same API contract
- Response formats unchanged

---

## Verification

After migration:
1. `npx turbo build --filter=svc-admin --force` - Build without cache
2. `npx tsc --noEmit -p apps/svc-admin/tsconfig.json` - Type check
3. Test auth flow: login → MFA → token refresh → logout
4. Test RBAC: permission checks on protected endpoints
5. Test approval workflow: create → vote → execute
6. Deploy to backend-dev-manazir namespace
7. Verify health checks respond

---

## Coordination with Other Agent (svc-government)

**Identified Pattern Misalignments in svc-government:**

The other agent working on svc-government has used patterns that deviate from svc-auth/svc-tokenomics:

| Aspect | svc-auth/svc-tokenomics (CORRECT) | svc-government (DEVIATION) |
|--------|-----------------------------------|----------------------------|
| Error Properties | Constructor-injected | Static readonly |
| Error Fields | Has `timestamp` + `details` | Missing these fields |
| Routes | Factory functions `createXxxRoutes()` | Direct exports |
| DI in Routes | Container passed to factories | Routes don't use container |

**Recommendation:**
- svc-admin will follow svc-auth/svc-tokenomics patterns
- Consider aligning svc-government in a follow-up task

---

## Notes

- This is a large refactoring effort (~115 use cases)
- Auth and RBAC are foundational - complete these first
- Entity Accounts is the largest module (17 use cases across 3 entity types)
- Frontend (gx-admin-frontend) won't need changes - API remains compatible
- Consider incremental migration if time-constrained
- Reference svc-auth for all patterns (gold standard)

---

## Phase 7.9: Error Resolution Completion (2026-01-21)

**Current State:** 573 TypeScript errors remaining
**Target:** 0 errors, successful build

### Error Distribution Analysis

| Error Code | Count | Description |
|------------|-------|-------------|
| TS2339 | 321 | Property does not exist on type |
| TS2353 | 70 | Object literal property mismatch |
| TS2345 | 49 | Argument type mismatch |
| TS7006 | 43 | Parameter implicitly has 'any' type |
| Other | 90 | Various type mismatches |

### Priority Fixes (High Impact)

#### 1. CreateAuditLogInput - Make Fields Optional (~34 use cases affected)
**File:** `src/application/ports/repositories/audit-log.repository.port.ts`

Current:
```typescript
export interface CreateAuditLogInput {
  eventType: AuditEventType;
  actorId: string;
  actorType: string;  // REQUIRED - causing errors
  description: string; // REQUIRED - causing errors
  // ...
}
```

Fix:
```typescript
export interface CreateAuditLogInput {
  eventType: AuditEventType;
  actorId: string;
  actorType?: string;  // OPTIONAL
  description?: string; // OPTIONAL
  // ...
}
```

#### 2. Deployment Repository Port Enhancement
**File:** `src/application/ports/repositories/deployment.repository.port.ts`

Add missing methods:
- `findMany(options?: DeploymentQueryOptions): Promise<DeploymentReadModel[]>`
- `updateStatus(id: string, status: DeploymentStatus, details?: Record<string, unknown>): Promise<DeploymentReadModel>`
- `execute(id: string, executedBy: string): Promise<DeploymentReadModel>`
- `executeRollback(id: string, rollbackBy: string, reason?: string): Promise<DeploymentReadModel>`

Add missing properties to DeploymentReadModel:
- `components?: string[]`
- `createdBy?: string`
- `rollbackFrom?: string | null`
- `promotedFrom?: string | null`
- `notes?: string | null`
- `rollbackReason?: string | null`

#### 3. Fix Duplicate DTO Definitions
**Files:**
- `src/application/dto/request/index.ts` - BulkRevokeSessionsRequestDTO defined twice
- `src/application/dto/response/index.ts` - BulkRevokeSessionsResponseDTO defined twice

Fix: Remove duplicate definitions, keep the more comprehensive one.

### Medium Priority Fixes

#### 4. Entity Account Repository Port
**File:** `src/application/ports/repositories/entity-account.repository.port.ts`

Add missing methods:
- `approve(id: string, approvedBy: string, notes?: string): Promise<EntityAccountReadModel>`
- `deny(id: string, deniedBy: string, reason: string): Promise<EntityAccountReadModel>`
- `freeze(id: string, frozenBy: string, reason: string): Promise<EntityAccountReadModel>`
- `unfreeze(id: string, unfrozenBy: string, notes?: string): Promise<EntityAccountReadModel>`
- `updateKycStatus(id: string, status: EntityStatus, updatedBy: string): Promise<EntityAccountReadModel>`

#### 5. Notification Repository Port
**File:** `src/application/ports/repositories/notification.repository.port.ts`

Add missing methods:
- `markRead(id: string): Promise<NotificationReadModel>`
- `markAllRead(recipientId: string): Promise<{ success: boolean; count: number }>`
- `getUnreadCount(recipientId: string): Promise<number>`
- `deleteNotification(id: string): Promise<void>`
- `getPreferences(adminId: string): Promise<NotificationPreferences>`
- `updatePreferences(adminId: string, prefs: Partial<NotificationPreferences>): Promise<NotificationPreferences>`

#### 6. Reconciliation Repository Port
**File:** `src/application/ports/repositories/reconciliation.repository.port.ts`

Add missing methods:
- `findMany(options?: ReconciliationQueryOptions): Promise<ReconciliationReadModel[]>`
- `getStatus(id: string): Promise<{ status: ReconciliationStatus; progress: number }>`
- `exportReport(id: string, format: 'json' | 'csv'): Promise<{ data: unknown; filename: string }>`

### Lower Priority Fixes (TS7006 Implicit Any)

#### 7. Add Type Annotations to Lambda Parameters
Many use cases have callbacks without explicit types:

```typescript
// Before (error)
items.map(item => ({ ... }))

// After (fixed)
items.map((item: ApprovalReadModel) => ({ ... }))
```

Affected files (~43 locations):
- `src/application/use-cases/approval/*.use-case.ts`
- `src/application/use-cases/audit/*.use-case.ts`
- `src/application/use-cases/deployment/*.use-case.ts`
- `src/application/use-cases/session/*.use-case.ts`
- `src/application/use-cases/notification/*.use-case.ts`

### Implementation Order

1. **Quick Wins (Est. reduction: ~150 errors)**
   - Remove duplicate DTO definitions
   - Make CreateAuditLogInput.actorType and description optional
   - Add property aliases to deployment repository port

2. **Repository Port Enhancements (Est. reduction: ~200 errors)**
   - Deployment repository methods
   - Entity account repository methods
   - Notification repository methods
   - Reconciliation repository methods

3. **Type Annotations (Est. reduction: ~50 errors)**
   - Add explicit types to lambda parameters
   - Fix implicit any in use case implementations

4. **Remaining Fixes (Est. reduction: ~173 errors)**
   - Object literal mismatches
   - Argument type mismatches
   - Method signature alignments

### Verification Steps

After reaching 0 errors:
1. Run full type check: `npx tsc --noEmit -p apps/svc-admin/tsconfig.json`
2. Run build: `npx turbo build --filter=svc-admin --force`
3. Verify no runtime import errors
4. Commit all changes with descriptive messages
5. Push to origin

### Critical Files to Modify

| File | Changes |
|------|---------|
| `ports/repositories/audit-log.repository.port.ts` | Make actorType/description optional |
| `ports/repositories/deployment.repository.port.ts` | Add methods, properties |
| `ports/repositories/entity-account.repository.port.ts` | Add approve/deny/freeze methods |
| `ports/repositories/notification.repository.port.ts` | Add notification management methods |
| `ports/repositories/reconciliation.repository.port.ts` | Add export/status methods |
| `dto/request/index.ts` | Fix duplicate DTOs |
| `dto/response/index.ts` | Fix duplicate DTOs |
| Various use-case files | Add type annotations |
