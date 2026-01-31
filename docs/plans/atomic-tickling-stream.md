# GX Protocol Backend Architecture v2.1 Update

## Status: PHASE 3 - Service Decomposition

**Current Task:** Extract svc-auth from svc-identity (Phase 3.1)

---

## IMPLEMENTATION PLAN: svc-auth Extraction

### Scope Summary

Extract authentication, registration, and session management into a new `svc-auth` microservice following Clean Architecture patterns established in svc-tokenomics.

**Key Finding from Chaincode Analysis:** svc-auth is purely off-chain. On-chain identity creation (`IdentityContract:CreateUser`) happens via svc-admin AFTER KYC approval. svc-auth does NOT interact with Hyperledger Fabric.

### Files to Extract from svc-identity

| Category | Source File | New Location |
|----------|-------------|--------------|
| **Controllers** | `controllers/auth.controller.ts` | Refactor to `interface/controllers/` |
| | `controllers/registration.controller.ts` | Refactor to `interface/controllers/` |
| | `controllers/user-security.controller.ts` | Refactor to `interface/controllers/` |
| **Services** | `services/auth.service.ts` | Split into use cases |
| | `services/registration.service.ts` | Split into use cases |
| | `services/user-security.service.ts` | Split into use cases |
| | `services/otp-cache.service.ts` | `infrastructure/services/` |
| | `services/email.service.ts` | `infrastructure/services/` |
| | `services/sms.service.ts` | `infrastructure/services/` |
| **Routes** | `routes/auth.routes.ts` | `interface/routes/` |
| | `routes/registration.routes.ts` | `interface/routes/` |
| | `routes/user-security.routes.ts` | `interface/routes/` |
| **DTOs** | `types/dtos.ts` (auth-related) | `application/dto/` |

### New svc-auth Directory Structure

```
apps/svc-auth/
├── package.json
├── tsconfig.json
├── Dockerfile
└── src/
    ├── index.ts                     # Entry point
    ├── app.ts                       # Express factory
    ├── config.ts                    # Zod-validated config
    │
    ├── domain/
    │   └── errors/
    │       ├── invalid-credentials.error.ts
    │       ├── user-not-found.error.ts
    │       ├── user-already-exists.error.ts
    │       ├── otp-expired.error.ts
    │       ├── max-otp-attempts.error.ts
    │       ├── pin-lockout.error.ts
    │       └── index.ts
    │
    ├── application/
    │   ├── use-cases/
    │   │   ├── auth/
    │   │   │   ├── login.use-case.ts
    │   │   │   ├── logout.use-case.ts
    │   │   │   ├── refresh-token.use-case.ts
    │   │   │   └── index.ts
    │   │   ├── registration/
    │   │   │   ├── check-email.use-case.ts
    │   │   │   ├── verify-email-otp.use-case.ts
    │   │   │   ├── check-phone.use-case.ts
    │   │   │   ├── verify-phone-otp.use-case.ts
    │   │   │   ├── complete-registration.use-case.ts
    │   │   │   └── index.ts
    │   │   └── security/
    │   │       ├── set-pin.use-case.ts
    │   │       ├── verify-pin.use-case.ts
    │   │       ├── enable-2fa.use-case.ts
    │   │       ├── verify-2fa.use-case.ts
    │   │       └── index.ts
    │   ├── ports/
    │   │   ├── repositories/
    │   │   │   ├── user.repository.port.ts
    │   │   │   ├── session.repository.port.ts
    │   │   │   ├── security-settings.repository.port.ts
    │   │   │   └── index.ts
    │   │   └── services/
    │   │       ├── password-hasher.port.ts
    │   │       ├── jwt-service.port.ts
    │   │       ├── otp-service.port.ts
    │   │       ├── email-service.port.ts
    │   │       ├── sms-service.port.ts
    │   │       └── index.ts
    │   └── dto/
    │       ├── request/
    │       │   ├── login.dto.ts
    │       │   ├── register.dto.ts
    │       │   ├── verify-otp.dto.ts
    │       │   └── index.ts
    │       └── response/
    │           ├── auth-response.dto.ts
    │           ├── user-profile.dto.ts
    │           └── index.ts
    │
    ├── infrastructure/
    │   ├── repositories/
    │   │   ├── prisma-user.repository.ts
    │   │   ├── prisma-session.repository.ts
    │   │   ├── prisma-security-settings.repository.ts
    │   │   └── index.ts
    │   └── services/
    │       ├── bcrypt-password-hasher.ts
    │       ├── jwt.service.ts
    │       ├── redis-otp.service.ts      # Replace in-memory with Redis
    │       ├── resend-email.service.ts
    │       ├── twilio-sms.service.ts
    │       └── index.ts
    │
    ├── interface/
    │   ├── controllers/
    │   │   ├── auth.controller.ts
    │   │   ├── registration.controller.ts
    │   │   ├── security.controller.ts
    │   │   ├── health.controller.ts
    │   │   └── index.ts
    │   └── routes/
    │       ├── auth.routes.ts
    │       ├── registration.routes.ts
    │       ├── security.routes.ts
    │       ├── health.routes.ts
    │       └── index.ts
    │
    └── shared/
        └── di/
            ├── container.ts
            └── index.ts
```

### API Endpoints

| Method | Path | Auth | Rate Limit |
|--------|------|------|------------|
| POST | `/api/v1/auth/login` | No | 5/min |
| POST | `/api/v1/auth/register` | No | 5/min |
| POST | `/api/v1/auth/refresh` | No | 60/min |
| POST | `/api/v1/auth/logout` | Yes | 60/min |
| GET | `/api/v1/auth/validate` | Yes (internal) | 1000/min |
| POST | `/api/v1/registration/check-email` | No | 60/min |
| POST | `/api/v1/registration/verify-email-otp` | No | 10/min |
| POST | `/api/v1/registration/check-phone` | No | 60/min |
| POST | `/api/v1/registration/verify-phone-otp` | No | 10/min |
| POST | `/api/v1/registration/complete` | No | 60/min |
| POST | `/api/v1/security/pin` | Yes | 10/min |
| POST | `/api/v1/security/pin/verify` | Yes | 10/min |
| POST | `/api/v1/security/2fa/enable` | Yes | 5/min |
| POST | `/api/v1/security/2fa/verify` | Yes | 10/min |
| GET | `/health`, `/readyz`, `/livez` | No | - |

### Dependencies

```json
{
  "dependencies": {
    "@gx/core-config": "*",
    "@gx/core-db": "*",
    "@gx/core-http": "*",
    "@gx/core-logger": "*",
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "helmet": "^8.1.0",
    "jsonwebtoken": "^9.0.2",
    "qrcode": "^1.5.3",
    "resend": "^2.0.0",
    "speakeasy": "^2.0.0",
    "twilio": "^4.19.0",
    "zod": "^3.22.4"
  }
}
```

### Configuration (config.ts)

```typescript
const configSchema = z.object({
  port: z.coerce.number().default(3041),
  nodeEnv: z.enum(['development', 'production', 'test']),
  databaseUrl: z.string().url(),
  redisUrl: z.string().url(),
  jwtSecret: z.string().min(32),
  jwtExpiresIn: z.string().default('24h'),
  jwtRefreshExpiresIn: z.string().default('7d'),
  resendApiKey: z.string(),
  emailFromAddress: z.string().email(),
  emailFromName: z.string().default('GXCoin'),
  emailEnabled: z.coerce.boolean().default(false),
  twilioAccountSid: z.string(),
  twilioAuthToken: z.string(),
  twilioPhoneNumber: z.string(),
  smsEnabled: z.coerce.boolean().default(false),
});
```

### Implementation Steps

1. **Create Service Scaffold** (~30 min)
   - Create `apps/svc-auth/` directory structure
   - Copy package.json, tsconfig.json from svc-tokenomics template
   - Add svc-auth-specific dependencies

2. **Domain Layer** (~1 hr)
   - Create domain errors (InvalidCredentials, UserNotFound, OtpExpired, etc.)
   - Port error handling patterns from svc-tokenomics

3. **Application Layer - Ports** (~1 hr)
   - Define repository interfaces (IUserRepository, ISessionRepository)
   - Define service interfaces (IPasswordHasher, IJwtService, IOtpService, IEmailService, ISmsService)

4. **Application Layer - Use Cases** (~3 hrs)
   - Login, Logout, RefreshToken use cases
   - Registration use cases (CheckEmail, VerifyEmailOtp, CheckPhone, VerifyPhoneOtp, Complete)
   - Security use cases (SetPin, VerifyPin, Enable2FA, Verify2FA)

5. **Application Layer - DTOs** (~1 hr)
   - Create Zod schemas for all request DTOs
   - Create response DTOs

6. **Infrastructure Layer** (~2 hrs)
   - Implement PrismaUserRepository, PrismaSessionRepository
   - Implement BcryptPasswordHasher, JwtService
   - Implement RedisOtpService (upgrade from in-memory)
   - Implement ResendEmailService, TwilioSmsService

7. **Interface Layer** (~2 hrs)
   - Create thin controllers (delegate to use cases)
   - Create routes with validation middleware and rate limiters

8. **Shared Layer** (~30 min)
   - Create DI container
   - Wire up all dependencies

9. **Entry Point** (~30 min)
   - Create app.ts (Express factory)
   - Create index.ts (startup with graceful shutdown)
   - Create config.ts (Zod validation)

10. **Kubernetes Deployment** (~30 min)
    - Create svc-auth-deployment.yaml
    - Create svc-auth-service.yaml
    - Update ingress for auth routes

### Critical Files to Create (Priority Order)

1. `apps/svc-auth/package.json`
2. `apps/svc-auth/tsconfig.json`
3. `apps/svc-auth/src/config.ts`
4. `apps/svc-auth/src/domain/errors/index.ts`
5. `apps/svc-auth/src/application/ports/repositories/*.ts`
6. `apps/svc-auth/src/application/ports/services/*.ts`
7. `apps/svc-auth/src/application/dto/request/*.ts`
8. `apps/svc-auth/src/application/use-cases/auth/*.ts`
9. `apps/svc-auth/src/application/use-cases/registration/*.ts`
10. `apps/svc-auth/src/infrastructure/repositories/*.ts`
11. `apps/svc-auth/src/infrastructure/services/*.ts`
12. `apps/svc-auth/src/interface/controllers/*.ts`
13. `apps/svc-auth/src/interface/routes/*.ts`
14. `apps/svc-auth/src/shared/di/container.ts`
15. `apps/svc-auth/src/app.ts`
16. `apps/svc-auth/src/index.ts`

### Verification Checklist

- [ ] Service builds: `cd apps/svc-auth && npm run build`
- [ ] TypeScript passes: `npx tsc --noEmit`
- [ ] Login endpoint works: `curl -X POST http://localhost:3041/api/v1/auth/login`
- [ ] Registration flow works (email OTP → phone OTP → complete)
- [ ] JWT validation endpoint works for other services
- [ ] Security endpoints work (PIN, 2FA)
- [ ] Health check responds: `curl http://localhost:3041/health`
- [ ] Integration tests pass with Testcontainers

### Migration Strategy (Strangler Fig)

**Week 1: Shadow Mode**
- Deploy svc-auth alongside svc-identity
- Route 10% traffic to new service
- Compare responses, log discrepancies

**Week 2: Gradual Rollout**
- Increase to 50%, then 100% traffic
- Monitor error rates, latency

**Week 3: Cutover**
- 100% traffic to svc-auth
- Remove auth code from svc-identity
- Update internal service references

---

## Summary

The architecture plan has been updated to v2.1 with enterprise-standard refinements based on expert review.

### Progress Tracker

| Phase | Task | Status |
|-------|------|--------|
| Phase 1 | Infrastructure (core-errors, core-config, core-http, test-utils) | ✅ COMPLETE |
| Phase 2 | Pilot Migration (svc-tokenomics Clean Architecture) | ✅ COMPLETE |
| Phase 2b | Auth Middleware Consolidation (9 services) | ✅ COMPLETE |
| **Phase 3.1** | **Service Decomposition - svc-auth** | 🔄 READY TO START |
| Phase 3.2 | Service Decomposition - svc-kyc | ⏳ PENDING |
| Phase 3.3 | Service Decomposition - svc-trust | ⏳ PENDING |
| Phase 3.4 | Service Decomposition - svc-wallet | ⏳ PENDING |
| Phase 3.5 | Service Decomposition - svc-business | ⏳ PENDING |
| Phase 3.6 | Service Decomposition - svc-institutional | ⏳ PENDING |

### What Was Done

**Files Updated:**
- `docs/backend-plan/ARCHITECTURE-REFACTORING-PLAN.md` - Updated to v2.1
- `docs/workrecords/work-record-2026-01-19.md` - Added Session 2-5 documentation
- Auth middlewares consolidated across 9 services to use @gx/core-http

### Key Changes in v2.1

| Change | Type | Details |
|--------|------|---------|
| RLS FORCE + Role Hardening | Security | `FORCE ROW LEVEL SECURITY`, non-login schema owner, verification tests |
| OTEL_SERVICE_NAME | Standards | Use OpenTelemetry standard env var |
| Network Error Retry | Correctness | Handle ECONNREFUSED, ETIMEDOUT, AbortError |
| Pact record-deployment | Operational | Post-deploy CI step for broker state |
| Shared DB Mitigations | Risk | Least privilege roles, schema isolation checklist |
| Sagas/Chaos Drills | Deferred | Explicitly marked as Phase 3+ |

### Updated Phase 2 Exit Gate

```
- [ ] RLS enabled + FORCE RLS + role hardening applied
- [ ] RLS proof test in CI
- [ ] OpenTelemetry traces visible
- [ ] RED metrics dashboard per service
- [ ] Idempotency keys for mutations
- [ ] Circuit breaker on service calls
- [ ] Pact contract published + verified
- [ ] record-deployment in CI/CD
```

### Philosophy Applied

> "Enterprise standards without over-engineering"

- **Included**: Security-critical, standards compliance, correctness, operational requirements
- **Deferred**: Premature optimizations (sagas, distributed locks, chaos engineering)

---

## Original Plan Reference

The complete architecture plan is at: `docs/backend-plan/ARCHITECTURE-REFACTORING-PLAN.md`

---

## Part 1: Current State Analysis (COMPLETED)

### Codebase Metrics

| Service | Files | LOC | Controllers | Purpose |
|---------|-------|-----|-------------|---------|
| svc-identity | 111 | 43,936 | 34 | Auth, KYC, profiles, wallets, relationships |
| svc-admin | 53 | 14,775 | 13 | Admin portal, RBAC, approvals |
| svc-government | 36 | 6,537 | 9 | Government treasury, multi-sig |
| svc-organization | 23 | 6,314 | 5 | Organization management |
| svc-tokenomics | 13 | 2,389 | 2 | Token transfers, balances |
| Others | 89 | 20,914 | - | Messaging, onboarding, governance, loans, tax |

### Critical Issues Identified

1. **Monolith within Microservices**: svc-identity handles 34 controllers across unrelated domains
2. **No Repository Pattern**: Services directly use Prisma without abstraction
3. **Mixed Concerns in Controllers**: Validation logic mixed with request handling
4. **No Use Case Separation**: Business logic tightly coupled in services
5. **Code Duplication**: Auth middleware, config schemas duplicated across services
6. **Inconsistent Structure**: No standard internal organization pattern

---

## Part 2: Enterprise Internal Service Structure

### Recommended Architecture: Clean Architecture + Hexagonal Pattern

```
apps/svc-{name}/src/
├── domain/                    # Core Business Logic (innermost layer)
│   ├── entities/             # Domain entities with behavior
│   │   ├── user.entity.ts
│   │   └── index.ts
│   ├── value-objects/        # Immutable domain concepts
│   │   ├── email.vo.ts
│   │   ├── money.vo.ts
│   │   └── index.ts
│   ├── services/             # Domain services (pure logic, no I/O)
│   │   └── velocity-tax-calculator.ts
│   ├── events/               # Domain events
│   │   └── user-registered.event.ts
│   └── errors/               # Domain-specific errors
│       └── insufficient-balance.error.ts
│
├── application/              # Application Layer (use cases)
│   ├── use-cases/           # Application use cases (feature-based)
│   │   ├── auth/
│   │   │   ├── login.use-case.ts
│   │   │   ├── register.use-case.ts
│   │   │   └── index.ts
│   │   └── user/
│   │       ├── get-profile.use-case.ts
│   │       └── update-profile.use-case.ts
│   ├── ports/               # Interfaces (contracts)
│   │   ├── repositories/    # Repository interfaces
│   │   │   └── user.repository.port.ts
│   │   └── services/        # External service interfaces
│   │       └── email.service.port.ts
│   ├── dto/                 # Application DTOs (Zod validated)
│   │   ├── request/
│   │   │   └── register.dto.ts
│   │   └── response/
│   │       └── user-profile.dto.ts
│   └── mappers/             # Entity <-> DTO mappers
│       └── user.mapper.ts
│
├── infrastructure/          # External Adapters (implementations)
│   ├── repositories/        # Repository implementations
│   │   └── prisma-user.repository.ts
│   ├── services/            # External service implementations
│   │   ├── resend-email.service.ts
│   │   └── bcrypt-password-hasher.ts
│   └── persistence/         # Database-specific concerns
│       └── prisma.client.ts
│
├── interface/               # Interface Adapters (HTTP layer)
│   ├── controllers/         # HTTP controllers (THIN!)
│   │   └── auth.controller.ts
│   ├── routes/              # Express route definitions
│   │   └── auth.routes.ts
│   ├── middlewares/         # HTTP middlewares
│   │   └── validation.middleware.ts
│   └── validators/          # Zod validation schemas
│       └── auth.validators.ts
│
├── shared/                  # Cross-cutting concerns
│   ├── di/                  # Dependency injection
│   │   ├── container.ts
│   │   └── types.ts
│   └── constants/
│
├── app.ts                   # Express app factory
└── index.ts                 # Entry point
```

### Key Patterns

#### 1. Repository Pattern with Prisma

```typescript
// application/ports/repositories/user.repository.port.ts
export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(user: User): Promise<User>;
  update(user: User): Promise<User>;
}
export const USER_REPOSITORY = Symbol('USER_REPOSITORY');

// infrastructure/repositories/prisma-user.repository.ts
@injectable()
export class PrismaUserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    const record = await db.userProfile.findUnique({ where: { profileId: id } });
    return record ? UserMapper.toDomain(record) : null;
  }
  // ... other methods
}
```

#### 2. Use Case Pattern (Single Responsibility)

```typescript
// application/use-cases/auth/register.use-case.ts
@injectable()
export class RegisterUseCase {
  constructor(
    @inject(USER_REPOSITORY) private userRepo: IUserRepository,
    @inject(EMAIL_SERVICE) private emailService: IEmailService,
    @inject(PASSWORD_HASHER) private hasher: IPasswordHasher,
  ) {}

  async execute(dto: RegisterRequestDTO): Promise<AuthResponseDTO> {
    // 1. Check if user exists
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) throw new UserAlreadyExistsError(dto.email);

    // 2. Hash password
    const hash = await this.hasher.hash(dto.password);

    // 3. Create user entity
    const user = User.create({ email: dto.email, passwordHash: hash, ...dto });

    // 4. Persist
    const saved = await this.userRepo.create(user);

    // 5. Send welcome email (fire and forget)
    this.emailService.sendWelcome(saved.email).catch(console.error);

    // 6. Return response
    return { user: UserMapper.toDTO(saved), token: this.generateToken(saved) };
  }
}
```

#### 3. DTO Validation with Zod

```typescript
// application/dto/request/register.dto.ts
import { z } from 'zod';

export const registerRequestSchema = z.object({
  email: z.string().email().transform(v => v.toLowerCase()),
  password: z.string().min(8).regex(/[A-Z]/).regex(/[a-z]/).regex(/[0-9]/),
  firstName: z.string().min(1).max(100),
  lastName: z.string().min(1).max(100),
  dateOfBirth: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  gender: z.enum(['male', 'female']),
});

export type RegisterRequestDTO = z.infer<typeof registerRequestSchema>;
```

#### 4. Thin Controllers

```typescript
// interface/controllers/auth.controller.ts
@injectable()
export class AuthController {
  constructor(
    @inject(RegisterUseCase) private registerUseCase: RegisterUseCase,
  ) {}

  // THIN: Only handles HTTP concerns, delegates ALL logic to use case
  register = async (req: Request, res: Response, next: NextFunction) => {
    try {
      // Validation already done by middleware
      const result = await this.registerUseCase.execute(req.body);
      res.status(201).json({ success: true, data: result });
    } catch (error) {
      next(error);
    }
  };
}
```

#### 5. Validation Middleware

```typescript
// interface/middlewares/validation.middleware.ts
export function validateBody<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body);
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(400).json({
          success: false,
          error: {
            code: 'VALIDATION_ERROR',
            message: 'Request validation failed',
            details: error.errors.map(e => ({
              field: e.path.join('.'),
              message: e.message,
            })),
          },
        });
      }
      next(error);
    }
  };
}
```

#### 6. Entity Mapper

```typescript
// application/mappers/user.mapper.ts
export class UserMapper {
  // Prisma → Domain Entity
  static toDomain(record: PrismaUserProfile): User {
    return User.reconstitute({
      id: record.profileId,
      email: Email.create(record.email),
      // ... other fields
    });
  }

  // Domain Entity → Prisma
  static toPersistence(entity: User): PrismaData {
    return {
      profileId: entity.id,
      email: entity.email.value,
      // ... other fields
    };
  }

  // Domain Entity → Response DTO
  static toDTO(entity: User): UserProfileDTO {
    return {
      profileId: entity.id,
      email: entity.email.value,
      // ... other fields
    };
  }
}
```

---

## Part 3: Service Decomposition (svc-identity Split) - Enterprise Standard

### Domain-Driven Design Analysis

**Current State:** svc-identity is a monolith with 32 controllers, 50+ services, 28 routes (~44K LOC).

**Identified Bounded Contexts:**

| # | Bounded Context | Core Aggregate | Key Entities |
|---|-----------------|----------------|--------------|
| 1 | Identity & Authentication | UserProfile | Session, TrustedDevice, RefreshToken |
| 2 | Compliance & KYC | KYCApplication | KYCDocument, AttorneyVerification, SanctionsScreening |
| 3 | Personal Wallet | Wallet | SubAccount, Transaction, Budget, Goal, AllocationRule |
| 4 | Trust & Relationships | TrustScore | FamilyRelationship, FinancialBehaviorScore |
| 5 | Business Entity Accounts | BusinessAccount | BusinessSignatory, Employee, PayrollBatch |
| 6 | Institutional Accounts | NPOAccount/GovtAccount | NPOProgram, NPODonation, Grant, Beneficiary |

### Proposed New Microservices (6 Services)

| Service | Controllers | Bounded Context | Est. LOC | Team |
|---------|-------------|-----------------|----------|------|
| **svc-auth** | auth, registration, user-security, users (partial) | Identity & Auth | ~5K | Platform |
| **svc-kyc** | kyc, documents, attorney, sanctions | Compliance | ~8K | Compliance |
| **svc-wallet** | wallet, subaccounts, budgets, goals, analytics, 8+ more | Personal Wallet | ~12K | Consumer |
| **svc-trust** | relationships | Trust & Relationships | ~4K | Data Science |
| **svc-business** | business-subaccounts, employees, business-reports | Business Entity | ~8K | Enterprise |
| **svc-institutional** | government, npo, npo-grant, npo-beneficiary, ngo-types | Institutional | ~10K | Public Sector |

### Context Map (Service Relationships)

```
                        ┌─────────────────────┐
                        │      svc-auth       │
                        │    (Upstream)       │
                        └─────────────────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
    │   svc-kyc    │      │  svc-wallet  │      │  svc-trust   │
    │ (Downstream) │      │ (Downstream) │      │ (Downstream) │
    └──────────────┘      └──────────────┘      └──────────────┘
                                  │                     │
                    ┌─────────────┴─────────────┐       │
                    │                           │       │
                    ▼                           ▼       │
            ┌──────────────┐            ┌──────────────┐
            │ svc-business │            │svc-institutional│
            │ (Downstream) │            │  (Downstream)  │
            └──────────────┘            └──────────────┘
```

### Service Communication Patterns

**Synchronous HTTP (Required):**
| Caller | Callee | Purpose |
|--------|--------|---------|
| All services | svc-auth | JWT validation, user lookup |
| svc-wallet | svc-trust | Trust score for transaction limits |
| svc-business | svc-auth | Signatory validation |

**Asynchronous Events:**
| Event | Producer | Consumers |
|-------|----------|-----------|
| `UserRegistered` | svc-auth | svc-kyc, svc-trust, svc-wallet |
| `KYCApplicationApproved` | svc-kyc | svc-auth, svc-wallet |
| `TransactionCommitted` | svc-projector | svc-wallet, svc-trust |
| `TrustScoreUpdated` | svc-trust | svc-wallet |

### Database Architecture

**Strategy:** Database per Service with schema separation

| Service | Schema | Justification |
|---------|--------|---------------|
| svc-auth | `auth` | Core identity, high availability |
| svc-kyc | `kyc` | Compliance isolation, GDPR |
| svc-wallet | `wallet` | High write volume |
| svc-trust | `trust` | Graph queries (Neo4j future) |
| svc-business | `business` | Enterprise isolation |
| svc-institutional | `institutional` | Public sector compliance |

### Migration Strategy (Strangler Fig Pattern)

**Phase 1 (Week 1-4):** Extract svc-auth (CRITICAL - upstream provider)
**Phase 2 (Week 5-7):** Extract svc-kyc (clear compliance boundary)
**Phase 3 (Week 8-9):** Extract svc-trust (low risk, async)
**Phase 4 (Week 10-13):** Extract svc-wallet (highest traffic)
**Phase 5 (Week 14-16):** Extract svc-business (enterprise feature)
**Phase 6 (Week 17-19):** Extract svc-institutional (public sector)

**Total Duration:** ~19 weeks (with parallel work: ~12 weeks)

### API Gateway Routing Strategy

| Current Route | New Service | New Route |
|---------------|-------------|-----------|
| `/api/v1/auth/*` | svc-auth | `/api/v1/auth/*` |
| `/api/v1/registration/*` | svc-auth | `/api/v1/auth/registration/*` |
| `/api/v1/kyc/*` | svc-kyc | `/api/v1/kyc/*` |
| `/api/v1/documents/*` | svc-kyc | `/api/v1/kyc/documents/*` |
| `/api/v1/wallets/*` | svc-wallet | `/api/v1/wallets/*` |
| `/api/v1/relationships/*` | svc-trust | `/api/v1/trust/relationships/*` |
| `/api/v1/business-accounts/*` | svc-business | `/api/v1/business/*` |
| `/api/v1/government-accounts/*` | svc-institutional | `/api/v1/institutional/government/*` |
| `/api/v1/npo-accounts/*` | svc-institutional | `/api/v1/institutional/npo/*` |

---

## Part 3.1: IMPLEMENTATION - Phase 1: Extract svc-auth

### Overview

**Service:** svc-auth
**Purpose:** Core identity management, authentication, session management, authorization
**Risk Level:** CRITICAL - Upstream provider for all other services
**Estimated LOC:** ~5K

### Why svc-auth First?

1. **Upstream Provider:** All other services depend on svc-auth for JWT validation
2. **Clear Boundary:** Authentication is a well-defined domain
3. **Foundation:** Establishes service-to-service communication patterns
4. **Testing Ground:** Validates the migration strategy before larger extractions

### Source Files to Extract from svc-identity

```
FROM: apps/svc-identity/src/
├── controllers/
│   ├── auth.controller.ts
│   ├── registration.controller.ts
│   ├── user-security.controller.ts
│   └── users.controller.ts (profile read operations only)
├── services/
│   ├── auth.service.ts
│   ├── registration.service.ts
│   ├── user-security.service.ts
│   ├── users.service.ts (partial)
│   ├── otp-cache.service.ts
│   ├── email.service.ts
│   └── sms.service.ts
└── routes/
    ├── auth.routes.ts
    ├── registration.routes.ts
    ├── user-security.routes.ts
    └── users.routes.ts (partial)
```

### API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/auth/login` | No | User login |
| POST | `/api/v1/auth/register` | No | User registration |
| POST | `/api/v1/auth/refresh` | Yes | Refresh access token |
| POST | `/api/v1/auth/logout` | Yes | Invalidate session |
| GET | `/api/v1/auth/validate` | Yes | JWT validation (internal) |
| GET | `/api/v1/users/:profileId` | Yes | Get user profile |
| PUT | `/api/v1/users/:profileId` | Yes | Update profile |
| POST | `/api/v1/registration/check-email` | No | Email availability + OTP |
| POST | `/api/v1/registration/verify-email-otp` | No | Verify email OTP |
| POST | `/api/v1/registration/check-phone` | No | Phone availability + OTP |
| POST | `/api/v1/registration/verify-phone-otp` | No | Verify phone OTP |
| POST | `/api/v1/registration/complete` | No | Complete registration |
| GET | `/api/v1/security/settings` | Yes | Security settings |
| PUT | `/api/v1/security/settings` | Yes | Update security settings |
| GET | `/health`, `/readyz`, `/livez` | No | Health checks |

### Data Entities Owned

- `UserProfile` (core fields: profileId, email, passwordHash, status)
- `UserSession`
- `TrustedDevice`
- `PushToken`
- `UserSecuritySettings`
- `UserDevice`

### New Service Directory Structure (Clean Architecture)

```
apps/svc-auth/
├── package.json
├── tsconfig.json
├── Dockerfile
└── src/
    ├── index.ts
    ├── app.ts
    ├── config.ts
    │
    ├── domain/
    │   ├── entities/
    │   │   ├── user.entity.ts
    │   │   └── session.entity.ts
    │   ├── value-objects/
    │   │   ├── email.vo.ts
    │   │   └── password.vo.ts
    │   └── errors/
    │       ├── invalid-credentials.error.ts
    │       └── user-not-found.error.ts
    │
    ├── application/
    │   ├── dto/
    │   │   ├── request/
    │   │   │   ├── login.dto.ts
    │   │   │   ├── register.dto.ts
    │   │   │   └── refresh-token.dto.ts
    │   │   └── response/
    │   │       ├── auth-response.dto.ts
    │   │       └── user-profile.dto.ts
    │   ├── ports/
    │   │   └── repositories/
    │   │       ├── user.repository.port.ts
    │   │       └── session.repository.port.ts
    │   ├── use-cases/
    │   │   ├── auth/
    │   │   │   ├── login.use-case.ts
    │   │   │   ├── logout.use-case.ts
    │   │   │   ├── refresh-token.use-case.ts
    │   │   │   └── validate-token.use-case.ts
    │   │   ├── registration/
    │   │   │   ├── check-email.use-case.ts
    │   │   │   ├── verify-otp.use-case.ts
    │   │   │   └── complete-registration.use-case.ts
    │   │   └── user/
    │   │       ├── get-profile.use-case.ts
    │   │       └── update-profile.use-case.ts
    │   └── mappers/
    │       └── user.mapper.ts
    │
    ├── infrastructure/
    │   ├── repositories/
    │   │   ├── prisma-user.repository.ts
    │   │   └── prisma-session.repository.ts
    │   └── services/
    │       ├── jwt.service.ts
    │       ├── bcrypt-password.service.ts
    │       ├── redis-otp.service.ts
    │       └── email-notification.service.ts
    │
    ├── interface/
    │   ├── controllers/
    │   │   ├── auth.controller.ts
    │   │   ├── registration.controller.ts
    │   │   ├── user.controller.ts
    │   │   └── health.controller.ts
    │   └── routes/
    │       ├── auth.routes.ts
    │       ├── registration.routes.ts
    │       ├── user.routes.ts
    │       └── health.routes.ts
    │
    └── shared/
        └── di/
            └── container.ts
```

### Events Published (Async)

| Event | Payload | Consumers |
|-------|---------|-----------|
| `UserRegistered` | `{ profileId, email, status }` | svc-kyc, svc-trust, svc-wallet |
| `UserStatusChanged` | `{ profileId, oldStatus, newStatus }` | svc-wallet, svc-business |
| `SessionCreated` | `{ profileId, sessionId, deviceId }` | Audit log |
| `SessionRevoked` | `{ profileId, sessionId, reason }` | Audit log |

### Implementation Steps

#### Step 1: Create Service Scaffold
```bash
mkdir -p apps/svc-auth/src/{domain/{entities,value-objects,errors},application/{dto/{request,response},ports/repositories,use-cases/{auth,registration,user},mappers},infrastructure/{repositories,services},interface/{controllers,routes},shared/di}

# Copy package.json and tsconfig from svc-tokenomics as template
cp apps/svc-tokenomics/package.json apps/svc-auth/
cp apps/svc-tokenomics/tsconfig.json apps/svc-auth/
```

#### Step 2: Create Core Files
- `config.ts` - Port 3041, JWT_SECRET, REDIS_URL
- `app.ts` - Express app with auth routes
- `index.ts` - Entry point

#### Step 3: Implement Clean Architecture Layers
1. **Domain:** User entity, Email/Password value objects
2. **Application:** Auth use cases (login, logout, refresh, validate)
3. **Infrastructure:** Prisma repositories, JWT service, bcrypt service
4. **Interface:** Thin controllers and routes

#### Step 4: Create Internal Validation Endpoint
```typescript
// GET /api/v1/auth/validate (internal use by other services)
// Request: Authorization: Bearer <token>
// Response: { valid: true, user: { profileId, email, status, roles } }
```

#### Step 5: Create Kubernetes Resources
- `svc-auth-deployment.yaml` - Deployment (port 3041)
- `svc-auth-service.yaml` - ClusterIP service
- Update ingress for auth routes

### Migration Strategy (Strangler Fig)

**Week 1: Shadow Mode**
1. Deploy svc-auth with read access to same database
2. Route 10% of auth traffic to new service
3. Compare responses with svc-identity
4. Log discrepancies without affecting users

**Week 2: Gradual Rollout**
1. Increase traffic to 50%, then 100%
2. Monitor error rates, latency, JWT validation
3. Ensure all downstream services work with new auth

**Week 3: Dual-Write Phase**
1. Both services write to database
2. Verify data consistency
3. Begin routing writes to svc-auth only

**Week 4: Cutover & Cleanup**
1. 100% traffic to svc-auth
2. Remove auth code from svc-identity
3. Update internal service references

### Rollback Strategy

**Immediate Rollback (< 5 minutes):**
```bash
# Feature flag disable
kubectl set env deployment/svc-auth USE_NEW_AUTH=false

# Or ingress switch
kubectl apply -f gx-devnet-ingress-auth-rollback.yaml
```

**Data Rollback:**
- Maintain dual-write for 2 weeks post-migration
- Event replay from Redis Streams if needed

### Critical Files to Create

**New Files (svc-auth):**
1. `apps/svc-auth/package.json`
2. `apps/svc-auth/tsconfig.json`
3. `apps/svc-auth/Dockerfile`
4. `apps/svc-auth/src/index.ts`
5. `apps/svc-auth/src/app.ts`
6. `apps/svc-auth/src/config.ts`
7. All domain, application, infrastructure, interface layers

**Infrastructure Files:**
1. `gx-infra-arch/k8s/devnet/svc-auth-deployment.yaml`
2. `gx-infra-arch/k8s/devnet/gx-devnet-ingress.yaml` (update)

**Files to Remove from svc-identity (after migration):**
1. `controllers/auth.controller.ts`
2. `controllers/registration.controller.ts`
3. `controllers/user-security.controller.ts`
4. `services/auth.service.ts`
5. `services/registration.service.ts`
6. `routes/auth.routes.ts`
7. `routes/registration.routes.ts`

### Verification Checklist

- [ ] New service builds successfully (`turbo build --filter=svc-auth`)
- [ ] Login/logout/refresh endpoints work correctly
- [ ] JWT validation endpoint returns correct user data
- [ ] Registration flow completes successfully
- [ ] OTP email/SMS delivery works
- [ ] Integration tests pass with Testcontainers
- [ ] Kubernetes deployment successful in backend-devnet
- [ ] Ingress routing works for new service
- [ ] All downstream services can validate JWTs via svc-auth
- [ ] Error rates stable for 48 hours after cutover
- [ ] Events published to Redis Streams correctly
- [ ] Cleanup: auth code removed from svc-identity

---

## Part 4: Shared Infrastructure Improvements

### 1. Create @gx/core-errors Package

```typescript
// Standard error classes
export class AppError extends Error {
  constructor(public code: string, public statusCode: number, message: string) {
    super(message);
  }
}

export class BadRequestError extends AppError { /* 400 */ }
export class UnauthorizedError extends AppError { /* 401 */ }
export class ForbiddenError extends AppError { /* 403 */ }
export class NotFoundError extends AppError { /* 404 */ }
export class ConflictError extends AppError { /* 409 */ }
export class ValidationError extends AppError { /* 422 */ }
```

### 2. Enhance @gx/core-config

```typescript
// Base schema all services share
export const baseServiceSchema = z.object({
  port: z.coerce.number().min(1000).max(65535),
  nodeEnv: z.enum(['development', 'production', 'test']).default('development'),
  databaseUrl: z.string().min(1),
  redisUrl: z.string().min(1),
  jwtSecret: z.string().min(32),
  logLevel: z.enum(['trace', 'debug', 'info', 'warn', 'error', 'fatal']).default('info'),
});

// Services extend this
const serviceSchema = baseServiceSchema.extend({
  customField: z.string().optional(),
});
```

### 3. Delete Duplicate Auth Middlewares

Remove from all services:
- `apps/svc-admin/src/middlewares/auth.middleware.ts`
- `apps/svc-governance/src/middlewares/auth.middleware.ts`
- `apps/svc-loanpool/src/middlewares/auth.middleware.ts`
- `apps/svc-organization/src/middlewares/auth.middleware.ts`
- `apps/svc-tokenomics/src/middlewares/auth.middleware.ts`
- `apps/svc-tax/src/middlewares/auth.middleware.ts`

All should use: `import { createAuthMiddleware } from '@gx/core-http';`

---

## Part 5: Enhanced Backend Skill

The complete enhanced backend skill is below. This replaces `.claude/commands/backend.md`:

```markdown
# Backend Development Skill

You are an expert backend developer for GX Protocol. This skill provides comprehensive patterns for enterprise-grade microservices following Clean Architecture principles.

---

## Architecture Overview

```
gx-protocol-backend/                    # Turbo monorepo (v1.13.4)
├── apps/                               # Microservices (13 services)
│   ├── svc-identity/                   # Auth, KYC, profiles
│   ├── svc-tokenomics/                 # Transfers, balances
│   ├── svc-government/                 # Government treasury
│   ├── svc-admin/                      # Admin portal
│   ├── svc-organization/               # Business management
│   └── ...                             # Other services
├── workers/                            # Background workers
│   ├── projector/                      # Blockchain event projection
│   └── outbox-submitter/               # CQRS command submission
├── packages/                           # Shared libraries (9 packages)
│   ├── core-http/                      # Express middleware, validation
│   ├── core-db/                        # Prisma ORM client
│   ├── core-config/                    # Zod env validation
│   ├── core-logger/                    # Pino structured logging
│   ├── core-events/                    # Event schema registry
│   ├── core-fabric/                    # Hyperledger Fabric SDK
│   ├── core-storage/                   # Document storage, OCR
│   ├── core-openapi/                   # OpenAPI validation
│   └── core-graph/                     # Neo4j graph database
└── db/prisma/schema.prisma             # Database schema (125+ models)
```

---

## Internal Service Structure (Clean Architecture)

Every service MUST follow this structure:

```
apps/svc-{name}/src/
├── domain/                    # Core Business Logic (NO dependencies)
│   ├── entities/             # Domain entities with behavior
│   ├── value-objects/        # Immutable domain concepts
│   ├── services/             # Domain services (pure logic)
│   ├── events/               # Domain events
│   └── errors/               # Domain-specific errors
│
├── application/              # Application Layer (use cases)
│   ├── use-cases/           # Business use cases (feature-grouped)
│   │   ├── auth/            # e.g., login, register, refresh
│   │   └── user/            # e.g., get-profile, update-profile
│   ├── ports/               # Interfaces (contracts)
│   │   ├── repositories/    # Repository interfaces
│   │   └── services/        # External service interfaces
│   ├── dto/                 # DTOs with Zod validation
│   │   ├── request/         # Request DTOs
│   │   └── response/        # Response DTOs
│   └── mappers/             # Entity <-> DTO mappers
│
├── infrastructure/          # External Adapters (implementations)
│   ├── repositories/        # Prisma repository implementations
│   └── services/            # External service implementations
│
├── interface/               # HTTP Layer
│   ├── controllers/         # THIN controllers
│   ├── routes/              # Express routes
│   ├── middlewares/         # HTTP middlewares
│   └── validators/          # Zod schemas
│
├── shared/                  # Cross-cutting
│   └── di/                  # Dependency injection
│
├── app.ts                   # Express app factory
└── index.ts                 # Entry point
```

---

## Core Patterns

### 1. CQRS with Transactional Outbox

```
WRITE PATH:
  API → Use Case → Repository → db.$transaction([data, outboxCommand]) → 202 Accepted
  Background: outbox-submitter → Fabric Chaincode → Blockchain

READ PATH:
  Fabric Events → projector → PostgreSQL read models → API Queries
```

### 2. Multi-Tenancy

ALWAYS filter by tenantId:
```typescript
await db.userProfile.findMany({
  where: { tenantId: req.user!.tenantId, status: 'ACTIVE' },
});
```

### 3. Repository Pattern

```typescript
// Port (interface)
export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  create(user: User): Promise<User>;
}

// Adapter (implementation)
export class PrismaUserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    const record = await db.userProfile.findUnique({ where: { profileId: id } });
    return record ? UserMapper.toDomain(record) : null;
  }
}
```

### 4. Use Case Pattern

```typescript
export class RegisterUseCase {
  constructor(
    private userRepo: IUserRepository,
    private hasher: IPasswordHasher,
  ) {}

  async execute(dto: RegisterRequestDTO): Promise<AuthResponseDTO> {
    // 1. Validate business rules
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) throw new UserAlreadyExistsError();

    // 2. Create domain entity
    const user = User.create({ ...dto, passwordHash: await this.hasher.hash(dto.password) });

    // 3. Persist
    const saved = await this.userRepo.create(user);

    // 4. Return DTO
    return UserMapper.toResponseDTO(saved);
  }
}
```

### 5. DTO Validation with Zod

```typescript
// application/dto/request/register.dto.ts
export const registerSchema = z.object({
  email: z.string().email().transform(v => v.toLowerCase()),
  password: z.string().min(8).regex(/[A-Z]/).regex(/[a-z]/).regex(/[0-9]/),
  firstName: z.string().min(1).max(100),
  lastName: z.string().min(1).max(100),
});

export type RegisterRequestDTO = z.infer<typeof registerSchema>;
```

### 6. Thin Controllers

```typescript
// Controllers only handle HTTP concerns
export class AuthController {
  constructor(private registerUseCase: RegisterUseCase) {}

  register = async (req: Request, res: Response, next: NextFunction) => {
    try {
      // Validation done by middleware, logic in use case
      const result = await this.registerUseCase.execute(req.body);
      res.status(201).json({ success: true, data: result });
    } catch (error) {
      next(error);
    }
  };
}
```

### 7. Validation Middleware

```typescript
// interface/middlewares/validation.middleware.ts
export function validateBody<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body);
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(400).json({
          success: false,
          error: {
            code: 'VALIDATION_ERROR',
            message: 'Validation failed',
            details: error.errors.map(e => ({ field: e.path.join('.'), message: e.message })),
          },
        });
      }
      next(error);
    }
  };
}
```

### 8. Entity Mapper

```typescript
export class UserMapper {
  // Prisma → Domain
  static toDomain(record: PrismaUserProfile): User {
    return User.reconstitute({ id: record.profileId, email: Email.create(record.email), ... });
  }

  // Domain → Prisma
  static toPersistence(entity: User): PrismaData {
    return { profileId: entity.id, email: entity.email.value, ... };
  }

  // Domain → Response DTO
  static toDTO(entity: User): UserProfileDTO {
    return { profileId: entity.id, email: entity.email.value, ... };
  }
}
```

---

## Creating a New Feature (Step by Step)

### Step 1: Define Domain Entity (if new)

```typescript
// domain/entities/feature.entity.ts
export class Feature {
  private constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly tenantId: string,
    public readonly createdAt: Date,
  ) {}

  static create(props: { name: string; tenantId: string }): Feature {
    return new Feature(generateId(), props.name, props.tenantId, new Date());
  }

  static reconstitute(props: FeatureProps): Feature {
    return new Feature(props.id, props.name, props.tenantId, props.createdAt);
  }
}
```

### Step 2: Define Repository Port

```typescript
// application/ports/repositories/feature.repository.port.ts
export interface IFeatureRepository {
  findById(id: string, tenantId: string): Promise<Feature | null>;
  findAll(tenantId: string, options: PaginationOptions): Promise<Feature[]>;
  create(feature: Feature): Promise<Feature>;
}
export const FEATURE_REPOSITORY = Symbol('FEATURE_REPOSITORY');
```

### Step 3: Define DTOs with Zod

```typescript
// application/dto/request/create-feature.dto.ts
export const createFeatureSchema = z.object({
  name: z.string().min(2).max(100),
  description: z.string().max(500).optional(),
});
export type CreateFeatureDTO = z.infer<typeof createFeatureSchema>;

// application/dto/response/feature.dto.ts
export interface FeatureResponseDTO {
  id: string;
  name: string;
  createdAt: string;
}
```

### Step 4: Create Use Case

```typescript
// application/use-cases/feature/create-feature.use-case.ts
@injectable()
export class CreateFeatureUseCase {
  constructor(@inject(FEATURE_REPOSITORY) private featureRepo: IFeatureRepository) {}

  async execute(dto: CreateFeatureDTO, tenantId: string): Promise<FeatureResponseDTO> {
    const feature = Feature.create({ name: dto.name, tenantId });
    const saved = await this.featureRepo.create(feature);
    return FeatureMapper.toDTO(saved);
  }
}
```

### Step 5: Implement Repository

```typescript
// infrastructure/repositories/prisma-feature.repository.ts
@injectable()
export class PrismaFeatureRepository implements IFeatureRepository {
  async create(feature: Feature): Promise<Feature> {
    const record = await db.feature.create({
      data: FeatureMapper.toPersistence(feature),
    });
    return FeatureMapper.toDomain(record);
  }
}
```

### Step 6: Create Controller (THIN)

```typescript
// interface/controllers/feature.controller.ts
@injectable()
export class FeatureController {
  constructor(@inject(CreateFeatureUseCase) private createUseCase: CreateFeatureUseCase) {}

  create = async (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    try {
      const result = await this.createUseCase.execute(req.body, req.user!.tenantId);
      res.status(201).json({ success: true, data: result });
    } catch (error) {
      next(error);
    }
  };
}
```

### Step 7: Create Routes

```typescript
// interface/routes/feature.routes.ts
const router = Router();
const controller = container.get(FeatureController);

router.post('/',
  createAuthMiddleware({ jwtSecret: config.jwtSecret }),
  validateBody(createFeatureSchema),
  controller.create
);

export default router;
```

---

## Authentication & Authorization

```typescript
import { createAuthMiddleware, requireRoles, AuthenticatedRequest } from '@gx/core-http';

const auth = createAuthMiddleware({ jwtSecret: config.jwtSecret });

// In routes
router.get('/protected', auth, controller.method);
router.post('/admin-only', auth, requireRoles(['ADMIN']), controller.adminMethod);

// In controller
async handler(req: AuthenticatedRequest, res: Response) {
  const userId = req.user!.sub;
  const tenantId = req.user!.tenantId;
  const roles = req.user!.roles;
}
```

---

## Error Handling

### Standard Response Format

```typescript
// Success
{ success: true, data: T, meta?: { page, limit, total } }

// Error
{ success: false, error: { code: string, message: string, timestamp: string, details?: [] } }
```

### Error Codes

| Code | Status | Usage |
|------|--------|-------|
| VALIDATION_ERROR | 400 | Invalid input |
| UNAUTHORIZED | 401 | Missing/invalid token |
| FORBIDDEN | 403 | Insufficient permissions |
| NOT_FOUND | 404 | Resource not found |
| CONFLICT | 409 | Duplicate resource |
| INSUFFICIENT_BALANCE | 400 | Not enough funds |

### Throwing Errors

```typescript
// In use cases
if (!user) throw new NotFoundError('User', userId);
if (balance < amount) throw new InsufficientBalanceError(balance, amount);
```

---

## Database Operations

```typescript
import { db } from '@gx/core-db';

// ALWAYS use tenantId
const users = await db.userProfile.findMany({
  where: { tenantId, status: 'ACTIVE' },
  skip: (page - 1) * limit,
  take: limit,
});

// ALWAYS use transactions for multi-table ops
await db.$transaction(async (tx) => {
  const user = await tx.userProfile.update({ ... });
  const wallet = await tx.wallet.create({ ... });
  await tx.outboxCommand.create({ type: 'CREATE_WALLET', ... });
  return { user, wallet };
});
```

---

## CQRS Outbox Pattern

For blockchain operations, use 202 Accepted:

```typescript
const result = await db.$transaction(async (tx) => {
  // Update local state
  const wallet = await tx.wallet.update({ ... });

  // Queue blockchain command
  const command = await tx.outboxCommand.create({
    data: {
      type: CommandType.TRANSFER_TOKENS,
      payload: { fromWalletId, toWalletId, amount: amount.toString() },
      status: OutboxStatus.PENDING,
      tenantId,
      createdBy: userId,
    },
  });

  return { commandId: command.id };
});

return res.status(202).json({
  success: true,
  data: { commandId: result.commandId, status: 'PENDING' },
});
```

---

## Logging

```typescript
import { logger } from '@gx/core-logger';

// Structured logging with context
logger.info({ userId, tenantId, action: 'login' }, 'User logged in');
logger.error({ error, userId }, 'Transfer failed');

// NEVER log sensitive data (passwords, tokens, etc.)
```

---

## Validation Checklist

Before completing ANY backend task:

**Architecture:**
- [ ] Domain logic in domain/ layer (no I/O)
- [ ] Use cases in application/ layer
- [ ] Repositories implement interfaces from ports/
- [ ] Controllers are THIN (delegate to use cases)

**Code Quality:**
- [ ] DTOs validated with Zod
- [ ] Entity mappers for Prisma ↔ Domain ↔ DTO
- [ ] Structured logging at key points
- [ ] No sensitive data in logs

**Security:**
- [ ] Auth middleware from @gx/core-http
- [ ] tenantId filtering on ALL queries
- [ ] Input sanitization

**Data Integrity:**
- [ ] Transactions for multi-table operations
- [ ] Outbox pattern for blockchain operations
- [ ] Proper status codes (201 create, 202 async)

---

## Common Mistakes to Avoid

1. **Fat Controllers** - Move logic to use cases
2. **Missing tenantId** - ALWAYS filter by tenantId
3. **Direct Prisma in Controllers** - Use repositories
4. **Validation in Controllers** - Use Zod middleware
5. **Duplicate Auth Middleware** - Import from @gx/core-http
6. **200 for Async Ops** - Use 202 Accepted
7. **Logging Secrets** - Never log passwords/tokens

---

## Quick Reference

```typescript
// Imports
import { db } from '@gx/core-db';
import { logger } from '@gx/core-logger';
import { createAuthMiddleware, requireRoles, AuthenticatedRequest } from '@gx/core-http';
import { CommandType, OutboxStatus } from '@prisma/client';

// Common models
db.userProfile, db.wallet, db.transaction, db.outboxCommand,
db.kycApplication, db.businessAccount, db.governmentTreasury
```

---

## Apply to: $ARGUMENTS
```

---

## Part 6: Implementation Phases

### Phase 1: Immediate (Week 1-2)
1. Update `.claude/commands/backend.md` with enhanced skill
2. Create `@gx/core-errors` package
3. Add base config schema to `@gx/core-config`
4. Create Zod validation middleware in `@gx/core-http`

### Phase 2: Pilot Migration (Week 3-4)
1. Refactor `svc-tokenomics` to Clean Architecture pattern
2. Add repository layer
3. Extract use cases from services
4. Validate pattern works

### Phase 3: Rollout (Week 5-8)
1. Apply pattern to remaining services
2. Delete duplicate auth middlewares
3. Standardize type organization

### Phase 4: Service Decomposition (Week 9-24)
1. Extract bounded contexts from svc-identity
2. Implement API gateway routing
3. Ensure backward compatibility

---

## Verification Criteria

After implementation:
1. All services follow Clean Architecture structure
2. Repository pattern implemented (no direct Prisma in controllers/use cases)
3. Use cases handle all business logic
4. Controllers are thin (< 20 lines per method)
5. Zod validates all request DTOs
6. Shared packages used (no duplicate code)

---

## Critical Files

1. **`.claude/commands/backend.md`** - Enhanced skill (Part 5)
2. **`packages/core-errors/src/`** - New error package
3. **`packages/core-http/src/middlewares/validation.middleware.ts`** - Zod middleware
4. **`apps/svc-tokenomics/`** - Pilot service for pattern
5. **`packages/test-utils/`** - New shared testing utilities package
6. **`vitest.workspace.ts`** - Vitest workspace configuration

---

## Part 7: Testing Architecture (Testing Trophy Pattern)

### Why Testing Pyramid is Dead for Microservices

The traditional pyramid (70% unit, 20% integration, 10% E2E) was designed for monoliths. In microservices, the most significant complexity is in **service interactions**, not individual functions.

### The Testing Trophy Distribution

```
                    ┌─────────┐
                    │   E2E   │  5-10% - Critical User Journeys ONLY
                    ├─────────┤
                    │ Integra-│  65-75% - THE FAT MIDDLE
                    │  tion   │  Real DB, Real Redis, Real Services
                    ├─────────┤
                    │Contract │  10-15% - Pact Consumer-Driven
                    ├─────────┤
                    │  Unit   │  10-15% - Pure Functions ONLY
                    └─────────┘
                    │ Static  │  FREE - TypeScript, ESLint
                    └─────────┘
```

### Test Types by Layer

| Layer | % | Tool | What to Test |
|-------|---|------|--------------|
| Static | FREE | TypeScript, ESLint | Types, linting |
| Unit | 10-15% | Vitest | Fee calculators, pure algorithms |
| Contract | 10-15% | Pact | API compatibility between services |
| Integration | 65-75% | Vitest + Testcontainers | Real DB, transactions, outbox |
| E2E | 5-10% | Playwright | Registration → Transfer journey |

---

### 1. Tool Selection (2025-2026 Standards)

**Vitest over Jest:**
- 10-20x faster in watch mode
- Native ESM and TypeScript support
- 95% Jest-compatible API

**Playwright over Cypress:**
- Multi-browser (Chromium, Firefox, WebKit)
- Native parallel execution
- Better for enterprise compliance testing

**Testcontainers for Integration:**
- Real PostgreSQL/Redis containers
- No more mocking databases
- Catches SQL errors, race conditions

**Pact for Contracts:**
- Consumer-driven contracts
- Breaks Provider CI if contract violated
- Prevents breaking API changes

---

### 2. Unit Tests (10-15%) - Pure Functions ONLY

**DO Test:**
```typescript
// Pure business logic - perfect for unit testing
export function calculateTransferFee(
  amount: number,
  txType: 'P2P' | 'B2B' | 'GOVT',
  isLocal: boolean
): { feeAmount: number; feePercentage: number } {
  if (amount <= 3) return { feeAmount: 0, feePercentage: 0 };

  let baseFee = amount <= 100 ? 0.001 : 0.0005;
  const typeMultiplier = { P2P: 1.0, B2B: 0.8, GOVT: 0.0 }[txType];
  const crossBorderMultiplier = isLocal ? 1.0 : 1.5;

  const feePercentage = baseFee * typeMultiplier * crossBorderMultiplier;
  return { feeAmount: amount * feePercentage, feePercentage };
}
```

```typescript
// Unit test - fast, no I/O
describe('calculateTransferFee', () => {
  it('should be free for micro-transactions <= 3', () => {
    expect(calculateTransferFee(3, 'P2P', true)).toEqual({ feeAmount: 0, feePercentage: 0 });
  });

  it('should apply 20% discount for B2B', () => {
    const result = calculateTransferFee(100, 'B2B', true);
    expect(result.feePercentage).toBe(0.0008);
  });

  it('should be free for government transactions', () => {
    expect(calculateTransferFee(1000000, 'GOVT', false).feeAmount).toBe(0);
  });
});
```

**DO NOT Test** (controllers, services that just call DB):
```typescript
// BAD - testing implementation, not behavior
describe('UserController', () => {
  it('should call service method', () => {
    expect(mockService).toHaveBeenCalled(); // Testing HOW, not WHAT
  });
});
```

---

### 3. Integration Tests (65-75%) - THE FAT MIDDLE

**Testcontainers Setup:**
```typescript
// packages/test-utils/src/database-container.ts
import { PostgreSqlContainer } from '@testcontainers/postgresql';
import { PrismaClient } from '@prisma/client';
import { execSync } from 'child_process';

let container: StartedPostgreSqlContainer;
let prisma: PrismaClient;

export async function setupTestDatabase(): Promise<PrismaClient> {
  container = await new PostgreSqlContainer('postgres:15-alpine')
    .withDatabase('gx_test')
    .start();

  const databaseUrl = container.getConnectionUri();

  // Apply migrations
  execSync(`DATABASE_URL="${databaseUrl}" npx prisma migrate deploy`);

  prisma = new PrismaClient({ datasources: { db: { url: databaseUrl } } });
  return prisma;
}

export async function teardownTestDatabase(): Promise<void> {
  await prisma?.$disconnect();
  await container?.stop();
}

export async function resetDatabase(): Promise<void> {
  await prisma.$executeRaw`TRUNCATE TABLE "OutboxCommand" CASCADE`;
  await prisma.$executeRaw`TRUNCATE TABLE "UserProfile" CASCADE`;
}
```

**Integration Test Example:**
```typescript
// Real database, real transactions, real behavior
describe('Organization Onboarding (Real DB)', () => {
  let prisma: PrismaClient;

  beforeAll(async () => {
    prisma = await setupTestDatabase();
  }, 60000);

  afterAll(async () => {
    await teardownTestDatabase();
  });

  beforeEach(async () => {
    await resetDatabase();
  });

  it('should create organization with outbox command atomically', async () => {
    const proposer = await prisma.userProfile.create({
      data: { profileId: 'proposer_001', email: 'proposer@acme.com', ... },
    });

    const result = await organizationService.createOrganization({
      proposerId: proposer.profileId,
      name: 'Acme Corp',
    });

    // Verify BOTH exist (atomic transaction)
    const org = await prisma.organizationV2.findUnique({ where: { orgId: result.orgId } });
    const command = await prisma.outboxCommand.findFirst({ where: { commandType: 'PROPOSE_ORGANIZATION' } });

    expect(org).not.toBeNull();
    expect(command).not.toBeNull();
    expect(command?.payload).toHaveProperty('orgId', result.orgId);
  });

  it('should rollback both on error', async () => {
    await expect(
      prisma.$transaction(async (tx) => {
        await tx.userProfile.create({ data: { profileId: 'user_002', ... } });
        throw new Error('Simulated failure');
      })
    ).rejects.toThrow();

    // Verify user was NOT created
    const user = await prisma.userProfile.findUnique({ where: { profileId: 'user_002' } });
    expect(user).toBeNull();
  });
});
```

---

### 4. Contract Tests (10-15%) - Microservice Glue

**Consumer Test (svc-tokenomics consuming svc-identity):**
```typescript
// apps/svc-tokenomics/src/__tests__/contracts/identity-consumer.pact.ts
import { PactV4, MatchersV3 } from '@pact-foundation/pact';

const provider = new PactV4({
  consumer: 'svc-tokenomics',
  provider: 'svc-identity',
});

describe('svc-tokenomics -> svc-identity Contract', () => {
  it('should return user profile with fabricUserId', async () => {
    await provider
      .addInteraction()
      .given('user GX_USER_001 exists')
      .uponReceiving('a request for user profile')
      .withRequest({
        method: 'GET',
        path: '/api/v1/users/GX_USER_001',
        headers: { 'Authorization': like('Bearer token123') },
      })
      .willRespondWith({
        status: 200,
        body: {
          profileId: like('profile_001'),
          fabricUserId: regex(/^GX_[A-Z0-9_]+$/, 'GX_USER_001'),
          status: like('ACTIVE'),
        },
      })
      .executeTest(async (mockServer) => {
        const response = await fetch(`${mockServer.url}/api/v1/users/GX_USER_001`);
        expect(response.status).toBe(200);
      });
  });
});
```

**Provider Verification (svc-identity):**
```typescript
// apps/svc-identity/src/__tests__/contracts/identity-provider.pact.ts
describe('svc-identity Provider Verification', () => {
  it('should fulfill all consumer contracts', async () => {
    const verifier = new Verifier({
      providerBaseUrl: `http://localhost:${port}`,
      provider: 'svc-identity',
      pactBrokerUrl: process.env.PACT_BROKER_URL,
      stateHandlers: {
        'user GX_USER_001 exists': async () => {
          await prisma.userProfile.upsert({
            where: { fabricUserId: 'GX_USER_001' },
            create: { profileId: 'profile_001', fabricUserId: 'GX_USER_001', ... },
            update: {},
          });
        },
      },
    });

    await verifier.verifyProvider();
  });
});
```

**Breaking Change Prevention:**
```
Consumer: svc-tokenomics expects { fabricUserId: string }
Provider: svc-identity PR #123 renames to blockchainUserId

Pact Broker: can-i-deploy → FAIL
PR #123 BLOCKED until contract is updated
```

---

### 5. E2E Tests (5-10%) - Critical User Journeys ONLY

**Playwright Configuration:**
```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e/critical-journeys',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  use: {
    baseURL: process.env.STAGING_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
    { name: 'webkit', use: { browserName: 'webkit' } },
  ],
});
```

**Critical User Journeys (CUJs) ONLY:**
```typescript
// e2e/critical-journeys/registration-to-transfer.spec.ts
describe('CUJ-1: User Registration to First Transfer', () => {
  test('complete user journey', async ({ page }) => {
    // Registration
    await page.goto('/register');
    await page.fill('[name=email]', 'newuser@test.com');
    await page.click('button[type=submit]');

    // KYC
    await expect(page).toHaveURL('/kyc');
    await page.setInputFiles('input[type=file]', 'fixtures/id-front.jpg');

    // First transfer
    await page.goto('/wallet/send');
    await page.fill('[name=amount]', '10');
    await page.click('button[type=submit]');

    await expect(page.locator('.success-message')).toBeVisible();
  });
});

// ONLY 4-5 critical journeys:
// CUJ-1: Registration → First Transfer
// CUJ-2: P2P Transfer with Fee Verification
// CUJ-3: Organization Multi-Sig Approval
// CUJ-4: Government Treasury Disbursement
```

---

### 6. Test Directory Structure

```
gx-protocol-backend/
├── packages/
│   └── test-utils/                     # Shared test utilities
│       ├── src/
│       │   ├── database-container.ts   # PostgreSQL container
│       │   ├── redis-container.ts      # Redis container
│       │   ├── fabric-mock.ts          # Fabric blockchain mock
│       │   └── factories/
│       │       ├── user.factory.ts
│       │       └── transaction.factory.ts
│       └── package.json
├── apps/
│   └── svc-{name}/
│       └── src/
│           └── __tests__/
│               ├── unit/               # Pure function tests
│               ├── integration/        # Testcontainers tests
│               └── contracts/          # Pact tests
├── e2e/
│   ├── playwright.config.ts
│   └── critical-journeys/              # E2E - CUJs only
└── vitest.workspace.ts                 # Workspace config
```

---

### 7. Vitest Workspace Configuration

```typescript
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config';

export default defineWorkspace([
  {
    test: {
      name: 'unit',
      include: ['**/*.unit.spec.ts'],
      environment: 'node',
    },
  },
  {
    test: {
      name: 'integration',
      include: ['**/*.integration.spec.ts'],
      environment: 'node',
      testTimeout: 60000,
      globalSetup: './packages/test-utils/src/global-setup.ts',
    },
  },
]);
```

---

### 8. CI/CD Pipeline for Tests

```yaml
# .github/workflows/test.yml
jobs:
  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint
      - run: npm run type-check

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:unit -- --coverage

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
      redis:
        image: redis:7-alpine
    steps:
      - run: npm run test:integration

  contract-tests:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:contracts
      - run: npx pact-broker publish pacts/

  e2e-tests:
    needs: [unit-tests, integration-tests]
    if: github.ref == 'refs/heads/main'
    steps:
      - run: npx playwright install
      - run: npm run test:e2e
```

---

### 9. Testing Checklist

Before completing ANY feature:

- [ ] **Unit tests** for pure business logic (fee calculations, validators)
- [ ] **Integration tests** with real Postgres (Testcontainers)
- [ ] **Contract tests** if consuming another service's API
- [ ] **No mocking databases** - use Testcontainers
- [ ] **E2E only** for critical user journeys (don't over-test)

---

## Part 8: Enhanced Backend Skill (Testing Section)

Add to the backend skill in `.claude/commands/backend.md`:

```markdown
---

## Testing Patterns (Testing Trophy)

### Test Distribution
| Layer | % | Tool | Test |
|-------|---|------|------|
| Unit | 10-15% | Vitest | Pure functions only |
| Integration | 65-75% | Vitest + Testcontainers | Real DB |
| Contract | 10-15% | Pact | API compatibility |
| E2E | 5-10% | Playwright | CUJs only |

### Unit Test (Pure Functions)
```typescript
// Only test pure business logic
describe('calculateFee', () => {
  it('should be free for amounts <= 3', () => {
    expect(calculateFee(3)).toEqual({ amount: 0 });
  });
});
```

### Integration Test (Real DB)
```typescript
// Use Testcontainers - NO MOCKS
describe('UserService (Real DB)', () => {
  let prisma: PrismaClient;

  beforeAll(async () => {
    prisma = await setupTestDatabase();
  }, 60000);

  it('should create user in real database', async () => {
    const user = await userService.create({ email: 'test@test.com' });
    const found = await prisma.userProfile.findUnique({ where: { id: user.id } });
    expect(found).not.toBeNull();
  });
});
```

### Contract Test (API Compatibility)
```typescript
// Consumer defines expectations, Provider must fulfill
await provider
  .given('user exists')
  .uponReceiving('get user request')
  .withRequest({ method: 'GET', path: '/api/v1/users/123' })
  .willRespondWith({ status: 200, body: { id: like('123') } })
  .executeTest(async (mockServer) => { ... });
```

### What NOT to Test
- Controllers (just call use cases)
- Repositories (just call Prisma)
- Anything that requires mocking the database

---
```

---

## Updated Implementation Phases

### Phase 1: Immediate (Week 1-2)
1. Update `.claude/commands/backend.md` with enhanced skill + testing
2. Create `@gx/core-errors` package
3. Create `packages/test-utils/` with Testcontainers setup
4. Add Vitest workspace configuration

### Phase 2: Pilot Migration (Week 3-4)
1. Migrate `svc-tokenomics` to Clean Architecture
2. Add integration tests with Testcontainers
3. Add contract tests with Pact
4. Delete all database mocks

### Phase 3: Rollout (Week 5-8)
1. Apply patterns to remaining services
2. Set up Pact Broker for contract testing
3. Create E2E critical journey tests with Playwright

### Phase 4: Service Decomposition (Week 9-24)
1. Extract bounded contexts from svc-identity
2. Add contract tests for new service boundaries
3. Ensure backward compatibility

---

*Plan created: 2026-01-19*
*Ready for approval*
