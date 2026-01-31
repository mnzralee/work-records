# Prisma Schema Refactoring Plan - GX Protocol

## Executive Summary

**Recommendation: Option A - Multi-file Prisma Schema (prismaSchemaFolder)**

This is the right choice for GX Protocol right now because:
- **Lowest risk**: No database changes, no migration impact, purely organizational
- **Immediate maintainability win**: 149 models in one file is unmanageable
- **Prisma 6.17.1 supports it**: Native feature (GA since 6.7.0), no preview flag needed
- **Aligns with DDD/Hexagonal Architecture**: Files map to bounded contexts and aggregate roots
- **Sets foundation for future**: Clear domain boundaries without breaking changes
- **Single client**: No multi-client complexity—all services continue using `@gx/core-db`

### DDD Alignment
The multi-file approach maps directly to your Hexagonal Architecture:
- Each `.prisma` file = **Bounded Context** or **Aggregate Root**
- Cross-context references use **ID-only** (no FK relations) = respects context boundaries
- Enums are **Value Objects** owned by their context or shared explicitly

---

## Part 1: Current State Assessment

### Schema Statistics
| Metric | Value |
|--------|-------|
| Total Models | 149 |
| Total Enums | 98 |
| Total Relations | 199 |
| Schema File Size | 223 KB |
| Migrations | 14 |
| Max FK references (single model) | 48 (UserProfile) |

### Current Structure
```
db/prisma/
├── schema.prisma     (5,000+ lines, all models + enums)
├── migrations/       (14 migrations)
│   └── migration_lock.toml
```

### High Coupling Risks Identified
1. **UserProfile**: Referenced by 48 models (central coupling point)
2. **Wallet**: 12+ models reference `walletId`
3. **OrganizationProfile**: Polymorphic base for Business/Government/NPO accounts
4. **Shared Enums**: `SignatoryRole`, `ApprovalStatus` used across 7+ models

### Services & Bounded Contexts (14 services)
| Service | Primary Domain | Key Models |
|---------|---------------|------------|
| svc-identity | Identity & Auth | UserProfile, PendingRegistration, Address |
| svc-tokenomics | Token Operations | Wallet, Transaction, OutboxCommand |
| svc-government | Treasury | GovernmentTreasury, GovernmentAccount, GovernmentSignatory |
| svc-admin | Administration | AdminUser, Permission, RolePermission, AdminAuditLog |
| svc-wallet | Wallet Features | SubAccount, AllocationRule, Contact, TransactionCategory |
| svc-kyc | KYC Processing | KYCApplication, KYCDocument, AttorneyProfile |
| svc-organization | Org Management | OrganizationProfile, BusinessAccount, BusinessSignatory |
| svc-trust | Trust Scores | TrustScore, FinancialBehaviorScore, FamilyRelationship |
| svc-messaging | Messaging | Conversation, Message, UserSignalKey |
| svc-auth | Authentication | UserSession, TrustedDevice, UserSecuritySettings |

---

## Part 2: Options Analysis (ADR)

### Option A: Multi-file Prisma Schema ✅ RECOMMENDED
Split the single `schema.prisma` into domain-organized `.prisma` files in a `schema/` folder.

**Pros:**
- Zero migration risk—purely file reorganization
- Improves code review (changes isolated to domain files)
- Clear ownership boundaries for teams
- Native Prisma support (6.17.1)
- Single generated client (no breaking changes to services)
- Easy rollback (just restore single file)

**Cons:**
- Cross-file relations require explicit imports (handled by Prisma)
- Enums shared across domains need careful placement
- IDE tooling varies (VSCode Prisma extension handles it well)

**Fit for GX:** Excellent. Addresses maintainability without operational risk.

---

### Option B: Postgres Schema-per-Service (Multi-Schema)
Use PostgreSQL schemas (e.g., `identity.users`, `wallet.transactions`) with Prisma's `@@schema()` directive.

**Pros:**
- True data isolation at DB level
- Can add RLS per schema
- Sets foundation for db-per-service

**Cons:**
- **HIGH RISK**: Requires data migration for existing tables
- Cross-schema queries are complex
- Foreign keys across schemas are tricky
- Prisma multi-schema is still preview feature

**Fit for GX:** Not now. Too risky for Phase 2. Consider for Phase 3/4.

---

### Option C: Multiple Prisma Schemas + Multiple Clients
Each service has its own `schema.prisma` and generates its own client.

**Pros:**
- True separation of concerns
- Each service owns its data completely
- Natural path to db-per-service

**Cons:**
- **BREAKING CHANGE**: Requires massive refactoring
- Cross-service relations become impossible
- Data consistency challenges
- Current shared models (UserProfile) would need duplication or external refs

**Fit for GX:** Not appropriate. Premature given shared database reality.

---

### Option D: Single File with Strict Ordering
Keep one file but enforce strict section ordering and naming conventions.

**Pros:**
- No changes required
- Simple to understand

**Cons:**
- 5,000+ lines is unmanageable
- No tooling enforcement
- Merge conflicts in PRs
- Doesn't scale

**Fit for GX:** Fallback only if multi-file fails.

---

## Part 3: Implementation Plan

### Target Structure (DDD Bounded Contexts)

**CRITICAL**: Files must be at same level as `migrations/` folder (no nested schema/ subfolder).

```
db/prisma/
├── migrations/                  # UNCHANGED - stays at this level
│   └── migration_lock.toml
├── 00-config.prisma             # Generator + Datasource only
├── 01-enums.prisma              # All 98 enums (shared value objects)
│
│ # ══════════════════════════════════════════════════════════════════
│ # BOUNDED CONTEXT: Identity (Aggregate Root: UserProfile)
│ # ══════════════════════════════════════════════════════════════════
├── 10-identity.prisma           # UserProfile, PendingRegistration, Address
├── 11-identity-session.prisma   # UserSession, TrustedDevice, UserPresence
├── 12-identity-security.prisma  # UserSecuritySettings, UserDevice
│
│ # ══════════════════════════════════════════════════════════════════
│ # BOUNDED CONTEXT: Wallet (Aggregate Root: Wallet)
│ # ══════════════════════════════════════════════════════════════════
├── 20-wallet.prisma             # Wallet (aggregate root)
├── 21-wallet-transaction.prisma # Transaction, TransactionApproval, TransactionReceipt
├── 22-wallet-subaccount.prisma  # SubAccount, SubAccountTransaction, AllocationRule
├── 23-wallet-budget.prisma      # BudgetPeriod, AllocationExecution
├── 24-wallet-contact.prisma     # Contact, ContactGroup, ContactGroupMember
│
│ # ══════════════════════════════════════════════════════════════════
│ # BOUNDED CONTEXT: KYC & Compliance (Aggregate Root: KYCApplication)
│ # ══════════════════════════════════════════════════════════════════
├── 30-kyc.prisma                # KYCApplication, KYCDocument, KYCVerification
├── 31-kyc-attorney.prisma       # AttorneyProfile, AttorneyVerification
├── 32-kyc-sanctions.prisma      # SanctionsScreening, OFACSdnEntry
│
│ # ══════════════════════════════════════════════════════════════════
│ # BOUNDED CONTEXT: Organization (Aggregate Root: OrganizationProfile)
│ # ══════════════════════════════════════════════════════════════════
├── 40-organization.prisma       # OrganizationProfile, OrganizationStakeholder
├── 41-org-business.prisma       # BusinessAccount, BusinessSignatory, BusinessSignatoryRule
├── 42-org-business-sub.prisma   # BusinessSubAccount, BusinessSubAccountTx, BusinessBudgetPeriod
├── 43-org-api.prisma            # OrganizationApiCredential, OrganizationDocument
│
│ # ══════════════════════════════════════════════════════════════════
│ # BOUNDED CONTEXT: Government Treasury (Aggregate Root: GovernmentTreasury)
│ # ══════════════════════════════════════════════════════════════════
├── 50-government.prisma         # GovernmentTreasury, GovernmentHierarchyAccount
├── 51-govt-account.prisma       # GovernmentAccount, GovernmentSubAccount
├── 52-govt-signatory.prisma     # GovernmentSignatory, GovernmentSignatoryRule
├── 53-govt-approval.prisma      # GovernmentApproval, GovernmentApprovalVote
├── 54-govt-transaction.prisma   # GovernmentTransaction, GovernmentAPICredential
│
│ # ══════════════════════════════════════════════════════════════════
│ # BOUNDED CONTEXT: NPO (Aggregate Root: NotForProfitAccount)
│ # ══════════════════════════════════════════════════════════════════
├── 60-npo.prisma                # NotForProfitAccount, NPOSignatory, NPOSignatoryRule
├── 61-npo-program.prisma        # NPOProgram, ProgramBeneficiary, BeneficiaryDisbursement
├── 62-npo-funding.prisma        # NPODonation, NPOGrant, NPOGrantDisbursement
├── 63-npo-api.prisma            # NPOApiCredential, NPOWebhook, NPOWebhookDelivery
│
│ # ══════════════════════════════════════════════════════════════════
│ # BOUNDED CONTEXT: Admin & RBAC (Aggregate Root: AdminUser)
│ # ══════════════════════════════════════════════════════════════════
├── 70-admin.prisma              # AdminUser, AdminSession, AdminNotification
├── 71-admin-rbac.prisma         # Permission, RolePermission, RbacConfig
├── 72-admin-audit.prisma        # AdminAuditLog, ApprovalRequest
├── 73-admin-webhook.prisma      # AdminWebhook, WebhookDelivery
│
│ # ══════════════════════════════════════════════════════════════════
│ # BOUNDED CONTEXT: Trust & Relationships (Aggregate Root: TrustScore)
│ # ══════════════════════════════════════════════════════════════════
├── 80-trust.prisma              # TrustScore, TrustScoreHistory, FinancialBehaviorScore
├── 81-trust-relationship.prisma # FamilyRelationship
│
│ # ══════════════════════════════════════════════════════════════════
│ # BOUNDED CONTEXT: Messaging (Aggregate Root: Conversation)
│ # ══════════════════════════════════════════════════════════════════
├── 85-messaging.prisma          # Conversation, ConversationParticipant, Message
├── 86-messaging-encryption.prisma # UserSignalKey, SignalPreKey, GroupEncryptionKey
│
│ # ══════════════════════════════════════════════════════════════════
│ # SHARED INFRASTRUCTURE (Cross-cutting concerns)
│ # ══════════════════════════════════════════════════════════════════
├── 90-infra-cqrs.prisma         # OutboxCommand, ProjectorState, EventLog, EventDLQ
├── 91-infra-notification.prisma # Notification, PushToken, WalletNotification
├── 92-infra-analytics.prisma    # DailyAnalytics, MonthlyAnalytics, BalanceSnapshot
├── 93-infra-tax.prisma          # HoardingTaxSnapshot, VelocityTaxTimer, VelocityDailyBalance
│
│ # ══════════════════════════════════════════════════════════════════
│ # SHARED KERNEL (Referenced by all contexts)
│ # ══════════════════════════════════════════════════════════════════
├── 95-shared-country.prisma     # Country, LegalTenderStatus, CountryAllocation
├── 96-shared-licensing.prisma   # LicensedPartner, License, Credential
├── 97-shared-deployment.prisma  # DeploymentRecord, DeploymentLog, Application
│
│ # ══════════════════════════════════════════════════════════════════
│ # DEPRECATED (Pending removal or migration)
│ # ══════════════════════════════════════════════════════════════════
└── 99-deprecated.prisma         # Organization (legacy), OrganizationV2
```

### Bounded Context Mapping (DDD)

| Bounded Context | Aggregate Root | Owns Data | Cross-Context Refs |
|-----------------|----------------|-----------|-------------------|
| Identity | `UserProfile` | Users, Sessions, Devices | → Country (ID) |
| Wallet | `Wallet` | Transactions, SubAccounts | → UserProfile (ID) |
| KYC | `KYCApplication` | Documents, Verifications | → UserProfile (ID), AttorneyProfile (ID) |
| Organization | `OrganizationProfile` | BusinessAccounts, Signatories | → UserProfile (ID), Wallet (ID) |
| Government | `GovernmentTreasury` | Hierarchy, Approvals | → OrganizationProfile (ID) |
| NPO | `NotForProfitAccount` | Programs, Grants | → OrganizationProfile (ID) |
| Admin | `AdminUser` | Permissions, Audit | → UserProfile (ID) for references |
| Trust | `TrustScore` | History, BehaviorScores | → UserProfile (ID) |
| Messaging | `Conversation` | Messages, Encryption | → UserProfile (ID) |
| Infrastructure | N/A (services) | Outbox, Events, Analytics | → Various (ID only) |

### Implementation Steps

#### Step 1: Create config and enum files
Create `00-config.prisma` (generator + datasource) and `01-enums.prisma` (all 98 enums).

#### Step 2: Split models by bounded context
Extract models from `schema.prisma` into context-specific files:
- Identity context: `10-identity.prisma`, `11-identity-session.prisma`, `12-identity-security.prisma`
- Wallet context: `20-wallet.prisma` through `24-wallet-contact.prisma`
- Continue for all bounded contexts...

**Key principle**: Aggregate root goes first in each context, related entities follow.

#### Step 3: Handle cross-context relations
For each cross-context FK relation, decide:
- **Keep FK**: If genuinely needed for data integrity (e.g., Wallet → UserProfile)
- **ID-only**: If contexts should be independent (add `// Cross-context: ID reference only` comment)

#### Step 4: Update Prisma configuration
Update `packages/core-db/package.json`:
```json
{
  "prisma": {
    "schema": "../../db/prisma"
  }
}
```

Update all scripts to use folder path (remove `/schema.prisma` suffix).

#### Step 5: Validate and generate
```bash
npx prisma format --schema=./db/prisma
npx prisma validate --schema=./db/prisma
npx prisma generate --schema=./db/prisma
npx prisma migrate status --schema=./db/prisma
```

#### Step 6: Update CI/CD and scripts
- Add validation checks to `.github/workflows/ci.yml`
- Update `scripts/setup-local-dev.sh`
- Update all Dockerfiles

#### Step 7: Create ADR and backup
- Document decision in `docs/decisions/ADR-001-prisma-multi-file-schema.md`
- Keep `schema.prisma.backup` for rollback

---

## Part 4: Conventions (Enterprise Standards - DDD Aligned)

### File Naming
- Numeric prefixes for ordering (00-, 10-, 20-, etc.)
- Context name first, then sub-domain: `10-identity.prisma`, `11-identity-session.prisma`
- Gaps in numbering for future insertions (10, 20, 30... not 1, 2, 3)
- Aggregate root file has lowest number in context

### Enum Placement (Value Objects)
All enums in `01-enums.prisma` for simplicity. Consider splitting only if:
- Schema becomes very large (200+ enums)
- Clear domain ownership needs enforcement

### Model Naming
- PascalCase for model names
- Singular nouns (User, not Users)
- Context prefix for clarity: `GovernmentSignatory`, `NPOSignatory`, `BusinessSignatory`
- Aggregate roots: Clean names without prefix when possible (`Wallet`, `UserProfile`)

### Standard Fields (Entity Template)
```prisma
model Example {
  id        String   @id @default(uuid())
  tenantId  String

  // ... domain fields

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  deletedAt DateTime?  // Soft delete

  @@index([tenantId])
  @@map("example")  // Explicit table name
}
```

### Cross-Context References (DDD Boundaries)

**Principle**: Bounded contexts should be as independent as possible. Cross-context references should use IDs, not full Prisma relations, unless data integrity requires it.

| Scenario | Approach | Example |
|----------|----------|---------|
| Same bounded context | Full Prisma relation | `Wallet` ↔ `SubAccount` |
| Cross-context, integrity needed | FK relation with comment | `Wallet.profileId` → `UserProfile` |
| Cross-context, loose coupling | ID field (String), no FK | `TrustScore.userId` (just stores ID) |
| Shared Kernel reference | FK relation allowed | `*.countryCode` → `Country` |

**Comment convention for cross-context FKs**:
```prisma
model Wallet {
  // Cross-context: Identity bounded context
  profileId String
  profile   UserProfile @relation(fields: [profileId], references: [id])
}
```

### Aggregate Root Rules
1. Each bounded context has ONE primary aggregate root
2. External contexts reference only the aggregate root, not child entities
3. Child entities always reference their aggregate root

### File Header Template
```prisma
// =============================================================================
// BOUNDED CONTEXT: <Context Name>
// Aggregate Root: <Root Model>
// Owned by: <service name>
// =============================================================================
```

---

## Part 5: Safety Nets

### CI Checks (add to .github/workflows/ci.yml)
```yaml
- name: Validate Prisma schema
  run: npx prisma validate --schema=./db/prisma

- name: Check Prisma formatting
  run: npx prisma format --schema=./db/prisma --check

- name: Generate Prisma client
  run: npx prisma generate --schema=./db/prisma
```

### Rollback Plan
1. Backup exists: `schema.prisma.backup` in `db/prisma/`
2. Revert files:
   ```bash
   cd db/prisma
   rm -f *.prisma  # Remove split files
   mv schema.prisma.backup schema.prisma
   ```
3. Revert `packages/core-db/package.json` prisma path to `../../db/prisma/schema.prisma`
4. Run `npx prisma generate --schema=./db/prisma/schema.prisma`

### How to Add a New Model (DDD Guide)

```markdown
## Adding a New Model

1. **Identify the bounded context**
   - Which aggregate root does this entity belong to?
   - Example: A new `PaymentMethod` belongs to Wallet context

2. **Choose the correct file**
   - If extending aggregate root: add to main context file (e.g., `20-wallet.prisma`)
   - If new sub-entity: add to appropriate sub-file (e.g., `21-wallet-transaction.prisma`)
   - If new aggregate root: create new file with next available number

3. **Add the model following DDD conventions**
   ```prisma
   model PaymentMethod {
     id        String   @id @default(uuid())
     tenantId  String

     // Aggregate root reference (same context)
     walletId  String
     wallet    Wallet @relation(fields: [walletId], references: [id])

     // Domain fields
     type      PaymentMethodType
     isDefault Boolean @default(false)

     createdAt DateTime @default(now())
     updatedAt DateTime @updatedAt
     deletedAt DateTime?

     @@index([tenantId])
     @@index([walletId])
     @@map("payment_method")
   }
   ```

4. **If adding cross-context reference**
   - Add comment: `// Cross-context: <context name>`
   - Prefer ID-only unless FK integrity is required

5. **Validate and format**
   ```bash
   npm run prisma:format
   npm run prisma:validate
   ```

6. **Create migration**
   ```bash
   npm run migrate:create -- --name add_payment_method
   ```

7. **Review generated SQL and apply**
   ```bash
   npm run migrate:dev
   ```
```

---

## Part 6: Verification Plan

### Local Verification
```bash
cd gx-protocol-backend

# 1. Validate schema structure
npx prisma validate --schema=./db/prisma

# 2. Format check (will reformat if needed)
npx prisma format --schema=./db/prisma

# 3. Generate client
npx prisma generate --schema=./db/prisma

# 4. Check migration status (should show all applied)
npx prisma migrate status --schema=./db/prisma

# 5. Run existing tests
npm test

# 6. Start a service to verify client works
npm run dev --workspace=apps/svc-identity
```

### CI Verification
- All existing tests pass
- New validation checks pass
- Docker builds succeed
- Prisma client generates correctly

---

## Files to Modify

### Primary Changes
| File | Change |
|------|--------|
| `db/prisma/*.prisma` | New split files (30+ files) |
| `db/prisma/schema.prisma` | Rename to `.backup` |
| `packages/core-db/package.json` | Change `"schema": "../../db/prisma"` |
| `gx-protocol-backend/package.json` | Update migrate script paths |
| `.github/workflows/ci.yml` | Add validate/format checks |
| `scripts/setup-local-dev.sh` | Update prisma commands |
| `apps/*/Dockerfile` | Update `--schema=./db/prisma` |

### New Files
| File | Purpose |
|------|---------|
| `docs/decisions/ADR-001-prisma-multi-file-schema.md` | Architecture Decision Record |
| `db/prisma/00-config.prisma` | Generator + Datasource |
| `db/prisma/01-enums.prisma` | All 98 enums |
| `db/prisma/10-identity.prisma` | Identity aggregate root |
| `db/prisma/11-identity-session.prisma` | Session entities |
| ... (30+ context files) | ... |
| `db/prisma/99-deprecated.prisma` | Legacy models |

---

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Migration breaks | Low | High | Test with copy of prod DB first |
| IDE tooling issues | Medium | Low | Document VSCode Prisma extension setup |
| Merge conflicts during transition | Medium | Medium | Merge PR quickly, communicate with team |
| Enum duplication | Low | Medium | Strict placement rules + PR review |

---

## Success Criteria

1. `prisma validate` passes on new structure
2. `prisma generate` produces identical client
3. All existing migrations work unchanged
4. All existing tests pass
5. CI pipeline passes
6. Services start and connect to DB correctly

---

## Technical Warnings

### Do NOT Upgrade to Prisma 7.x
There is a **known regression in Prisma 7.0.0** where multi-file schemas do not work correctly ([GitHub Issue #28673](https://github.com/prisma/prisma/issues/28673)). Stay on **6.17.1** until resolved.

### VSCode Extension Limitation
The Prisma VSCode extension cannot auto-generate opposite relation fields across files. Workaround:
- Run `npx prisma format` after adding cross-file relations
- Or manually add both sides of the relation

### No Preview Flag Needed
Multi-file schemas became GA in Prisma 6.7.0. Do NOT add `previewFeatures = ["prismaSchemaFolder"]` - it's no longer recognized and will cause warnings.

---

## PR Summary

### What Changed
- Split `schema.prisma` (5000+ lines, 149 models) into 30+ domain-specific `.prisma` files
- Organized by DDD bounded contexts (Identity, Wallet, KYC, Organization, Government, NPO, Admin, Trust, Messaging, Infrastructure)
- Added CI validation checks (`prisma validate`, `prisma format --check`)
- Created ADR documenting the decision

### Why
- Maintainability: 149 models in one file is unmanageable
- Code review: Changes isolated to domain files
- DDD alignment: Schema structure mirrors bounded contexts
- Team productivity: Clear ownership boundaries

### How to Verify Locally
```bash
cd gx-protocol-backend
npx prisma validate --schema=./db/prisma
npx prisma generate --schema=./db/prisma
npm test
npm run dev --workspace=apps/svc-identity
```

### Rollback
If issues arise, restore from `schema.prisma.backup` and revert package.json paths.
