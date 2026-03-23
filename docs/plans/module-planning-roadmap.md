# GX Protocol — Module Roadmap

## Context

Modules 1-3 (Token Economics, Government Treasury, Identity/KYC/Trust) are complete. The codebase has 16 backend services, 3 frontend apps, 7 chaincode contracts, and 169 Prisma models. This plan organizes all remaining work into clearly scoped modules with dependency mapping and parallel execution tracks.

## Methodology: Spec-First

Before any implementation, create a detailed specification document for each module in `docs-dev-manazir/specs/`. Each spec covers requirements, UI/UX, API contracts, data models, acceptance criteria, and edge cases. A master `docs-dev-manazir/specs/README.md` tracks the checklist of all modules and their spec status.

**Workflow per module:**
1. Draft spec from audit findings + developer input
2. Q&A session to refine requirements and close gaps
3. Finalize spec document
4. Implementation follows spec

---

## Dependency Graph

```
PHASE 1 (parallel, no blockers)
├── MOD-4  CC Participant Management ──┐
├── MOD-7  Wallet Dashboard Polish     │
├── MOD-8  Send Page + Fiat Calc       │
└── MOD-13 KYC Review Queue            │
                                       │
PHASE 2 (after MOD-4)                  │
├── MOD-5  SSO + Admin Registration ───┤  ◀ MAJOR (OAuth2 IdP)
│                                      │
PHASE 3 (after MOD-5)                  │
├── MOD-6  Hub (Post-Login Router) ────┤  ◀ CRITICAL GATE
│                                      │
PHASE 4 (after MOD-6, parallel)        │
├── MOD-9  Government Onboarding       │
├── MOD-10 Institution Onboarding ─────┤
├── MOD-12 Messaging Deploy            │  (deferred from Phase 1)
│                                      │
PHASE 5 (after MOD-10)                 │
├── MOD-11 Partner Onboarding          │
│                                      │
PHASE 6                                │
└── MOD-14 E2E Testing ◀──────────────┘
```

**Critical path**: MOD-4 → MOD-5 (SSO) → MOD-6 (Hub) → MOD-10 → MOD-11 → MOD-14

---

## Module Definitions

### MOD-4: Command Center — Individual Participant Management

| | |
|---|---|
| **Complexity** | M |
| **Dependencies** | None |
| **Blocks** | MOD-5 |
| **Apps** | command-center |
| **Services** | svc-admin (existing APIs + new filter params) |

**Deliverables:**

1. **Sidebar cleanup** — Remove "Attorneys" and "Sanctions Screening" entirely (D28)
   - File: `command-center/src/navigation/sidebar/sidebar-items.ts`

2. **Users table overhaul** — Enterprise data table with per-column sort+filter
   - File: `command-center/src/app/(main)/dashboard/users/_components/users-table.tsx`
   - **Columns** (D4): Name+Email, Fabric ID, On-chain Status, Nationality, Created Date
   - **Remove** 3-dot action menu, make rows clickable → `/dashboard/users/[id]` (D6)
   - **Column header filters** (D5): click filter icon → dropdown/input per column type
   - **Server-side** sort/filter/pagination (D9): generic filter syntax `?filter[status]=ACTIVE&sort=createdAt:desc` (D26)
   - **Pagination** (D42): 25 default, selectable 10/25/50/100, "Page X of Y"
   - Fix null/dash rendering — audit `useUsers` hook data

3. **Tab reduction** — Keep "On-chain" and "Pending On-chain" only. Remove Suspended, Locked, Deactivated, Banned tabs. Filters replace status-based tabs.

4. **Backend: enhance listUsers endpoint** — Add generic filter/sort params to svc-admin user-management controller

5. **User Detail Page** — Keep tabs + sidebar layout (D10)
   - File: `command-center/src/app/(main)/dashboard/users/[id]/page.tsx`
   - Keep all 10 tabs (D7): Profile, Documents, Activity, Sessions, Wallet, Accounts, Transactions, Trust Score, Compliance, Tickets
   - Real data: Profile, Documents, Activity, Sessions
   - Placeholder with "Integration pending" message: Wallet, Accounts, Transactions, Trust Score, Compliance, Tickets
   - **Expanded admin actions** (D35): Add Force Password Reset, Force 2FA Reset, Trigger KYC Re-verification, Send Notification, Export User Data, Revoke All Sessions

---

### MOD-5: SSO Architecture + Admin Registration

| | |
|---|---|
| **Complexity** | L (upgraded from M due to SSO) |
| **Dependencies** | MOD-4 (soft) |
| **Blocks** | MOD-6 |
| **Apps** | user-wallet, command-center, institutional-admin-center, backend-core |
| **Services** | svc-auth (OAuth2 IdP), svc-admin (new endpoints) |

**MAJOR SCOPE: This module now includes SSO architecture (D11, D13, D17, D41)**

**Deliverables:**

1. **svc-auth as OAuth2 Identity Provider** (D13, D41):
   - Add OAuth2 endpoints: `/authorize`, `/token`, `/userinfo`
   - JWT with basic claims: profileId, email, name (D25)
   - Apps call `GET /api/v1/auth/contexts` after login for roles/contexts
   - Authorization code flow for cross-app token relay (D41)

2. **user-wallet hosts THE login** (D17):
   - Existing `/login` page becomes the single login for all apps
   - After login → check contexts → Hub or direct to `/dashboard`
   - command-center and IAC redirect to user-wallet for auth

3. **command-center auth migration**:
   - Remove standalone username/password login (except superowner — D33)
   - Accept OAuth2 auth code from user-wallet
   - Exchange code for token via svc-auth
   - Keep Zustand store but populate from OAuth2 token instead of local credentials

4. **Admin creation form redesign**:
   - File: `command-center/src/app/(main)/dashboard/admins/create/page.tsx`
   - Participant-first lookup flow (email or fabricId → verify on-chain → user card → role → permissions → confirm)
   - No separate password — admin authenticates via SSO
   - Backend: `POST /api/v1/admin/admins/from-participant` with `{ profileId, role, permissions[] }`
   - Backend: `GET /api/v1/admin/users/lookup?email=x&fabricId=y`

5. **Superowner exception** (D33): Keep separate command-center login for superowner only

---

### MOD-6: Hub (Post-Login Router)

| | |
|---|---|
| **Complexity** | L |
| **Dependencies** | MOD-5 (SSO) |
| **Blocks** | MOD-9, MOD-10, MOD-11 |
| **Apps** | user-wallet, backend-core |
| **Services** | svc-auth (contexts endpoint from MOD-5) |

**Current state**: `user-wallet/app/(root)/(client)/context-selector/page.tsx` exists but is not a post-login routing layer.

**Deliverables:**

1. **Backend contexts endpoint** (built in MOD-5): `GET /api/v1/auth/contexts`
   - Returns: `{ personalWallet: true, institutions: [{orgId, name, role, notificationCount}], treasuryAssignments: [{treasuryId, country, role, notificationCount}], adminAccess: {isAdmin, role, notificationCount} | null }`

2. **Post-login routing logic** (D22):
   - Triggered on EVERY login
   - Single context (wallet only) → skip to `/dashboard`
   - Multiple contexts → show Hub at `/hub`
   - Re-evaluated each login (contexts can change between sessions)

3. **Redesign context-selector into Hub** (D12):
   - File: `user-wallet/app/(root)/(client)/hub/page.tsx` (rename from context-selector)
   - Horizontally swipeable card sections:
     - **Wallets card**: Personal Wallet + institutional wallets
     - **Administration card** (conditional): Government, Institutional admin, Command Center
   - **Notification badges on each card** (D37): e.g., "3 pending approvals", "2 new messages"
   - Each item clickable:
     - Personal wallet → `/dashboard`
     - Institution wallet → `/entities/institutions/[orgId]`
     - Government treasury → `/government/[treasuryId]`
     - Institutional admin → OAuth2 auth code → IAC
     - Command Center → OAuth2 auth code → command-center
   - Hub link in sidebar/header for manual navigation when skipped

---

### MOD-7: Wallet Dashboard, Settings, PWA & Dark Mode

| | |
|---|---|
| **Complexity** | L (upgraded from M — adds PWA, dark mode, error standardization) |
| **Dependencies** | None |
| **Blocks** | None |
| **Apps** | user-wallet |
| **Services** | svc-wallet or svc-auth (user preferences endpoint) |

**Deliverables:**

1. **Dashboard refinements**:
   - File: `user-wallet/app/(root)/(client)/dashboard/page.tsx`
   - **Customizable quick actions** (D14, D31): user picks 4 from 10+ master list. Stored localStorage + backend sync. Master list: Send, Receive, QR Pay, Velocity Tax, Trust Network, Contacts, Messages, Settings, Transactions, Analytics, KYC Status, Institution Create
   - "Create Institutional Account" promotional card at bottom
   - Ensure all widgets render real data

2. **Settings page expansion**:
   - Profile card, KYC card (exists), KYR card (exists), App Settings (theme, default currency, fiat toggle, quick actions config), Management Settings (privacy, export data), Security (exists)
   - "Create Institutional Account" card

3. **Dark mode** (D36): Add light/dark theme toggle to user-wallet. Use Tailwind dark: classes.

4. **PWA** (D24): manifest.json, service worker, offline fallback page, install prompt banner

5. **Standardized error/loading/empty components** (D39): Create reusable ErrorBoundary, EmptyState, LoadingState. Apply across user-wallet.

6. **User preferences backend**: New endpoint to store/retrieve user preferences (quick actions, theme, default currency, fiat toggle)

---

### MOD-8: Send Page — Fiat Conversion + Transfer Security

| | |
|---|---|
| **Complexity** | M |
| **Dependencies** | None |
| **Blocks** | None |
| **Apps** | user-wallet |
| **Services** | svc-auth (PIN/2FA verification) |

**Current state**: Send page has 4-step TransferFlow. No fiat conversion. No PIN/2FA on transfers.

**Deliverables:**

1. **Fiat conversion utility** (D15):
   - Copy from `the-website/src/lib/genesisCurrencyData.ts` — 150+ currencies, calculator, conversion algorithms
   - Refine for send page use case
   - Algorithm: 1 GX = 1 gram gold, genesis day rates per currency

2. **Dual-input AmountInput** (D29):
   - File: `user-wallet/components/send/AmountInput.tsx`
   - Toggle: GX-only (setting OFF) or GX+Fiat dual display (setting ON)
   - Two linked inputs: type fiat → GX auto-fills, type GX → fiat auto-fills
   - Currency selector dropdown, saved to user preferences
   - Controlled from app settings (fiat toggle from MOD-7)

3. **Transfer security** (D21):
   - **PIN for ALL transfers**: 4-6 digit PIN confirmation step after Review
   - **2FA for large transfers**: TOTP additionally required above configurable threshold
   - Flow: Compose → Review → PIN → (2FA if large) → Processing → Receipt
   - Uses existing svc-auth: verify-pin, verify-2fa endpoints

4. **Registered users only** (D38): No invite/claim links. Recipient must have fabricUserId.

5. **Contacts page verification**: Ensure `/beneficiaries/` works end-to-end

---

### MOD-9: Government Onboarding and Access

| | |
|---|---|
| **Complexity** | M (upgraded — admin-initiated, custom per government) |
| **Dependencies** | MOD-6 (Hub) |
| **Blocks** | None |
| **Apps** | command-center, user-wallet |
| **Services** | svc-government (existing, verify) |

**Onboarding model** (D18): Admin-only from command-center. Created by GX admins after agreements/meetings. Custom per government.

**Deliverables:**

1. **Government creation in command-center**: Admin creates treasury from `/dashboard/treasury/` (pages already exist). Customize per-government setup.
2. Wire government access into Hub (MOD-6 delivers the card)
3. Verify government treasury dashboard pages in user-wallet connect to svc-government APIs end-to-end
4. Verify onboarding wizard flow works for treasury activation
5. Defer separate government-admin app — existing user-wallet government pages are sufficient for now

---

### MOD-10: Institution Onboarding and Admin Center

| | |
|---|---|
| **Complexity** | L (upgraded from M — multi-step wizard) |
| **Dependencies** | MOD-6 (Hub) |
| **Blocks** | MOD-11 |
| **Apps** | user-wallet, institutional-admin-center, backend-core |
| **Services** | svc-organization, svc-onboarding |

**Current state**: Institution create/list/detail pages exist in user-wallet. IAC app exists with 18 pages.

**Onboarding model** (D18): Self-service by participants from user-wallet.

**Deliverables:**

1. **Multi-step institution creation wizard** (D23, D34):
   - File: `user-wallet/app/(root)/(client)/entities/institutions/create/`
   - Step 1: Basic info (name, type [Business/NGO/Institution], country)
   - Step 2: Business details (registration number, address, description)
   - Step 3: Documents (registration cert, articles of incorporation)
   - Step 4: Stakeholders/signatories
   - Step 5: Review & submit
   - **Auto-save per step** (D34): InstitutionOnboardingSession table. Resume later.
   - Robust validation like KYC wizard

2. Wire institutional wallet access into Hub (MOD-6)
3. Verify OAuth2 token relay from user-wallet → IAC works (replaces hash handoff)
4. Deploy svc-organization if not already running
5. Ensure OrganizationV2, OrganizationStakeholder models are properly served

---

### MOD-11: Partner/FSP Onboarding

| | |
|---|---|
| **Complexity** | L |
| **Dependencies** | MOD-10 |
| **Blocks** | None |
| **Apps** | command-center, backend-core |
| **Services** | svc-admin or svc-organization |

**Onboarding model** (D18): Admin-only from command-center. Created after agreements. Custom per partner.

**Deliverables:**

1. **Partner creation in command-center**: Admin creates partners from `/dashboard/partners/` (pages exist)
2. **Scoped API credentials** (D27): Full CRUD with scopes (read:users, write:transfers, read:balances, etc.)
3. API credential management for approved partners (PartnerApiCredential model)
4. Backend: partner CRUD use cases using existing PartnerProfile, PartnerApiCredential, PartnerSlaMetric, PartnerGrant models
5. SLA monitoring dashboard in command-center

---

### MOD-12: Messaging Service Deployment

| | |
|---|---|
| **Complexity** | M |
| **Dependencies** | None |
| **Blocks** | None |
| **Apps** | user-wallet (already built), infra-ops |
| **Services** | svc-messaging (deploy) |

**Current state**: svc-messaging has full source code (controllers, websocket, compliance). Messages page in user-wallet is fully built.

**Deliverables:**

1. Build svc-messaging Docker image
2. Create K8s manifest, deploy to backend-dev-manazir
3. Verify websocket connectivity through K8s ingress
4. Connect user-wallet messages page to deployed service
5. Test direct messaging, group chat end-to-end

---

### MOD-13: KYC Review Queue Polish (Command Center)

| | |
|---|---|
| **Complexity** | S |
| **Dependencies** | None |
| **Blocks** | None |
| **Apps** | command-center |
| **Services** | svc-admin, svc-kyc (existing) |

**Deliverables:**

1. Audit KYC review queue at `/dashboard/kyc` — verify document viewer, face verification results, approve/reject workflows
2. Ensure document preview works via presigned URLs (recently fixed in Session 6)
3. Polish the review UI for admin workflow efficiency
4. Verify status transitions: PENDING → IN_REVIEW → APPROVED/REJECTED

---

### MOD-14: Comprehensive E2E Testing

| | |
|---|---|
| **Complexity** | XL |
| **Dependencies** | All modules (runs last) |
| **Blocks** | Production readiness |
| **Apps** | All |
| **Services** | All |

**Deliverables:**

1. E2E: Registration → KYC → First Transaction
2. E2E: Admin login → User management → KYC review → Approve
3. E2E: Government treasury onboarding → Account creation → Multi-sig approval
4. E2E: Institution creation → Stakeholder endorsement → Transaction
5. E2E: Partner application → Approval → API credential generation
6. E2E: Access Panel routing — all 4 cases
7. Integration tests for backend services
8. Frontend Playwright tests for critical journeys

---

## Execution Phases

### Phase 1 — Parallel Foundation (no blockers)

| Module | Track | Notes |
|--------|-------|-------|
| **MOD-4** CC Participant Management | A — Command Center | Frontend only, clean up existing pages |
| **MOD-7** Wallet Dashboard Polish | B — Wallet | Frontend only, polish existing components |
| **MOD-8** Send Page + Fiat Calc | B — Wallet | Frontend only, client-side math |
| **MOD-12** Messaging Deploy | C — Infra | Docker + K8s, connect existing UI |
| **MOD-13** KYC Review Queue | A — Command Center | Polish + verify existing |

All 5 modules can run simultaneously.

### Phase 2 — Admin Architecture

| Module | Track | Notes |
|--------|-------|-------|
| **MOD-5** Admin Registration | A — Command Center | Requires MOD-4 (soft). Backend + frontend. |

### Phase 3 — The Critical Gate

| Module | Track | Notes |
|--------|-------|-------|
| **MOD-6** Access Panel | D — Cross-App | Backend aggregation + frontend redesign. Unlocks all entity modules. |

### Phase 4 — Entity Onboarding (parallel after MOD-6)

| Module | Track | Notes |
|--------|-------|-------|
| **MOD-9** Government Access | D — Entities | Mostly verification, pages already exist |
| **MOD-10** Institution Onboarding | D — Entities | Verify flows, deploy svc-organization |

### Phase 5 — Partners

| Module | Track | Notes |
|--------|-------|-------|
| **MOD-11** Partner Onboarding | D — Entities | After MOD-10 |

### Phase 6 — Quality

| Module | Track | Notes |
|--------|-------|-------|
| **MOD-14** E2E Testing | E — QA | After all modules |

---

## Summary

| ID | Module | Size | Deps | Phase | Spec Status |
|----|--------|------|------|-------|-------------|
| MOD-4 | CC Participant Management | M | — | 1 | PENDING |
| MOD-5 | SSO + Admin Registration | L | MOD-4 | 2 | PENDING |
| MOD-6 | Hub (Post-Login Router) | L | MOD-5 | 3 | PENDING |
| MOD-7 | Dashboard, Settings, PWA, Dark Mode | L | — | 1 | PENDING |
| MOD-8 | Send Page + Fiat + Transfer Security | M | — | 1 | PENDING |
| MOD-9 | Government Onboarding | M | MOD-6 | 4 | PENDING |
| MOD-10 | Institution Onboarding | L | MOD-6 | 4 | PENDING |
| MOD-11 | Partner Onboarding | L | MOD-10 | 5 | PENDING |
| MOD-12 | Messaging Deploy | M | — | 4+ | PENDING |
| MOD-13 | KYC Review Queue | S | — | 1 | PENDING |
| MOD-14 | E2E Testing | XL | All | 6 | PENDING |
| MOD-15 | Governance & Voting | L | MOD-6 | Future | PENDING |
| MOD-16 | Loan Pool | L | MOD-10 | Future | PENDING |
| MOD-17 | Tax Engine | M | MOD-8 | Future | PENDING |
| MOD-18 | Trust Score UI + Neo4j | L | — | Future | PENDING |
| MOD-19 | Compliance & Sanctions | M | MOD-4 | Future | PENDING |
| MOD-20 | Support Ticketing | M | — | Future | PENDING |
| MOD-21 | Reporting & Analytics | L | MOD-14 | Future | PENDING |
| MOD-22 | Notification System | M | MOD-12 | Future | PENDING |

---

## Next Steps (Implementation Plan)

1. **Exit plan mode** → Create `docs-dev-manazir/specs/README.md` (master checklist)
2. **Write SPEC-MOD-4** → Full enterprise spec for CC Participant Management
3. **Write SPEC-MOD-7** → Dashboard, Settings, PWA, Dark Mode (parallel with MOD-4)
4. **Write SPEC-MOD-8** → Send Page + Fiat + Transfer Security (parallel)
5. **Write SPEC-MOD-13** → KYC Review Queue (parallel)
6. **Write SPEC-MOD-5** → SSO Architecture (critical, needs deep spec)
7. **Write SPEC-MOD-6** → Hub
8. Continue specs for remaining modules...
9. **Update work record** with this planning session narrative

---

## Pending Work Record

**Session: 2026-02-25 — Project Planning & Module Roadmap**

To be written when exiting plan mode. Covers:
- Full codebase audit (3 parallel explore agents)
- Module roadmap design (22 modules identified)
- 43 architectural decisions via structured Q&A
- Spec-first methodology established
- Key decisions: Unified SSO (OAuth2 IdP), Hub naming, tiered transfer security, PWA, multi-step institution wizard, server-side filtering, dark mode
| MOD-15 | Governance/Voting | L | MOD-6 | Future |
| MOD-16 | Loan Pool | L | MOD-10 | Future |
| MOD-17 | Tax Engine (Velocity + Hoarding) | M | MOD-8 | Future |
| MOD-18 | Trust Score UI + Neo4j | L | — | Future |
| MOD-19 | Compliance & Sanctions | M | MOD-4 | Future |
| MOD-20 | Support Ticketing System | M | — | Future |
| MOD-21 | Reporting & Analytics Engine | L | MOD-14 | Future |
| MOD-22 | Notification System (Push/Email/SMS) | M | MOD-12 | Future |

---

## Additional Modules Identified (Full Audit)

### MOD-15: Governance & Voting
- **Scope**: On-chain proposals, voting mechanisms, execution
- **Backend**: svc-governance (stub exists, GovernanceContract chaincode implemented with 5 functions)
- **Frontend**: command-center `/dashboard/governance` (stub page), user-wallet voting UI needed
- **Prisma**: Proposal model exists
- **Deps**: MOD-6 (participants need context to vote)

### MOD-16: Loan Pool
- **Scope**: Interest-free peer lending, collateral management, repayment tracking
- **Backend**: svc-loanpool (stub exists, LoanPoolContract chaincode implemented)
- **Frontend**: command-center `/dashboard/loans` (stub), user-wallet loan pages needed
- **Prisma**: Loan model exists
- **Deps**: MOD-10 (institutions can be lenders)

### MOD-17: Tax Engine (Velocity + Hoarding)
- **Scope**: Automated velocity tax calculation, hoarding tax cycles, fee distribution
- **Backend**: svc-tax (stub exists, TaxAndFeeContract chaincode implemented)
- **Frontend**: user-wallet `/velocity-tax` page exists, needs real data connection
- **Prisma**: HoardingTaxSnapshot, VelocityTaxTimer, VelocityDailyBalance, VelocityTaxLog, FeeRate models exist
- **Deps**: MOD-8 (transfer fee integration)

### MOD-18: Trust Score UI + Neo4j Integration
- **Scope**: Visual trust graph, family relationship management, score calculation
- **Backend**: svc-trust (partially running, Neo4j deferred), core-graph package exists
- **Frontend**: user-wallet `/relationships/graph` page exists (stub)
- **Prisma**: TrustScore, TrustScoreHistory, FamilyRelationship, FinancialBehaviorScore models exist
- **Deps**: None (independent)

### MOD-19: Compliance & Sanctions
- **Scope**: Sanctions screening, OFAC SDN list integration, compliance rules engine, alerts
- **Backend**: svc-admin compliance endpoints exist (listRules, createRule, listAlerts, runScreening)
- **Frontend**: command-center `/dashboard/compliance/*` (4 sub-pages, partially implemented), `/dashboard/sanctions`
- **Prisma**: ComplianceRule, ComplianceAlert, SanctionsScreening, OFACSdnEntry models exist
- **Deps**: MOD-4 (participant management context)

### MOD-20: Support Ticketing System
- **Scope**: Admin support ticket queue, knowledge base, participant-facing help
- **Backend**: svc-admin support endpoints exist (listTickets, createTicket, assignTicket, escalateTicket, listArticles)
- **Frontend**: command-center `/dashboard/support/tickets`, `/dashboard/support/knowledge-base`
- **Prisma**: SupportTicket, TicketMessage, KnowledgeBaseArticle models exist
- **Deps**: None (independent)

### MOD-21: Reporting & Analytics Engine
- **Scope**: Scheduled report generation, dashboard KPIs, data export, analytics dashboards
- **Backend**: svc-admin reporting endpoints exist (listReports, generateReport, scheduled reports)
- **Frontend**: command-center `/dashboard/reports`, user-wallet `/analytics` (stub)
- **Prisma**: DailyAnalytics, MonthlyAnalytics, BalanceSnapshot models exist
- **Deps**: MOD-14 (needs working data flows first)

### MOD-22: Notification System (Push/Email/SMS)
- **Scope**: Multi-channel notifications, push tokens, email templates, SMS gateway
- **Backend**: svc-admin notification endpoints exist, svc-messaging has notification support
- **Frontend**: command-center `/dashboard/notifications`, user-wallet notification UI
- **Prisma**: Notification, PushToken, WalletNotification, NotificationPreferences models exist
- **Deps**: MOD-12 (messaging infrastructure)

---

## Decisions Log

### Session: 2026-02-25

**Decision 1: Spec location**
- Specs go in `docs-dev-manazir/specs/`
- Naming: `SPEC-MOD-{N}-{SHORT-NAME}.md`
- Master checklist: `docs-dev-manazir/specs/README.md`

**Decision 2: Spec depth**
- Full enterprise specs (requirements, UI wireframes, API contracts, data models, state machines, edge cases, acceptance criteria)

**Decision 3: Module order for spec writing**
- Start with MOD-4 (CC Participant Management), then proceed through roadmap

**Decision 4: MOD-4 — Table columns (CONFIRMED)**
- Name + Email
- Fabric ID
- On-chain Status (replaces generic "Status")
- Nationality (country)
- Created Date
- NO: Phone, Gender, DOB, KYC Status, Balance, Last Transaction (not in first version)

**Decision 5: MOD-4 — Filter design (CONFIRMED)**
- Column header filters (click filter icon on column header → dropdown/input appears)
- Enterprise pattern (Jira/Salesforce style)

**Decision 6: MOD-4 — Row click behavior (CONFIRMED)**
- Navigate to full detail page `/dashboard/users/[id]`
- Remove 3-dot action menu entirely

**Decision 7: MOD-4 — Detail page tabs (CONFIRMED)**
- Keep all 10 tabs
- Tabs with real data: Profile, Documents, Activity, Sessions
- Tabs with placeholder: Wallet, Accounts, Transactions, Trust Score, Compliance, Tickets
- Placeholders show clear "Integration pending" messaging

**Decision 8: Work records**
- Narrative storytelling style — comprehensive, descriptive journal/memory
- Update work record when exiting plan mode

**Decision 9: MOD-4 — Sort/Filter (CONFIRMED)**
- Server-side sorting and filtering
- Backend already supports `status`, `search`, `page`, `limit` params
- Need to add: `sortBy`, `sortOrder`, column-specific filter params to svc-admin listUsers endpoint

**Decision 10: MOD-4 — Detail page layout (CONFIRMED)**
- Keep current tabs + sidebar layout
- Polish UI, fix mock data, don't over-engineer the layout

**Decision 11: MOD-5 — Admin auth model (CONFIRMED: Unified SSO)**
- Build a proper SSO/OAuth system
- Single login page for all apps
- svc-auth becomes the central Identity Provider (IdP)
- No more separate admin username/password in command-center
- All apps authenticate through svc-auth
- This is a significant architectural change — needs its own detailed spec
- IMPACT: Changes MOD-5 scope from "admin registration form" to "SSO architecture + admin role promotion"
- IMPACT: Changes MOD-6 from "Access Panel" to "Hub" with SSO-integrated context switching

**Decision 12: MOD-6 — Naming (CONFIRMED: "Hub")**
- Page name: "Hub"
- Route: `/hub`
- Concept: Central launchpoint after SSO login where user sees all available contexts

**Decision 13: SSO Flow (CONFIRMED: OAuth2 IdP)**
- svc-auth becomes the OAuth2 Identity Provider
- All apps redirect to single login page
- After login, user gets JWT, sees Hub, picks context
- Each app validates the same JWT
- ARCHITECTURAL CHANGE: This is the biggest single change in the roadmap
- Needs dedicated deep spec document

**Decision 14: MOD-7 — Quick actions storage (CONFIRMED: Both)**
- localStorage for instant response (no loading)
- Background sync to backend UserPreferences
- On new device, pull from backend and populate localStorage
- Requires new UserPreferences endpoint in svc-wallet or svc-auth

**Decision 15: MOD-8 — Currency data (CONFIRMED: Copy from the-website)**
- Copy genesisCurrencyData.ts + calculator + conversion algorithms from the-website
- Refine for send page use case (the-website has the full calculator already)
- Source: `the-website/src/lib/genesisCurrencyData.ts` + price signal calculator components

**Decision 16: MOD-12 — Messaging (DEFERRED to Phase 4+)**
- Not a priority — focus on core flows first
- Move MOD-12 from Phase 1 to Phase 4+
- UI already built, just needs backend deployment when ready

**Decision 17: SSO Login Page Host (CONFIRMED: user-wallet)**
- user-wallet's existing /login page becomes THE login for all apps
- command-center and IAC redirect to user-wallet for auth
- After login, user-wallet issues JWT, shows Hub
- User picks context → token relayed to target app
- Reuses existing NextAuth.js setup in user-wallet

**Decision 18: Onboarding Model (CONFIRMED: Split)**
- **Institutions (Business/NGO)**: Self-service by participants from user-wallet
  - On-chain user creates business/NGO/institution account from wallet
  - No admin approval needed for creation (but may need for activation)
- **Government**: Admin-only from command-center
  - Created by GX admins after agreements/meetings
  - Custom per-government setup
  - NOT self-service
- **Partners/FSPs**: Admin-only from command-center
  - Created by GX admins after agreements
  - Custom per-partner setup
  - NOT self-service

**Decision 19: i18n (CONFIRMED: English-only)**
- Ship in English first
- Use string constants (not hardcoded literals) for future i18n readiness
- No i18n framework yet

**Decision 20: Mobile (CONFIRMED: PWA)**
- Progressive Web App for user-wallet
- Add service worker, manifest.json, offline support
- Installable from browser on mobile
- Already responsive (Tailwind mobile-first)

**Decision 21: Transfer Security (CONFIRMED: Tiered)**
- PIN confirmation for EVERY transfer (4-6 digit)
- 2FA (TOTP) additionally required for transfers above configurable threshold
- Backend already has: set-pin, verify-pin, enable-2fa, verify-2fa in svc-auth
- Transfer flow: Compose → Review → PIN → (2FA if large) → Processing → Receipt

**Decision 22: Hub Skip Logic (CONFIRMED: Dynamic per-login)**
- Hub check triggered on EVERY login
- If single context (wallet only) → skip to /dashboard
- If multiple contexts → show Hub
- Decision is re-evaluated each login (contexts can change between sessions)
- Hub always accessible via sidebar/header link for manual navigation

**Decision 23: Institution Creation (CONFIRMED: Multi-step Wizard)**
- Full onboarding wizard similar to KYC
- Step 1: Basic info (name, type, country)
- Step 2: Business details (registration number, address, description)
- Step 3: Documents (registration cert, articles of incorporation)
- Step 4: Stakeholders/signatories
- Step 5: Review & submit
- Robust validation like KYC flow
- Lives in user-wallet (self-service for participants)

**Decision 24: PWA Timing (CONFIRMED: Phase 1 with MOD-7)**
- Add PWA support during MOD-7 dashboard polish
- manifest.json, service worker, offline fallback, install prompt
- Small effort alongside dashboard refinements

**Decision 25: JWT Contents (CONFIRMED: Basic claims + /contexts API)**
- JWT: profileId, email, name (minimal)
- After login, apps call `GET /api/v1/auth/contexts` for roles/contexts
- Context always fresh (not stale in JWT)
- Standard OAuth2 pattern

**Decision 26: Filter API (CONFIRMED: Generic filter syntax)**
- Support: `?filter[status]=ACTIVE&filter[country][in]=IN,US&sort=createdAt:desc`
- Flexible, handles future columns
- Backend parsing: parse filter object from query params
- Also keep existing `search` param for full-text search

**Decision 27: Partner API Access (CONFIRMED: Full CRUD with scopes)**
- Scoped API keys: read:users, write:transfers, read:balances, etc.
- Fine-grained per-partner access control
- Aligns with PartnerGrant, PartnerApiCredential models in Prisma
- Partners can: check balances, initiate transfers (with user consent), query status

**Decision 28: Attorneys/Sanctions (CONFIRMED: Remove entirely)**
- Remove from sidebar completely
- Code is stubs with lint errors (dev-azim's code)
- Re-add when compliance module (MOD-19) is built properly
- No feature flag needed

**Decision 29: Fiat Display (CONFIRMED: Toggle GX-only or GX+Fiat)**
- App setting toggle: OFF = GX/Qirat only (current). ON = dual display (GX primary + fiat below/beside)
- Simple on/off toggle in app settings
- When ON: AmountInput shows GX field + fiat field, balance shows GX + (fiat equivalent)

**Decision 30: KYC Document Review (CONFIRMED: Inline preview)**
- Documents displayed directly in review page using img tags with presigned URLs
- Admin sees ID photo, proof of address, selfie without leaving the page
- Faster review workflow — no tab switching

**Decision 31: Quick Actions Master List (CONFIRMED: Expand to 8-10)**
- Master list: Send, Receive, QR Pay, Velocity Tax, Trust Network, Contacts, Messages, Settings, Transactions, Analytics, KYC Status, Institution Create
- User picks 4 for their dashboard
- Stored: localStorage + backend sync (Decision 14)

**Decision 32: Shared UI Library (CONFIRMED: Keep separate)**
- Each app maintains its own shadcn/ui components
- No shared package — avoids cross-repo dependency complexity
- Apps evolve independently
- Duplication is acceptable for stability

**Decision 33: Superowner SSO Exception (CONFIRMED: Dual login for superowner)**
- Superowner keeps separate command-center login (legacy credentials)
- All OTHER admins use SSO via user-wallet
- Minimal migration — superowner is a bootstrap exception
- Superowner can optionally link to participant profile later

**Decision 34: Institution Wizard Auto-Save (CONFIRMED)**
- Each completed step saves to backend (InstitutionOnboardingSession table in Prisma)
- User can close browser and resume later
- Same pattern as KYC wizard

**Decision 35: Admin Actions Expanded (CONFIRMED: Add more)**
- Keep: Suspend, Unsuspend, Lock, Unlock, Ban
- Add: Force Password Reset, Force 2FA Reset, Trigger KYC Re-verification, Send Notification, Export User Data, Revoke All Sessions
- Full admin toolkit on user detail page

**Decision 36: Dark Mode (CONFIRMED: Add in MOD-7)**
- Add dark mode to user-wallet as part of MOD-7 dashboard polish
- command-center already has light/dark toggle
- IAC inherits command-center pattern
- Consistent dark mode across all apps

**Decision 37: Hub Content (CONFIRMED: Context cards + notification badges)**
- Hub shows only context selection cards (Wallets + Administration)
- Each card has notification count badge (e.g., "3 pending approvals" on Admin, "2 new messages" on Wallet)
- Clean + informative without clutter
- Requires backend to expose per-context notification counts

**Decision 38: Send to Non-Registered (CONFIRMED: Registered only)**
- Only send to existing on-chain participants with fabricUserId
- No invite/claim links for now
- Simple, secure, current behavior

**Decision 39: Error State Standardization (CONFIRMED: In MOD-7)**
- Create reusable ErrorBoundary, EmptyState, LoadingState components
- Apply in user-wallet during MOD-7, then propagate pattern to command-center
- Consistent error UX across all pages

**Decision 40: Testing Strategy (CONFIRMED: Test as you go)**
- Each module writes tests for critical paths during implementation
- MOD-14 focuses on cross-module E2E integration tests
- Progressive testing — better quality, bugs caught early

**Decision 41: Token Relay (CONFIRMED: OAuth2 Authorization Code Flow)**
- Full OAuth2 auth code flow for cross-app authentication
- user-wallet redirects to command-center with auth code
- command-center exchanges code for token via svc-auth backend
- Most secure, most standard
- svc-auth needs: /authorize, /token, /userinfo endpoints

**Decision 42: Pagination (CONFIRMED: 25 default, selectable)**
- Default 25 rows per page
- Dropdown selector: 10, 25, 50, 100
- Page indicator: "Page X of Y"
- Standard enterprise table pattern

**Decision 43: Roadmap Scope (CONFIRMED: Finalize, add later)**
- Current 22 modules (MOD-4 through MOD-22) cover known scope
- Additional modules can be added during spec writing as gaps are discovered

---

## Audit Findings Cache

### Auth Architecture (3 Separate Systems)
- **user-wallet**: NextAuth.js + JWT session cookies + svc-auth
- **command-center**: Zustand + localStorage + svc-admin auth
- **institutional-admin-center**: Zustand + localStorage + JWT handoff from wallet
- Token handoff exists: wallet → IAC (URL hash fragment). No reverse direction.
- AdminUser.profileId links to UserProfile (FK in Prisma schema)
- No unified `/contexts` endpoint exists yet
- No session bridge between apps

### Send Page (Production-Ready)
- 4-step TransferFlow: Compose → Review → Processing → Receipt
- Unit system: Qirat internally (1 GX = 1,000,000 Qirat), GX for display
- Recipient resolution: debounced 500ms lookup, visual feedback
- Fee estimation: 0.1% for ≥3 GX, free below
- Blockchain polling: 2s intervals, 60s timeout
- Genesis currency data: 150+ currencies in `the-website/src/lib/genesisCurrencyData.ts`
- NO fiat conversion UI currently on send page

### Backend User Management Endpoints (svc-admin)
- 21 endpoints: list, get, approve, deny, freeze, unfreeze, suspend, unsuspend, lock, unlock, ban, getUserHistory, searchUsers, exportUsers, getPendingOnchain, listFrozen, batchRegister, getProcessing, getFailed, retryFailed, resetFailedOnchain

### Current Users Table State
- 6 columns: Name+Email, FabricID, Status(badge), Country, Created, Actions(3-dot)
- 6 tabs: Active, Pending Onchain, Suspended, Locked, Deactivated, Banned
- 1 filter: search input (name/email/fabricId)
- Row click exists but competes with 3-dot menu
- Pagination: Previous/Next

### User Detail Page State
- 10 tabs, 4 with real data (Profile, Documents, Activity, Sessions)
- Sidebar: UserActionsPanel with status card + action buttons
- 5 action dialogs: Suspend, Unsuspend, Lock, Unlock, Ban

---

## Pending Questions

### MOD-4 — Round 2 (Next)
- Server-side vs client-side sorting/filtering?
- Pagination: page size? current page + size selector?
- Bulk actions: multi-select rows for batch operations?
- Export: keep CSV export? add more formats?
- User detail page: redesign layout or keep current tabs+sidebar?
- Actions panel: keep suspend/lock/ban in sidebar or move elsewhere?

### MOD-5 — Pending
- Admin roles: any new roles beyond SUPER_ADMIN, ADMIN, MODERATOR, DEVELOPER, AUDITOR?
- Permission granularity: page-level or feature-level?
- Should existing standalone admins be migrated to linked-participant model?

### MOD-6 — Pending
- Access Panel naming: "Access Panel", "Context Selector", "Hub"?
- Should command-center accept token handoff (like IAC does)?
- Single sign-on aspiration or keep separate auth stores?

### MOD-8 — Pending
- Genesis gold price: exact value to use? (the-website has per-currency rates)
- Should fiat conversion show on receive page too?
- Should transaction history show fiat equivalents?

### All Modules — Pending
- Mobile responsiveness requirements?
- Accessibility (WCAG) level?
- i18n/localization?
- Dark mode support across all apps?
