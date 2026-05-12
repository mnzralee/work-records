# Building a Module End to End: The MOD-11 Case Study

> **A sit-down walkthrough for the engineer who is about to ship their first GX Protocol module.**
>
> Author voice: senior engineer teaching the new hire. We will not just describe what the codebase looks like. We will follow the development lifecycle of a real module (MOD-11, the Partner / FSP module) from blank slate to "feature-complete on DevNet, gated for MainNet", call out every decision that mattered, every trap that bit us, and every pattern you will reuse on your next module.
>
> **Last Updated**: 2026-05-04
> **Length**: long. Read it in two sessions. Treat it like an apprenticeship.
> **Prerequisites**: you have read [LECTURE-02-INTRODUCTION-TO-GX-PROTOCOL.md](./LECTURE-02-INTRODUCTION-TO-GX-PROTOCOL.md), have shell access on VPS3, and have run a service locally at least once.

---

## Table of Contents

1. [Foreword: How to read this guide](#foreword-how-to-read-this-guide)
2. [The Map of the Territory](#1-the-map-of-the-territory)
3. [What "a module" actually is in GX Protocol](#2-what-a-module-actually-is-in-gx-protocol)
4. [The full lifecycle in one picture](#3-the-full-lifecycle-in-one-picture)
5. [Phase 0: Mandate, scope, and the locked-decisions register](#phase-0-mandate-scope-and-the-locked-decisions-register)
6. [Phase 1: Specs that survive contact with reality](#phase-1-specs-that-survive-contact-with-reality)
7. [Phase 2: Smart contract first (on-chain is the source of truth)](#phase-2-smart-contract-first)
8. [Phase 3: Database schema and Prisma sanity](#phase-3-database-schema-and-prisma-sanity)
9. [Phase 4: Backend service (Clean Architecture in practice)](#phase-4-backend-service-clean-architecture-in-practice)
10. [Phase 5: The CQRS bridge (outbox + projector)](#phase-5-the-cqrs-bridge-outbox--projector)
11. [Phase 6: Frontend (Next.js App Router, scoped JWT, design system)](#phase-6-frontend-nextjs-app-router-scoped-jwt-design-system)
12. [Phase 7: Containerise and deploy (Docker, K3s, ingress, health)](#phase-7-containerise-and-deploy-docker-k3s-ingress-health)
13. [Phase 8: Observability, security, audit](#phase-8-observability-security-audit)
14. [Phase 9: Testing pyramid in practice](#phase-9-testing-pyramid-in-practice)
15. [Phase 10: Commit discipline, work records, and PR hygiene](#phase-10-commit-discipline-work-records-and-pr-hygiene)
16. [Phase 11: The Super-Engineer Review Board and audit remediation](#phase-11-the-super-engineer-review-board-and-audit-remediation)
17. [The Gotcha Registry (real bugs, real fixes)](#the-gotcha-registry-real-bugs-real-fixes)
18. [What is still incomplete on MOD-11](#what-is-still-incomplete-on-mod-11)
19. [Exercise: develop MOD-12 (Messaging) on your own](#exercise-develop-mod-12-messaging-on-your-own)
20. [Required reading and lecture cross-references](#required-reading-and-lecture-cross-references)

---

## Foreword: How to read this guide

You are not going to memorise this. You are going to use it the way I used the senior engineers around me when I joined a real codebase: keep it open in a tab, work through one phase, then come back when the next phase confuses you.

I am writing this in the voice of someone sitting at a desk next to you. I will say "we" because we are building this together. I will name files with their real paths so you can open them in your editor. When I quote code I am quoting code that actually shipped, sometimes in its broken-first form and then in its fixed form, because the fix is where the lesson lives.

A few ground rules before we start:

1. **Open the actual files as you read.** Do not just read the snippets here. Open [backend-core/apps/svc-organization/src/application/use-cases/partner/](../../backend-core/apps/svc-organization/src/application/use-cases/partner/) in another window and follow along.
2. **The lifecycle is not linear.** I am going to present it phase by phase, but in real life you will jump back and forth. You will write a test, find a missing column, edit the schema, regenerate Prisma, edit the use-case, write a new outbox handler, and only then come back to the test. That is normal.
3. **Read the "why" lines.** Every section has a paragraph that begins "Why this way". That paragraph is the actual lesson. Everything else is just code.
4. **MOD-11 is not 100% complete.** It is feature-complete on DevNet. There are 13 explicit MainNet-gate items deferred by user decision. We will list them at the end so you know exactly what "done" means in this codebase.

Let us start by zooming way out.

---

## 1. The Map of the Territory

Before you write a line of code for a new module, you should be able to draw this from memory.

```
                                ┌─────────────────────────────┐
                                │       PARTICIPANTS           │
                                │  (people, institutions, FSPs)│
                                └──────────┬──────────────────┘
                                           │ HTTPS
                                           ▼
            ┌────────────────────────────────────────────────────────────────┐
            │                  FRONTEND APPS (Next.js 16)                    │
            │   user-wallet:3000  command-center:3001  IAC:3002  PAC:3003    │
            └─────────┬─────────────┬───────────────┬───────────────┬────────┘
                      │             │               │               │
                      ▼             ▼               ▼               ▼
            ┌────────────────────────────────────────────────────────────────┐
            │              BACKEND MICROSERVICES (Express.js)                 │
            │   svc-auth (3041)  svc-kyc (3042)  svc-admin (3040)             │
            │   svc-wallet (3043)  svc-tokenomics (3044)  svc-government...   │
            │   svc-organization (3046)  svc-partner (3056)  svc-trust...     │
            │   ...                                                           │
            └──────┬────────────────────────────┬────────────────────────────┘
                   │ Prisma                     │ writes outbox commands
                   ▼                            ▼
            ┌──────────────┐            ┌──────────────────┐
            │  PostgreSQL  │            │ outbox_commands  │
            │ (read models │ ◄──────────│  table (PENDING) │
            │  + audit)    │            └────────┬─────────┘
            └──────────────┘                     │
                   ▲                             │ polled
                   │                             ▼
                   │                  ┌────────────────────────┐
                   │                  │  outbox-submitter      │
                   │                  │  (worker)              │
                   │                  └──────────┬─────────────┘
                   │                             │ ethers.js
                   │                             ▼
                   │                  ┌────────────────────────┐
                   │                  │  Besu QBFT chain       │
                   │                  │  (3 validators)         │
                   │                  │  GX Diamond at         │
                   │                  │  0xe7f1...0512          │
                   │                  └──────────┬─────────────┘
                   │                             │ events
                   │                             ▼
                   │                  ┌────────────────────────┐
                   └──────────────────┤  projector worker      │
                                      │  (read-model writer)   │
                                      └────────────────────────┘
```

That picture has six pieces. Memorise their roles, because every module you build slots into this map:

| Piece | Role | Where it lives |
|-------|------|----------------|
| Frontend app | UI, validation, calls HTTP API | `user-wallet/`, `command-center/`, `institutional-admin-center/`, `partner-admin-center/` |
| Backend microservice | HTTP entry, business logic, writes outbox | `backend-core/apps/svc-*/` |
| PostgreSQL read model | Source of truth for queries (NOT writes to chain) | managed by Prisma; namespace `gx-data` |
| Outbox table | Durable queue between services and chain | `outbox_commands` table, written by services in same `$transaction` as read-model |
| Outbox-submitter | Polls outbox, submits to Besu, marks SUBMITTED | `backend-core/workers/outbox-submitter/` |
| Besu chain (GX Diamond) | On-chain truth (balances, partner status, governance) | `blockchain-besu/contracts/`, deployed in `gx-data` |
| Projector | Reads chain events, updates read models | `backend-core/workers/projector/` |

A new module almost always touches all six. Let me prove it: MOD-11 ("partners") touched five of the six (it did not need a new frontend app because Partner Admin Center already existed by Session 115; it added 13 routes to it). MOD-9 ("government treasury") touched all six. MOD-3 ("identity / KYC") touched five (no Besu facet originally, KYC doesn't go on chain).

Hold this picture in your head for the rest of the guide.

---

## 2. What "a module" actually is in GX Protocol

A "module" is a vertical slice that delivers one coherent capability of the protocol to a participant.

Compare three modules:

- **MOD-1 (Token Economics)** is the issuance mechanic of the GX unit itself. It is mostly chain logic plus tokenomics parameters.
- **MOD-7 (Wallet Dashboard)** is mostly frontend on top of existing balance and transfer endpoints.
- **MOD-11 (Partner / FSP)** is the entire partner ecosystem. It is everything: chain functions for the partner registry, off-chain use-cases for onboarding and settlement, projector handlers for partner events, the Partner Admin Center frontend, KYB documentation, the three-eyes approval flow, and the validator bond logic.

The pattern across all of them is the same:

> A module is a **bounded context** plus the **infrastructure to operate it**.

Bounded context comes from Domain-Driven Design. It means "the part of the model where the words mean one specific thing". In MOD-11, "partner" has a precise meaning: an organisation that has been licensed under one of seven categories (FSP, PMS, INS, MKT, CRD, VAL, TIP), with a profit-share ratio in basis points, with a multi-signature approval requirement on settlement. That definition does not bleed into other modules. It is bounded.

The "infrastructure to operate it" part is the boring-but-mandatory part: K8s manifests, observability, runbooks, audit logs, design-system compliance, work records. New engineers want to skip this. Senior engineers know this is where 40% of the wall-clock time goes.

A useful test: if a module has a one-page spec describing only the happy path, it is not a real module yet. Real modules have:

1. A **scope** statement (what it does, what it does not do)
2. An **architectural decision register** (locked decisions, see Phase 0)
3. An **on-chain surface** (functions, events, storage layout) where applicable
4. An **off-chain surface** (HTTP routes, use-cases, repositories)
5. A **read-model surface** (Prisma models, indices, expiry sweepers)
6. A **CQRS bridge** (outbox commands plus projector handlers)
7. A **frontend surface** (routes, components, design tokens, scoped-auth gates)
8. A **deployment surface** (Dockerfile, K8s manifest, ingress rule)
9. A **test pyramid** (unit, integration, E2E, load)
10. A **compliance surface** (security threats reviewed, audit findings closed, PII handling)

Open the MOD-11 implementation plan and you will see all ten:
[docs-dev-manazir/specs/SPEC-MOD-11-IMPLEMENTATION-PLAN-V1.md](../specs/SPEC-MOD-11-IMPLEMENTATION-PLAN-V1.md).

When you propose a new module the question I will ask in review is: "Where is each of those ten?" If you cannot answer, your spec is incomplete.

---

## 3. The full lifecycle in one picture

```
   PHASE 0: MANDATE                  PHASE 6: FRONTEND
   - business problem                - routes
   - locked decisions register       - components
                                     - scoped-auth
   PHASE 1: SPECS                    - design tokens
   - implementation plan
   - bridge spec (CQRS)              PHASE 7: DEPLOY
   - frontend spec                   - Dockerfile
                                     - K8s manifest
   PHASE 2: CHAIN                    - ingress
   - facet (Solidity)                - secrets
   - storage layout
   - hardhat tests                   PHASE 8: OBSERVE
                                     - OTel spans
   PHASE 3: SCHEMA                   - Prom metrics
   - prisma models                   - SLO dashboards
   - migrations                      - log labels
   - indices
                                     PHASE 9: TEST
   PHASE 4: BACKEND                  - unit
   - domain entities                 - integration
   - use-cases                       - E2E
   - repositories                    - load (k6)
   - HTTP routes                     - chaos
   - DI container
                                     PHASE 10: COMMIT
   PHASE 5: CQRS BRIDGE              - file-by-file
   - outbox commands                 - work record
   - projector handlers              - PR
   - idempotency
                                     PHASE 11: REVIEW
                                     - super-engineer review
                                     - audit remediation
                                     - scorecard
```

The arrow is suggestive, not strict. You will iterate. But this is the order you should *plan* in. Trying to write a frontend before the backend exists is how juniors burn three days drawing fake data into shadcn cards. Trying to write a backend before the chain function exists is how juniors define the wrong outbox payload shape.

We will now walk through every phase, using MOD-11 as the worked example.

---

## Phase 0: Mandate, scope, and the locked-decisions register

### 0.1 What problem are we solving?

The mandate for MOD-11 was a one-paragraph statement from the protocol governance group, summarised:

> The protocol needs to onboard external organisations as partners across seven categories. Partners receive a license, deposit stake, can have their stake slashed if they misbehave, and (for FSPs) settle a quarterly profit share. All slashes and settlements need three independent signatures from authorised admins. The entire lifecycle from Expression of Interest to ACTIVATED to SETTLED must be observable from the Partner Admin Center.

Read that paragraph until it tells you the whole shape of the module. Notice what it locks in:

- **Seven categories**: FSP, PMS, INS, MKT, CRD, VAL, TIP
- **State machine**: EoI -> DUE_DILIGENCE -> INTEGRATION_TEST -> ACTIVATED -> (SUSPENDED|REVOKED), with parallel STAKED and SETTLED states
- **Three-eyes flow**: initiate -> approve -> execute, with three different humans
- **Frontend surface**: Partner Admin Center

Notice what it does *not* tell you:

- How to model profit-share ratio (basis points? percentage? decimal?)
- Whether stake slashing is time-locked (front-running? grace period?)
- What happens if an admin gets promoted between initiate and execute
- How to bound storage growth from pending approvals

Those are the questions that go into the **architectural decisions register** (also called the "locked decisions" or "L# decisions"). Here is the actual MOD-11 register, all 13 items locked before implementation started. Read the full table in [SPEC-MOD-11-IMPLEMENTATION-PLAN-V1.md](../specs/SPEC-MOD-11-IMPLEMENTATION-PLAN-V1.md). The compressed version:

| L# | Decision | Why we made it |
|----|----------|----------------|
| L1 | Partner code is a bounded context inside `svc-organization`, not a new microservice | Partner shares the organisation database boundary; spinning up a new container would have been pure overhead. (Note: in Session 120 we did extract `svc-partner` as the 22nd service after `svc-organization` got too large. The bounded context stayed; the deployment unit changed. That is fine.) |
| L2 | All on-chain wiring follows [SPEC-PARTNER-ONCHAIN-BRIDGE-V1.md](../specs/SPEC-PARTNER-ONCHAIN-BRIDGE-V1.md) | Single source of truth for the use-case to outbox to facet to projector path |
| L3 | `svc-trust` real implementation (7-factor algorithm) replaces the constant-zero stub before MOD-11.1 starts | The stub returned zero for everyone; that is a credit-risk failure mode |
| L4 | Three-eyes snapshot captures the authorised-admin set at *initiate* time, not at execute time | Otherwise a new admin promoted between initiate and execute could approve a settlement they had no business approving |
| L5 | Multi-sig enforces initiator different from recipient when recipient is a stakeholder | Mitigates the FSP self-deal scenario (admin pays themselves) |
| L6 | Validator bond withdrawals are time-locked (7 days) | Mitigates bond front-running on slashing: validator sees the slash coming, withdraws first |
| L7 | Pending-tx storage capped at 100 per organisation; `pruneExpiredTxs` is callable | Mitigates a storage-bloat denial-of-service vector |
| L8 | All MOD-11 frontend surfaces follow [DESIGN-SYSTEM-CANON.md](../specs/DESIGN-SYSTEM-CANON.md) v1.0 | Visual consistency from day one beats fixing it later |
| L9 | Domain language uses "participant", "unit", `formatGxAmount` | ESLint enforces; never hardcode a hex colour or write "user" |
| L10 | New code uses `generateParticipantKey` not `generateFabricUserId` | We are migrating off Fabric to Besu; the deprecated name fails CI |
| L11 | All outbox writes wrapped in `prisma.$transaction` with their read-model write | Atomic CQRS, ESLint at error level |
| L12 | Off-chain `expiresAt` mirrors on-chain `LibPartner.PENDING_EXPIRY_SECONDS = 7 days` | Scheduler runs every 15 min |
| L13 | Scoped JWT (30 min TTL, PIN/2FA gated) required for partner-admin three-eyes endpoints | Same gate that government-treasury and institution use |

### 0.2 Why the locked decisions matter

A locked decision is one that you do *not* relitigate during implementation. If your use-case author and your frontend author disagree about whether profit-share is in basis points or a percentage, you do not negotiate in PR comments. You go to the L# decision and apply it. If there is no L# decision, that is a missing locked decision and you go back to the mandate group.

I cannot overstate how much time this saves. On MOD-11 we touched roughly 50 use-cases across 4 sub-phases. Without L1 to L13, every PR would have re-debated profit-share precision, three-eyes ordering, expiry semantics. With L1 to L13, every PR was either compliant (merge) or non-compliant (one-line comment: "violates L11, please wrap in $transaction"). That is your goal.

### 0.3 Your move

When you start a new module, sit down for two hours and write the mandate paragraph plus the L# table. Do *not* start coding. Open the MOD-11 plan as your template:

```bash
cp docs-dev-manazir/specs/SPEC-MOD-11-IMPLEMENTATION-PLAN-V1.md \
   docs-dev-manazir/specs/SPEC-MOD-XX-YOUR-MODULE-PLAN-V1.md
```

Fill it in. Send it to a senior for review. Only when L# decisions are locked do you go to Phase 1.

---

## Phase 1: Specs that survive contact with reality

Specs in this codebase are not bureaucratic theatre. They are the contract between the senior who decided the architecture and the junior (or AI agent) who implements it. A good spec gets re-read 30 times during implementation. A bad spec gets ignored after week one.

For MOD-11 the spec stack was:

| Spec file | Audience | Purpose |
|-----------|----------|---------|
| [SPEC-MOD-11-IMPLEMENTATION-PLAN-V1.md](../specs/SPEC-MOD-11-IMPLEMENTATION-PLAN-V1.md) | All implementers | Phasing (MOD-11.1 to MOD-11.6), L# decisions, acceptance criteria |
| [SPEC-PARTNER-ONCHAIN-BRIDGE-V1.md](../specs/SPEC-PARTNER-ONCHAIN-BRIDGE-V1.md) | Backend + chaincode | Use-case to outbox to facet to projector mapping; defect register (D1 to D7) |
| [SPEC-MOD-11-PARTNER-API-BACKEND-V1.md](../specs/SPEC-MOD-11-PARTNER-API-BACKEND-V1.md) | Backend implementer | HTTP routes, request/response shapes, error codes |
| [SPEC-MOD-11-PARTNER-ADMIN-CENTER-V1.md](../specs/SPEC-MOD-11-PARTNER-ADMIN-CENTER-V1.md) | Frontend implementer | Routes, components, scoped-auth gates, design tokens |
| [SPEC-MOD-11-HUB-EXTENSION-V1.md](../specs/SPEC-MOD-11-HUB-EXTENSION-V1.md) | Frontend implementer | How partner role plugs into the existing Hub context-switcher |
| [SPEC-MOD-11-SESSION-PLAN-V1.md](../specs/SPEC-MOD-11-SESSION-PLAN-V1.md) | Orchestrator | Per-session track dispatch (parallel agent pattern) |

A good spec has these sections:

1. **Scope and non-scope** (what is in, what is explicitly out)
2. **Locked decisions** (the L# table from Phase 0, repeated for context)
3. **Phasing** (sub-phases each producing 4-8 PRs, max ~800 LOC per PR)
4. **Acceptance criteria** (concrete tests; for example "register a partner end-to-end on DevNet in 90 seconds")
5. **Defect register** (D# items: known issues to fix during this module)
6. **Risk register** (what can still go wrong, mitigations)
7. **Carry-forward** (what is intentionally deferred)

Notice the defect register is *part of the spec*. Defects are not surprises that emerge during implementation. They are pre-known issues we already know we need to fix and we name them D1, D2, D3 so that PRs say "fixes D3" and we can grep for closure.

For MOD-11 the bridge defects were:

- **D1**: Three-eyes initiator-set snapshot was not being captured at initiate time
- **D2**: Outbox `service` field was inconsistent (some commands wrote `'svc-organization'`, others wrote `'svc-partner'`)
- **D3**: Some producer use-cases wrote outbox *outside* `prisma.$transaction`, risking diverged state on crash
- **D4**: Off-chain expiry sweep ran every 24h instead of 15 min, creating a 23h window where a stale approval was still acceptable
- **D5**: Profit-report submission did not check on-chain partner balance before writing the outbox command, creating a reconciliation nightmare when balance was insufficient
- **D6**: Race condition where `registerPartner` outbox command was followed immediately by `fundPartnerBalance`; if the projector had not yet set `Partner.fabricPartnerId`, the second command would resolve to null
- **D7**: Pending-transaction storage was unbounded, creating a DoS vector

We *named these* before implementation started. Then every PR closed one or two of them. By Session 119 the D-register was empty.

> **Why this way**: pre-naming defects converts unknowns into knowns. A junior engineer who has never seen this codebase can pick up a D-item and fix it. An AI agent can be told "fix D3" and produce a tightly-scoped change. Compare that to "the CQRS pipeline has some atomicity issues, please look into it", which is unworkable.

---

## Phase 2: Smart contract first

> **Iron rule**: when a module touches the chain, you write the chain code first. The chain function is the source of truth. Off-chain code mirrors it; never the reverse.

MOD-11's chain surface is `PartnerFacet.sol`, an EIP-2535 Diamond facet:
[blockchain-besu/contracts/facets/PartnerFacet.sol](../../blockchain-besu/contracts/facets/PartnerFacet.sol).

It is 1,397 lines and exposes 19 external functions and 17 events. If those numbers feel intimidating, remember that 8 of them existed before MOD-11 (registerPartner, activatePartnerLicense, suspendPartner, etc.). MOD-11 added 11.

### 2.1 What is a Diamond facet?

If you have a Fabric background, the mental model is:

> *Diamond proxy* is to *facets* as the *Fabric peer* is to *chaincode contracts*.

A Diamond proxy is a single contract address that delegates to many implementation contracts called facets. Each facet contributes a set of function selectors. The Diamond keeps a routing table: "selector 0xabcd1234 -> PartnerFacet implementation address". When you call `diamond.registerPartner(...)`, it `delegatecall`s to PartnerFacet, which executes against the Diamond's storage.

This is EIP-2535. Read [LECTURE-11-SMART-CONTRACT-ARCHITECTURE.md](./LECTURE-11-SMART-CONTRACT-ARCHITECTURE.md) and [BESU-SOLIDITY-DIAMOND-LECTURE.md](./BESU-SOLIDITY-DIAMOND-LECTURE.md) before you write a facet. I will not duplicate that material here. I will tell you what you do *with* a facet during a module.

### 2.2 Designing the facet surface

For MOD-11 the design question was: which on-chain functions does a partner lifecycle need?

| Lifecycle stage | Needs to be on chain? | Why |
|-----------------|----------------------|-----|
| EoI (Expression of Interest) | No | This is a paperwork stage, off-chain DB only |
| Due diligence review | No | Internal admin process, off-chain |
| Approve license | **Yes** | This is the moment the partner becomes a registered on-chain entity |
| Activate | **Yes** | License goes from REGISTERED to ACTIVE on chain |
| Suspend / revoke | **Yes** | Operational state matters on chain (other facets check it) |
| Stake deposit | **Yes** | Custody of GX units, must be on chain |
| Slash | **Yes** | Three-eyes; must be enforced on chain |
| Profit report submission | **Hash on chain** | The full report is too large; we hash it on chain and store the hash |
| Profit settlement | **Yes** | Three-eyes; transfer of funds |
| Reverse settlement | **Yes** | Audit trail, GX units move back |
| Configure staking tiers | **Yes** | Determines stake amounts; must be on chain |

That table gave us our 19 functions. Here is the slimmed-down list with the three-eyes ones flagged:

| # | Function | 3-eyes? | Notes |
|---|----------|---------|-------|
| 1 | `registerPartner` | No | One-shot |
| 2 | `activatePartnerLicense` | No | After due diligence |
| 3 | `suspendPartner` | No | Single admin |
| 4 | `initiateRevocation` | No | Multi-step; pending tx, not three-eyes |
| 5 | `completeRevocation` | No | |
| 6 | `reissueLicense` | No | |
| 7 | `adminFundPartnerBalance` | No | Treasury-funded |
| 8 | `stakeForPartner` | No | Partner deposits |
| 9 | `initiateSlash` | 1st eye | Returns slashId |
| 10 | `approveSlash` | 2nd eye | caller != initiator |
| 11 | `executeSlash` | 3rd eye | caller != initiator AND != approver |
| 12 | `unstake` | No | Time-locked (L6) |
| 13 | `submitProfitReportHash` | No | |
| 14 | `validateProfitReport` | No | Single admin |
| 15 | `initiateSettlement` | 1st eye | |
| 16 | `approveSettlement` | 2nd eye | |
| 17 | `executeSettlement` | 3rd eye | Transfers GX |
| 18 | `reverseProfitShare` | No | Audit-trail; reverses a settled share |
| 19 | `configureStaking` | No | Tier configuration |

### 2.3 Storage layout: the Diamond storage trap

Each facet must declare its storage in a unique slot computed by `keccak256("gx.protocol.storage.partner")`. This is non-negotiable. If two facets share a storage slot, they will overwrite each other and you will spend three days debugging "why does my balance keep resetting".

The MOD-11 storage layout in `LibPartner.sol`:

```solidity
struct Partner {
  string fabricPartnerId;
  uint8 category;
  uint8 marketScope;
  uint16 profitShareRatioBps;   // 4500..7000 (45% to 70% GX share)
  uint40 registeredAt;
  uint256 balanceQirat;
}

struct PartnerStake {
  uint256 amountQirat;
  uint256 slashedAmountQirat;
  uint40 stakedAt;
  bool withdrawn;
}

struct PendingAction {
  uint8 phase;            // 0=initiated, 1=approved, 2=executed
  uint40 initiatedAt;
  uint40 expiresAt;       // initiatedAt + 7 days
  string initiatorAddress;
}

struct StakingConfig {
  uint8 modelType;
  uint256 baseRequiredAmount;
  uint16 tierCount;
  uint256[] tierThresholds;
  uint256[] tierAmounts;
  uint16 enabledTiers;
  uint256 maxStakePrincipal;
}
```

A few details that matter:

- `profitShareRatioBps` is `uint16` because `4500..7000` fits in 16 bits and we need to validate the range on every write. Junior engineers waste storage with `uint256` here.
- All amounts are `Qirat` (the GX subunit). 1 GX = 10,000 Qirat. We never use floats. Read [LECTURE-13-GENESIS-DISTRIBUTION-TOKENOMICS.md](./LECTURE-13-GENESIS-DISTRIBUTION-TOKENOMICS.md) for why.
- `PendingAction.expiresAt = initiatedAt + 7 days` is the on-chain mirror of L12. The off-chain sweeper has the same number.
- `string fabricPartnerId` is the legacy field name. New code calls this `participantKey` (L10). Solidity strings on chain are expensive but partners need a human-readable id, so we eat the cost.

> **Why this way**: design the storage struct *before* you write any function. The storage is the slow-changing part of the contract. Functions are cheap to add later. Storage layout is hard to migrate. Get it right on day one.

### 2.4 Writing the facet, then the tests

Functions are straightforward Solidity. The pattern for a three-eyes function:

```solidity
function approveSlash(string calldata slashId) external {
  LibPartner.Storage storage s = LibPartner.layout();
  LibPartner.PendingAction storage action = s.slashes[slashId];

  // 1. State checks
  require(action.phase == 0, "PartnerFacet: not in INITIATED phase");
  require(block.timestamp <= action.expiresAt, "PartnerFacet: expired");

  // 2. Caller-distinct check (the heart of three-eyes)
  require(
    keccak256(bytes(LibAccessControl.callerKey())) !=
    keccak256(bytes(action.initiatorAddress)),
    "PartnerFacet: approver must differ from initiator"
  );

  // 3. State transition
  action.phase = 1;
  action.approverAddress = LibAccessControl.callerKey();

  // 4. Emit (the projector needs this)
  emit SlashApproved(slashId, action.approverAddress, block.timestamp);
}
```

Do not be clever here. Read the action, check state, check caller distinct, mutate state, emit. The tests for this function should be:

1. Happy path: initiator A initiates, approver B approves, state goes to phase 1
2. Self-deal: A tries to approve their own initiate, reverts
3. Expired: time travel past `expiresAt`, B tries to approve, reverts
4. Wrong phase: B tries to approve when phase is already 2, reverts

Hardhat tests for MOD-11 ended up at 72+, exceeding the planned 50. Examples:
[blockchain-besu/test/PartnerFacet.test.ts](../../blockchain-besu/test/PartnerFacet.test.ts) (41 tests),
[blockchain-besu/test/ValidatorSlashThreeEyes.spec.ts](../../blockchain-besu/test/ValidatorSlashThreeEyes.spec.ts),
[blockchain-besu/test/ValidatorBondTimelock.spec.ts](../../blockchain-besu/test/ValidatorBondTimelock.spec.ts),
[blockchain-besu/test/SelfDealRevert.spec.ts](../../blockchain-besu/test/SelfDealRevert.spec.ts),
[blockchain-besu/test/PendingTxStorageGrowth.spec.ts](../../blockchain-besu/test/PendingTxStorageGrowth.spec.ts),
[blockchain-besu/test/SelectorCollision.test.ts](../../blockchain-besu/test/SelectorCollision.test.ts).

### 2.5 The Selector Collision test (this saved us)

EIP-2535 routes by 4-byte function selector. If two facets export functions whose selectors collide, the Diamond deploy will fail at cut time, but only if you have a test for it. Otherwise you will deploy and silently overwrite a function. We learned this in Session 121:

```typescript
// blockchain-besu/test/SelectorCollision.test.ts
it("no two facets share a selector", async () => {
  const allSelectors = new Map<string, string>();
  for (const facet of facets) {
    for (const fn of facet.functions) {
      const sel = ethers.utils.id(fn.signature).slice(0, 10);
      if (allSelectors.has(sel)) {
        throw new Error(
          `Selector collision: ${facet.name}.${fn.name} ` +
          `collides with ${allSelectors.get(sel)}`
        );
      }
      allSelectors.set(sel, `${facet.name}.${fn.name}`);
    }
  }
});
```

Add this test before you ship any new facet. It is cheap, it catches a class of bugs that is otherwise invisible.

### 2.6 Compile, build the ABI, and pin the address

Once your facet compiles and tests pass:

```bash
cd blockchain-besu
npx hardhat compile
npx hardhat test
```

The ABI lands in `artifacts/contracts/facets/PartnerFacet.sol/PartnerFacet.json`. The backend depends on this ABI. **Stale ABIs are a class of bug.** Read the memory entry [project_besu_abi_stale.md](../../../.claude/projects/-home-dev-manazir-code-vault/memory/project_besu_abi_stale.md): if the Solidity changes and you do not rebuild the ABI, the backend will silently drop events whose signatures no longer match. Always rebuild ABIs after Solidity changes; CI enforces a hash check.

Deploy the facet with `hardhat-deploy` scripts. The Diamond on DevNet is at `0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512`. After cut, verify with `diamondLoupe.facets()` that your selectors are present.

---

## Phase 3: Database schema and Prisma sanity

The chain is the source of truth for state. The database is the source of truth for *queries*. They are not the same. You read from Postgres, you write through the outbox. Get this distinction cemented.

### 3.1 Modelling read-models

For MOD-11, the new Prisma models lived in [backend-core/db/prisma/schema.prisma](../../backend-core/db/prisma/schema.prisma) (and its sibling files in `db/prisma/schema/`):

- `Partner` (one row per registered partner, mirrors on-chain Partner struct)
- `PartnerApplication` (the EoI / due diligence / integration test workflow)
- `PartnerLicense` (the issued license document; multiple per partner over time)
- `PartnerStake` (mirrors on-chain stake state)
- `PartnerSlashApproval` (the off-chain three-eyes mirror with `authorisedAdminsSnapshot`)
- `PartnerProfitReport`
- `PartnerSettlementApproval`
- Plus extensions to `OrganizationV2` (link to partner), `OutboxCommand` (new command types in enum), `AuditLogV2` (partner event types).

Here is the `Partner` model exactly as it shipped:

```prisma
model Partner {
  id                    String      @id @default(cuid())
  organizationId        String
  organization          OrganizationV2 @relation("PartnerOrg", fields: [organizationId], references: [orgId])

  fabricPartnerId       String?     // Set by PartnerRegistered projector handler
  partnerName           String
  category              PartnerCategory  // FSP, PMS, INS, MKT, CRD, VAL, TIP
  profitShareRatioGxBps Int         // 4500..7000

  status                PartnerStatus    // PENDING_EoI -> ... -> ACTIVE
  balanceQirat          BigInt      @default(0)

  licensingTier         PartnerLicensingTier @default(SANDBOX)
  kybStatus             KybStatus            @default(PENDING)
  kybLastReviewedAt     DateTime?  @db.Timestamptz(3)

  payoutMethod          Json?
  notificationPrefs     Json?

  createdAt             DateTime    @default(now()) @db.Timestamptz(3)
  updatedAt             DateTime    @updatedAt @db.Timestamptz(3)

  applications          PartnerApplication[]
  licenses              PartnerLicense[]
  stakes                PartnerStake[]
  profitReports         PartnerProfitReport[]

  @@unique([organizationId])
  @@index([fabricPartnerId])
  @@index([status])
}
```

A few details that matter:

- `fabricPartnerId String?` is **nullable**. This is intentional. When a use-case writes the `REGISTER_PARTNER` outbox command, the projector has not yet seen the on-chain `PartnerRegistered` event, so the `fabricPartnerId` is unknown. The projector fills it in later. Your code must tolerate the nullable window. (This is defect D6.)
- All money fields are `BigInt`. **Never** use `Float` or `Decimal` for Qirat. Read the production-grade-code rule.
- `Timestamptz(3)` (millisecond precision, with timezone) is the standard. Naive `DateTime` will bite you in distributed systems.
- The `@@unique([organizationId])` constraint enforces "one partner per organisation". Without it, a single org could create two partner rows and you would have a multi-tenant nightmare.
- `@@index([status])` because the partner-list query filters by status; this index makes that query log-time.

### 3.2 The Prisma multi-file gotcha

This one bites everyone exactly once. The backend uses *two* schema setups:

- `db/prisma/schema.prisma` (single monolithic file, used by `prisma generate --schema=schema.prisma`)
- `db/prisma/schema/*.prisma` (multi-file, auto-discovered by `prisma db push` and `prisma migrate`)

If you add a model only to a multi-file schema and run `prisma generate --schema=schema.prisma`, the model will not be in the generated client. Your TypeScript will not see it. You will write `db.partner.findUnique(...)` and get a runtime "undefined" error.

The fix: **all models that need to be queried by code must live in `schema.prisma` itself**, not just the multi-file directory. Read [.claude/rules/prisma-schema.md](../../.claude/rules/prisma-schema.md). Read [LECTURE-08-PRISMA-DATABASE-DESIGN.md](./LECTURE-08-PRISMA-DATABASE-DESIGN.md).

### 3.3 Migrations

Once the schema compiles:

```bash
cd backend-core
npx prisma migrate dev --name add_partner_models
```

This creates a versioned migration in `db/prisma/migrations/`. Commit it. The CI applies migrations to the test database; the deploy applies them to DevNet through a Helm hook.

**Pitfall**: a migration listed in the `_prisma_migrations` table is *not* the same as a migration actually applied. We have been bitten by stale migration state on DevNet. Always verify a migration with:

```sql
\d "Partner"
SELECT column_name, data_type FROM information_schema.columns
  WHERE table_name = 'Partner';
```

If the column is missing from the live schema, no amount of "the migration is recorded" matters.

### 3.4 Indices and expiry sweepers

For every read-model you must answer two questions:

1. **What does the dominant query look like?** Add an index for it.
2. **Does this row expire?** If yes, write a sweeper.

For `PartnerSettlementApproval`, the dominant queries are "show me pending approvals for this partner" and "expire approvals older than 7 days". Hence:

```prisma
@@index([profitReportId])
@@index([expiresAt])
```

The sweeper is a scheduled use-case that runs every 15 minutes and bulk-updates `status = 'EXPIRED' WHERE expiresAt < NOW() AND status = 'PENDING'`. It is its own use-case, with its own test, in `backend-core/apps/svc-organization/src/application/use-cases/partner/scheduler/expire-pending-approvals.use-case.ts`.

This is the off-chain mirror of L12. If you forget the sweeper, you will accumulate stale approvals forever.

---

## Phase 4: Backend service (Clean Architecture in practice)

This is the largest phase by code volume. It is also the phase where Clean Architecture pays for itself. Read [LECTURE-21-CLEAN-ARCHITECTURE-HEXAGONAL-PATTERN.md](./LECTURE-21-CLEAN-ARCHITECTURE-HEXAGONAL-PATTERN.md) before continuing if you have not yet.

### 4.1 The four layers, in the order you write them

```
        ┌─────────────────────────────────────────────────┐
        │  INTERFACE  (Express controllers, routes, DTOs) │
        └─────────────────────────────────────────────────┘
                               │ depends on
                               ▼
        ┌─────────────────────────────────────────────────┐
        │  APPLICATION  (use-cases, ports, services)      │
        └─────────────────────────────────────────────────┘
                               │ depends on
                               ▼
        ┌─────────────────────────────────────────────────┐
        │  DOMAIN  (entities, value-objects, errors)      │
        └─────────────────────────────────────────────────┘
                               ▲
                               │ implements ports
        ┌─────────────────────────────────────────────────┐
        │  INFRASTRUCTURE (Prisma repos, HTTP clients)    │
        └─────────────────────────────────────────────────┘
```

Dependencies point *inward*. Domain knows nothing. Infrastructure knows about ports. Application knows about ports and domain. Interface knows about application and domain (so it can translate domain errors to HTTP status codes).

Write them inside-out:

1. Domain entities and value objects
2. Application ports (the interfaces that infrastructure will implement)
3. Application use-cases (the business logic)
4. Infrastructure (Prisma repositories, HTTP clients to other services)
5. Interface (Express controllers, routes, request DTOs, error mapper)
6. DI container (the "manual wiring" file that constructs everything)

Why this order? Because the inner layers compile and test in isolation. You can write the use-case and unit-test it with a mocked repository before any real database exists. By the time you reach the controller layer, the business logic is already proven. The controller is the dumb glue.

### 4.2 Sample domain entity

```typescript
// backend-core/apps/svc-organization/src/domain/value-objects/profit-share-ratio.vo.ts
export class ProfitShareRatio {
  private constructor(public readonly bps: number) {}

  static create(bps: number): ProfitShareRatio {
    if (!Number.isInteger(bps)) {
      throw new InvalidProfitShareRatioError("must be integer basis points");
    }
    if (bps < 4500 || bps > 7000) {
      throw new InvalidProfitShareRatioError(
        `must be between 4500 and 7000 bps, got ${bps}`
      );
    }
    return new ProfitShareRatio(bps);
  }

  toPercent(): number {
    return this.bps / 100;
  }
}
```

This is a value object. It enforces its own invariant. It cannot be in an invalid state. If you write `ProfitShareRatio.create(8000)` it throws. If you have a `ProfitShareRatio` instance in your hand, you *know* it is valid. That is the whole point.

### 4.3 Sample port (the interface)

```typescript
// backend-core/apps/svc-organization/src/application/ports/repositories/partner.repository.port.ts
import { Prisma } from "@prisma/client";
import { Partner } from "../../../domain/entities/partner.entity";

export interface IPartnerRepository {
  findById(id: string, organizationId?: string): Promise<Partner | null>;
  findByFabricPartnerId(fabricPartnerId: string): Promise<Partner | null>;
  create(tx: Prisma.TransactionClient, data: CreatePartnerInput): Promise<Partner>;
  update(tx: Prisma.TransactionClient, id: string, data: UpdatePartnerInput): Promise<Partner>;
}
```

Two things to notice:

- The methods take an optional `tx: Prisma.TransactionClient`. If passed, the repo uses it; otherwise it falls back to the global `db`. This lets a use-case pass the transaction so multiple repo calls all participate in the same atomic write. This is the idiom that makes L11 (atomic outbox) actually work.
- `findById` takes an *optional* `organizationId`. This is for multi-tenant safety (BOLA mitigation). If the calling context has a tenant, pass it; the repo enforces `WHERE organizationId = ?`. Session 121 audit caught one path that was not passing it. We fixed it.

### 4.4 Sample use-case (the heart)

This is `SubmitProfitReportUseCase`, after it absorbed the D2, D3, D5 fixes. Read it slowly:

```typescript
// backend-core/apps/svc-organization/src/application/use-cases/partner/settlement/submit-profit-report.use-case.ts
export class SubmitProfitReportUseCase {
  constructor(
    private readonly uow: UnitOfWorkPort,
    private readonly partnerRepo: PartnerRepositoryPort,
    private readonly profitRepo: PartnerProfitReportRepositoryPort,
    private readonly externalDataService: ExternalDataServicePort,
    private readonly outboxRepo: OutboxRepositoryPort,
    private readonly auditRepo: PartnerAuditLogRepositoryPort,
    private readonly logger: Logger,
  ) {}

  async execute(input: SubmitProfitReportInput): Promise<SubmitProfitReportResult> {
    // D5: pre-validate on-chain state before any write
    const partner = await this.partnerRepo.findById(input.partnerId);
    if (!partner) {
      throw new ConflictError(`Partner ${input.partnerId} not found`);
    }
    if (partner.balanceQirat < input.protocolShareQirat) {
      throw new InsufficientPartnerBalanceError(
        `Partner balance ${partner.balanceQirat} < required ${input.protocolShareQirat}`
      );
    }

    return this.uow.run(async (tx) => {
      // 1. Create the read-model row
      const report = await this.profitRepo.create(tx, {
        partnerId: input.partnerId,
        reportId: input.reportId,
        periodStart: input.periodStart,
        protocolShareQirat: input.protocolShareQirat,
        partnerShareQirat: input.partnerShareQirat,
        status: "SUBMITTED",
      });

      // 2. Create the outbox command (D3: same transaction)
      await this.outboxRepo.create(tx, {
        tenantId: input.partnerId,
        service: "svc-partner",                       // D2 fix
        commandType: CommandType.SUBMIT_PROFIT_REPORT_HASH,
        requestId: input.requestId,
        payload: {
          reportId: input.reportId,
          partnerId: input.partnerId,
          reportHash: input.reportHash,
          protocolShareQirat: input.protocolShareQirat,
          partnerShareQirat: input.partnerShareQirat,
        },
      });

      // 3. Audit row (same transaction; cannot be orphaned)
      await this.auditRepo.create(tx, {
        partnerOrgId: input.partnerId,
        actorProfileId: input.initiatorId,
        eventType: "PROFIT_REPORT_SUBMITTED",
        eventDetails: {
          reportId: input.reportId,
          periodStart: input.periodStart,
          protocolShare: input.protocolShareQirat,
        },
        targetResourceType: "PartnerProfitReport",
        targetResourceId: report.id,
        ipAddress: input.ipAddress,
        userAgent: input.userAgent,
        requestId: input.requestId,
      });

      this.logger.info("partner.profit_report.submitted", {
        partnerId: input.partnerId,
        reportId: input.reportId,
        protocolShareQirat: input.protocolShareQirat,
      });

      return {
        reportId: report.id,
        status: report.status,
        createdAt: report.createdAt,
      };
    });
  }
}
```

Read the comments. Every numbered step is a *deliberate* decision:

- **D5 fix**: we read the on-chain balance *before* writing any outbox command. If the partner has insufficient balance, we fail loudly with a typed domain error. Without this, we would have written a `SUBMIT_PROFIT_REPORT_HASH` command that the chain would reject, and now we have a diverged state where the read-model says SUBMITTED and the chain says nothing.
- **D3 fix**: the read-model write, the outbox write, and the audit write are *all inside one* `prisma.$transaction`. If any one fails, all three roll back. This is the atomicity rule.
- **D2 fix**: the `service` field is `'svc-partner'`, not `'svc-organization'`. The outbox-submitter uses this for routing. Consistency matters.
- **Typed errors**: `ConflictError`, `InsufficientPartnerBalanceError`. Never `throw new Error("foo")` from a use-case. Read the production-grade-code rule.
- **Structured logging**: a single line, machine-parseable, no PII, no secret values. The log will appear in Loki.

> **Why this way**: a use-case is a *transaction script with discipline*. It is allowed to know about ports, to read input, to write multiple things atomically, and to throw typed errors. It is *not* allowed to know about Prisma directly, or HTTP, or the outbox-submitter implementation. If you can mock the ports, you can unit-test the use-case with no infrastructure. That is the test you write first.

### 4.5 The DI container (manual wiring)

We do not use a DI framework. We have one file per service that constructs every instance by hand. For svc-organization (which contains the partner bounded context):
[backend-core/apps/svc-organization/src/shared/di/container.ts](../../backend-core/apps/svc-organization/src/shared/di/container.ts).

In Session 112 the audit flagged this file at 391 LOC. We refactored it in Session 114 by splitting it into modular sub-containers (lifecycle.di.ts, staking.di.ts, settlement.di.ts, portal.di.ts) each composing a slice of the partner use-cases. After the refactor, `container.ts` came down to 188 LOC.

```typescript
// container.ts (excerpt)
export const createPartnerUseCases = () => {
  // Repositories (singletons)
  const partnerRepo = new PrismaPartnerRepository();
  const applicationRepo = new PrismaPartnerApplicationRepository();
  const stakeRepo = new PrismaPartnerStakingRepository();
  const outboxRepo = new PrismaOutboxRepository();
  const auditRepo = new PrismaPartnerAuditLogRepository();

  // External services
  const externalDataService = new ExternalDataService();
  const webhookDispatch = getWebhookDispatchService();

  // Unit of Work
  const uow = new PrismaUnitOfWork();

  // Use-cases (transient)
  return {
    submitEoi: new SubmitEoiUseCase(uow, applicationRepo, auditRepo),
    approveLicense: new ApproveApplicationUseCase(
      uow, applicationRepo, partnerRepo, outboxRepo, auditRepo
    ),
    activatePartner: new ActivatePartnerUseCase(
      uow, partnerRepo, outboxRepo, auditRepo
    ),
    recordStake: new RecordStakeUseCase(uow, stakeRepo, outboxRepo),
    initiateSlash: new InitiateSlashOnchainUseCase(
      uow, stakeRepo, outboxRepo, auditRepo
    ),
    submitProfitReport: new SubmitProfitReportUseCase(
      uow, partnerRepo, profitRepo, externalDataService, outboxRepo, auditRepo, logger
    ),
    listPartners: new ListPartnersUseCase(partnerRepo),
  };
};
```

Why manual? Because explicit wiring is easy to read and easy to test. A reflection-based DI framework would shave maybe 30 lines per service and add a magic indirection that costs hours when something goes wrong at startup. We optimise for "comprehensible at 3am during an incident".

### 4.6 The HTTP layer (controllers, routes, DTOs)

The controller is dumb glue:

```typescript
// interface/controllers/partner-settlement.controller.ts
export class PartnerSettlementController {
  constructor(private readonly submitProfitReport: SubmitProfitReportUseCase) {}

  async submit(req: Request, res: Response): Promise<void> {
    const dto = SubmitProfitReportSchema.parse(req.body);  // Zod validation
    const result = await this.submitProfitReport.execute({
      partnerId: req.params.partnerId,
      initiatorId: req.user.profileId,
      ipAddress: req.ip,
      userAgent: req.headers["user-agent"] ?? "",
      requestId: req.id,
      ...dto,
    });
    res.status(201).json({ success: true, data: result });
  }
}
```

The route:

```typescript
// interface/routes/partner-settlement.routes.ts
export function createPartnerSettlementRoutes(
  controller: PartnerSettlementController
): Router {
  const router = Router();
  router.post(
    "/:partnerId/profit-reports",
    jwtAuthMiddleware,
    requireScopedAuth("partner_admin"),     // L13
    rateLimitMiddleware({ window: "1m", max: 30 }),
    (req, res, next) => controller.submit(req, res).catch(next),
  );
  return router;
}
```

The DTO and Zod schema live alongside the controller. Read [LECTURE-20-API-DESIGN-OPENAPI-VALIDATION.md](./LECTURE-20-API-DESIGN-OPENAPI-VALIDATION.md).

### 4.7 The error mapper (the only place HTTP knows about domain)

```typescript
// interface/middleware/error-handler.middleware.ts
export const errorHandler = (
  err: unknown, req: Request, res: Response, next: NextFunction
) => {
  if (err instanceof ConflictError) {
    return res.status(409).json({ success: false, error: err.message, code: err.code });
  }
  if (err instanceof InsufficientPartnerBalanceError) {
    return res.status(422).json({ success: false, error: err.message, code: err.code });
  }
  if (err instanceof NotFoundError) {
    return res.status(404).json({ success: false, error: err.message, code: err.code });
  }
  if (err instanceof ZodError) {
    return res.status(400).json({ success: false, error: "Validation failed", details: err.issues });
  }
  // Fallback: log full error, return generic message to caller (security)
  logger.error("unhandled.error", { err });
  res.status(500).json({ success: false, error: "Internal server error" });
};
```

This is the *only* place where the HTTP layer translates domain errors to status codes. Use-cases never call `res.status(...)`. Domain entities never know there is an HTTP request. This separation is what lets you add a CLI, a gRPC interface, or a queue consumer later without touching business logic.

---

## Phase 5: The CQRS bridge (outbox + projector)

This is the most subtle phase. Get it wrong and you ship financial software with diverged state. Read [LECTURE-04-CQRS-PATTERN-DEEP-DIVE.md](./LECTURE-04-CQRS-PATTERN-DEEP-DIVE.md) and [LECTURE-05-TRANSACTIONAL-OUTBOX-PATTERN.md](./LECTURE-05-TRANSACTIONAL-OUTBOX-PATTERN.md) before you do anything in this phase.

### 5.1 The mental model

```
       service                 outbox                    chain
   ─────────────────       ─────────────────       ─────────────────

   use-case writes  ─────► outbox row PENDING
   read-model + outbox     (atomic with read-model)
   in one $transaction

                           outbox-submitter polls
                               │
                               ▼
                           submits tx to Besu
                               │
                               ▼
                           marks outbox SUBMITTED  ─────► chain accepts,
                                                          emits event

                                                              │
                                                              ▼
                                                          projector reads event,
                                                          updates read-model again
                                                          (idempotent)
```

Two writes happen to the read-model: once at outbox time (the "command was issued" state), once at projection time (the "chain confirmed" state). The projector handler must be **idempotent**: if it sees the same event twice, the second run does nothing.

### 5.2 Outbox commands for MOD-11

19 command types. The CommandType enum lives in [backend-core/db/prisma/schema.prisma](../../backend-core/db/prisma/schema.prisma):

| Command | Producer use-case | Outbox-submitter handler | Facet function |
|---------|-------------------|--------------------------|----------------|
| `REGISTER_PARTNER` | approve-application | register-partner.command.ts | `registerPartner` |
| `ACTIVATE_PARTNER_LICENSE` | activate-partner | activate-partner-license.command.ts | `activatePartnerLicense` |
| `INITIATE_PARTNER_SLASH` | initiate-slash-onchain | initiate-slash.command.ts | `initiateSlash` |
| `APPROVE_PARTNER_SLASH` | approve-slash | approve-slash.command.ts | `approveSlash` |
| `EXECUTE_PARTNER_SLASH` | execute-slash | execute-slash.command.ts | `executeSlash` |
| `SUBMIT_PROFIT_REPORT_HASH` | submit-profit-report | submit-profit-report.command.ts | `submitProfitReportHash` |
| `INITIATE_PROFIT_SETTLEMENT` | initiate-settlement-onchain | initiate-settlement.command.ts | `initiateSettlement` |
| `APPROVE_PROFIT_SETTLEMENT` | approve-settlement | approve-settlement.command.ts | `approveSettlement` |
| `EXECUTE_PROFIT_SETTLEMENT` | execute-settlement | execute-settlement.command.ts | `executeSettlement` |
| `REVERSE_PROFIT_SETTLEMENT` | reverse-profit-share | reverse-settlement.command.ts | `reverseProfitShare` |
| ... and 9 more | | | |

> **Iron rule (L11)**: every producer use-case writes its outbox command in the same `prisma.$transaction` as its read-model write. A `outboxRepository.create()` outside `$transaction` is a CI failure. We have an ESLint rule for it: [backend-core/eslint-rules/outbox-create-must-be-transactional.js](../../backend-core/eslint-rules/outbox-create-must-be-transactional.js). Read [.claude/rules/cqrs-outbox.md](../../.claude/rules/cqrs-outbox.md).

### 5.3 An outbox-submitter command handler

The outbox-submitter is a worker that polls the `outbox_commands` table for `PENDING` rows, picks them up in arrival order (per partition), submits them to Besu, and marks them `SUBMITTED` on success. The handler for each command type lives in [backend-core/workers/outbox-submitter/src/commands/](../../backend-core/workers/outbox-submitter/src/commands/).

```typescript
// commands/partner/initiate-slash.command.ts
export class InitiateSlashCommand extends BaseCommandHandler {
  async handle(command: OutboxCommand): Promise<void> {
    const { partnerId, amountQirat, reason } = command.payload;

    const contract = await this.contractResolver.resolve("PartnerFacet");
    const tx = await contract.initiateSlash(partnerId, amountQirat, reason);
    const receipt = await tx.wait();

    const event = receipt.events.find((e) => e.event === "SlashInitiated");
    if (!event) {
      throw new Error("Expected SlashInitiated event missing from receipt");
    }
    const slashId = event.args.slashId;

    await this.prisma.partnerSlash.update({
      where: { id: command.payload.internalSlashId },
      data: { onchainSlashId: slashId },
    });

    await this.outboxRepository.markSubmitted(command.id);
  }
}
```

A few things to notice:

- The handler uses `contractResolver` to look up the facet. This is so we can swap addresses across environments without touching code.
- We extract `slashId` from the event in the receipt and persist it back to the off-chain row. This is the bridge between off-chain and on-chain identity.
- `markSubmitted` updates the outbox row to `SUBMITTED`. Failures throw, the worker catches, the row stays `PENDING`, and the next poll retries it.

### 5.4 Projector handlers (the read-model writer)

The projector is a separate worker. It subscribes to chain events. For each event, it has an idempotent handler. The handlers live in [backend-core/workers/projector/src/handlers/](../../backend-core/workers/projector/src/handlers/).

```typescript
// handlers/partner/slash-executed.handler.ts
export class SlashExecutedHandler extends BaseEventHandler {
  async handle(event: ChainEvent<SlashExecutedArgs>): Promise<void> {
    // 1. Idempotency
    if (await this.isTransactionAlreadyProcessed(event.transactionHash)) {
      return;
    }

    // 2. Atomic read-model update
    await this.prisma.$transaction(async (tx) => {
      const slash = await tx.partnerSlash.findUnique({
        where: { onchainSlashId: event.args.slashId },
      });
      if (!slash) {
        this.logger.warn("partner.slash.no_off_chain_row", {
          slashId: event.args.slashId,
        });
        return;
      }

      await tx.partnerSlash.update({
        where: { id: slash.id },
        data: { status: "EXECUTED", executedAt: new Date() },
      });

      await tx.partnerStake.update({
        where: { id: slash.stakeId },
        data: {
          slashedAmountQirat: { increment: BigInt(event.args.amountQirat) },
        },
      });

      await this.markTransactionProcessed(tx, event.transactionHash);
    });

    // 3. Notifications outside transaction (non-fatal)
    try {
      await this.notify("partner.slash.executed", {
        slashId: event.args.slashId,
        partnerId: event.args.partnerId,
      });
    } catch (err) {
      this.logger.error("partner.slash.notify.failed", {
        slashId: event.args.slashId,
        err: err instanceof Error ? err.message : "unknown",
      });
    }
  }
}
```

The pattern (read [.claude/rules/cqrs-outbox.md](../../.claude/rules/cqrs-outbox.md)):

1. **Idempotency check first**. Without this, replaying events double-counts.
2. **All read-model mutations in `prisma.$transaction`**. Atomic.
3. **`markTransactionProcessed` inside the transaction.** If the read-model rolls back, so does the idempotency record.
4. **Notifications outside the transaction.** If the email service is down, the read-model still gets updated.
5. **Backward-compat field names** (defect D6 mitigation): legacy events used `fromID`, new events use `senderId`. Handlers read both: `const senderId = event.args.senderId ?? event.args.fromID`. Old blocks are immutable, so we can never drop the alias.

### 5.5 The fabricUserId-vs-profileId bridge

This trips up every new engineer. The chain emits events with `fabricUserId` (a 20-character human-readable id like `MY 506 ABI832 0PTNY 5758`). The off-chain database uses `profileId` (a UUID). They are not the same. The projector resolves one to the other via `UserProfile.fabricUserId`:

```typescript
const profile = await tx.userProfile.findUnique({
  where: { fabricUserId: event.args.fromID },
});
if (!profile) {
  this.logger.error("projector.no_profile_for_fabric_id", {
    fabricUserId: event.args.fromID,
  });
  return;
}
const wallet = await tx.wallet.findFirst({
  where: { profileId: profile.profileId },
});
```

If you ever see a projector handler doing `tx.wallet.findFirst({ where: { profileId: event.args.fromID } })`, that is a bug. Push back in code review.

---

## Phase 6: Frontend (Next.js App Router, scoped JWT, design system)

For MOD-11 we built a brand-new frontend application: the **Partner Admin Center** (PAC), at port 3003. New repo: [partner-admin-center/](../../partner-admin-center/).

### 6.1 What goes in a frontend app

- Routes (Next.js App Router; `(public)` for unauthenticated, `(auth)` for OAuth flow, `(main)` for the authenticated dashboard)
- Page-level data fetching (`fetch` in server components or `useQuery` in client components)
- Components (shadcn/ui base, custom components for partner-specific surfaces)
- Forms (React Hook Form + Zod, never raw `<input>`)
- State (TanStack Query for server state, Zustand for client state, never `useState` for cross-page state)
- Auth (OAuth2 PKCE access token + scoped JWT for sensitive routes)
- Design tokens (OKLCH colours from [DESIGN-SYSTEM-CANON.md](../specs/DESIGN-SYSTEM-CANON.md))
- Error and offline states (real ones, not stubs)

The PAC structure:

```
partner-admin-center/src/app/
├── (public)/             # unauthenticated
│   ├── 404/
│   ├── 500/
│   └── maintenance/
├── (auth)/               # OAuth2 PKCE flow
│   ├── signin/
│   ├── signin/callback/
│   ├── signout/
│   └── scoped-auth/      # PIN + 2FA gate
└── (main)/               # authenticated, sidebar layout
    ├── dashboard/        # 24h volume, error rate, settlement summary
    ├── api-keys/         # issue, rotate, revoke, IP allowlist
    ├── webhooks/         # CRUD, deliveries, replay, signature helper
    ├── usage/            # quotas, history, incidents
    ├── licenses/         # tier upgrade, PDF, invoice
    ├── validators/       # only visible if category == VAL
    ├── settlements/      # only visible if revenue-share enabled
    ├── team/             # members, invite, role
    ├── audit-log/        # events, filter, CSV/JSON export
    ├── docs/             # OpenAPI ref, SDK download, changelog
    ├── support/          # tickets, account-manager contact
    └── settings/         # profile, security, KYB, payout, notifications
```

13 core routes plus 30 sub-routes. Read [SPEC-MOD-11-PARTNER-ADMIN-CENTER-V1.md](../specs/SPEC-MOD-11-PARTNER-ADMIN-CENTER-V1.md).

### 6.2 OAuth2 PKCE plus scoped JWT

The auth pattern is standard for any frontend that talks to svc-auth. It is layered:

**Step 1: OAuth2 PKCE (RFC 9700)**

```
1. User lands at /signin
2. Frontend generates code_verifier (43-128 chars random)
   code_challenge = base64url(sha256(code_verifier))
3. Redirect to svc-auth /oauth/authorize?response_type=code
   &client_id=partner-admin-center
   &redirect_uri=...
   &code_challenge=...
   &code_challenge_method=S256
4. svc-auth verifies user session, returns code in redirect
5. Frontend POST /api/auth/signin/callback with code + verifier
   svc-auth validates verifier matches, returns access token (60 min)
6. Token stored in localStorage as gx_pac_access_token
```

**Step 2: Scoped JWT (PIN + 2FA gated)**

For sensitive routes (issue an API key, approve a settlement, slash a partner) we require an additional 30-minute scoped JWT:

```
1. User clicks /api-keys/new (sensitive route)
2. Middleware checks for X-Scoped-Authorization header; missing
3. Redirect to /scoped-auth?return=/api-keys/new
4. Page asks for PIN, calls svc-auth /verify-pin
5. If 2FA enabled, asks for TOTP, calls svc-auth /verify-2fa
6. svc-auth issues scoped JWT (scope: 'partner_admin', TTL 30 min)
7. Stored in sessionStorage as gx_pac_scoped_token
8. Redirect back to /api-keys/new
9. API call now attaches X-Scoped-Authorization
10. On 401 scope_expired, restart this flow
```

This is L13. The same pattern is used by government-treasury and institution flows. Read [LECTURE-12-ATTRIBUTE-BASED-ACCESS-CONTROL.md](./LECTURE-12-ATTRIBUTE-BASED-ACCESS-CONTROL.md) and [jwt-authentication-and-rbac.md](./jwt-authentication-and-rbac.md).

### 6.3 Components: copy-from-CC versus invent-new

We ship two kinds of components:

- **Copied from Command Center** (with adjustments): tables, filter bars, sidebar, status pills.
- **Invented for partner use cases**: 20+ new components.

A subset of the invented components:

| Component | Purpose |
|-----------|---------|
| `KeyPrefixDisplay` | Masked API key prefix with copy button and last-4 reveal on hover |
| `KeyScopeSelector` | Multi-checkbox scope picker (read, transfer, validator-ops, fsp-settlement) plus presets |
| `KeySecretRevealOnce` | One-time secret display with `.env`-format download (the secret is *never* shown again) |
| `WebhookDeliveryStatus` | Badge for SUCCESS, RETRYING, FAILED, DLQ |
| `WebhookSignatureHelper` | Stripe-compatible validator for the `GX-Signature: t=,v1=` header |
| `RateLimitGauge` | Live progress bar fed from `wss://dev-api.gxcoin.money/v1/usage/stream` |
| `SettlementApprovalQueue` | Three-eyes grid: initiator + approver + authoriser cells with timestamps |
| `SettlementPLBreakdown` | P&L card: revenue, costs, net, hash badge, GX-share split |
| `KYBDocumentChecklist` | Status table for KYB docs: MISSING, SUBMITTED, VERIFIED, REJECTED, EXPIRED |
| `IPAllowlistEditor` | CIDR input list (IPv4 + IPv6, max 10 entries, validates format) |

> **Why this way**: do not reach for the design system to draw a generic "table". Reach for it to draw a *partner-shaped* component. `SettlementApprovalQueue` knows about three-eyes. `WebhookSignatureHelper` knows about the Stripe-style header format we adopted. Domain knowledge in the component name is good.

### 6.4 The Next.js client/server gotcha

Next.js 16 server-renders `"use client"` components on first paint. That means anything that touches `window`, `document`, `localStorage`, or `sessionStorage` at module top level will crash on the server.

Wrong:

```typescript
"use client";
const token = localStorage.getItem("gx_pac_access_token"); // crashes on SSR
```

Right:

```typescript
"use client";
import { useEffect, useState } from "react";

function useAccessToken(): string | null {
  const [token, setToken] = useState<string | null>(null);
  useEffect(() => {
    setToken(localStorage.getItem("gx_pac_access_token"));
  }, []);
  return token;
}
```

Or guard explicitly:

```typescript
const token = typeof window === "undefined"
  ? null
  : localStorage.getItem("gx_pac_access_token");
```

### 6.5 Frontend-to-backend proxy

In dev, the frontend hits `http://localhost:3056` for partner endpoints. In production, the same URLs go through ingress to the in-cluster service. We use Next.js API routes as a thin proxy where we need to attach server-only secrets:

```
/api/v1/partners/keys     →  http://svc-partner:3056/api/v1/partners/keys
/api/v1/partners/webhooks →  http://svc-partner:3056/api/v1/partners/webhooks
```

Session 121 audit caught a bug where one proxy route was pointing to `svc-organization` instead of `svc-partner` (after the Session 120 service extraction). We fixed it. Always re-verify proxy routes when a service is extracted or renamed.

---

## Phase 7: Containerise and deploy (Docker, K3s, ingress, health)

> **Hard rule** (from CLAUDE.md): do not deploy to K8s until all work, testing, and verification is complete. Run services locally with `npx tsx watch src/index.ts` for fast iteration.

When you *do* deploy, here is what it takes.

### 7.1 The Dockerfile

We use a 3-stage Dockerfile per service:

```dockerfile
# Stage 1: deps
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
COPY apps/ ./apps/
COPY packages/ ./packages/
COPY db/prisma/ ./db/prisma/
RUN npm ci --legacy-peer-deps

# Stage 2: builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate --schema=./db/prisma/schema.prisma
RUN cp -r ./node_modules/.prisma ./node_modules/.prisma
RUN npx tsc -p apps/svc-partner/tsconfig.build.json
# Build core packages explicitly:
RUN npx tsc -p packages/core-config/tsconfig.json
RUN npx tsc -p packages/core-db/tsconfig.json
RUN npx tsc -p packages/core-errors/tsconfig.json
RUN npx tsc -p packages/core-http/tsconfig.json
RUN npx tsc -p packages/core-logger/tsconfig.json

# Stage 3: production
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -g 1001 gxprotocol \
  && adduser -S gxprotocol -u 1001 -G gxprotocol

# Copy built service:
COPY --from=builder --chown=gxprotocol:gxprotocol /app/apps/svc-partner/dist ./apps/svc-partner/dist
COPY --from=builder --chown=gxprotocol:gxprotocol /app/apps/svc-partner/package.json ./apps/svc-partner/package.json

# Copy ALL transitive deps (this is where new engineers get bitten):
COPY --from=builder --chown=gxprotocol:gxprotocol /app/packages/core-config ./packages/core-config
COPY --from=builder --chown=gxprotocol:gxprotocol /app/packages/core-db ./packages/core-db
COPY --from=builder --chown=gxprotocol:gxprotocol /app/packages/core-errors ./packages/core-errors
COPY --from=builder --chown=gxprotocol:gxprotocol /app/packages/core-http ./packages/core-http
COPY --from=builder --chown=gxprotocol:gxprotocol /app/packages/core-logger ./packages/core-logger
COPY --from=builder --chown=gxprotocol:gxprotocol /app/node_modules ./node_modules

# Symlink @gx/* packages into node_modules:
RUN mkdir -p /app/node_modules/@gx \
  && ln -s /app/packages/core-config /app/node_modules/@gx/core-config \
  && ln -s /app/packages/core-db /app/node_modules/@gx/core-db \
  && ln -s /app/packages/core-errors /app/node_modules/@gx/core-errors \
  && ln -s /app/packages/core-http /app/node_modules/@gx/core-http \
  && ln -s /app/packages/core-logger /app/node_modules/@gx/core-logger

USER gxprotocol
EXPOSE 3056
CMD ["node", "apps/svc-partner/dist/index.js"]
```

Read [.claude/rules/docker-monorepo.md](../../.claude/rules/docker-monorepo.md). The transitive-deps trap is the single most common Docker build failure. `core-http` depends on `core-errors`. If your service uses `core-http` and you do not COPY `core-errors` into the production stage, you will get `Cannot find module '@gx/core-errors'` at startup. Always copy: `core-config`, `core-db`, `core-errors`, `core-http`, `core-logger`. Add `core-fabric`, `core-storage`, `core-events` per service need.

### 7.2 Build, push, deploy

```bash
# 1. Type-check before Docker (Docker is too slow for tight loops)
cd backend-core/apps/svc-partner && npx tsc --noEmit

# 2. Build
cd backend-core
docker build -t localhost:5555/svc-partner:dev-$(git rev-parse --short HEAD) \
  -f apps/svc-partner/Dockerfile .

# 3. Push to in-cluster registry
docker push localhost:5555/svc-partner:dev-$(git rev-parse --short HEAD)

# 4. Roll out
sudo kubectl set image deployment/svc-partner \
  svc-partner=localhost:5555/svc-partner:dev-$(git rev-parse --short HEAD) \
  -n gx-backend

sudo kubectl rollout status deployment/svc-partner -n gx-backend
```

### 7.3 The K8s manifest

```yaml
# infra-ops/k8s/devnet/svc-partner-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: svc-partner
  namespace: gx-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: svc-partner
  template:
    metadata:
      labels:
        app: svc-partner
    spec:
      serviceAccountName: svc-partner
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
      containers:
        - name: svc-partner
          image: localhost:5555/svc-partner:dev-{{ .Values.gitSha }}
          imagePullPolicy: Always
          ports:
            - containerPort: 3056
              name: http
          env:
            - name: PORT
              value: "3056"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: svc-partner-secrets
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: svc-partner-secrets
                  key: jwt-secret
            - name: ALLOWED_ORIGINS
              value: "https://dev-partner.gxcoin.money,https://dev-api.gxcoin.money"
          livenessProbe:
            httpGet:
              path: /health
              port: 3056
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health
              port: 3056
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits:   { memory: 512Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: svc-partner
  namespace: gx-backend
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 3056
  selector:
    app: svc-partner
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: svc-partner-ingress
  namespace: gx-backend
spec:
  ingressClassName: nginx
  rules:
    - host: dev-api.gxcoin.money
      http:
        paths:
          - path: /api/v1/partners
            pathType: Prefix
            backend:
              service:
                name: svc-partner
                port: { number: 80 }
  tls:
    - hosts: [dev-api.gxcoin.money]
      secretName: letsencrypt-prod
```

K8s gotchas you will hit:

- **Service port vs container port mismatch**: `service.port` is what other services dial; `containerPort` is what the pod exposes. The service maps from one to the other via `targetPort`. If those do not align you get silent routing failures (no error, just no traffic).
- **Liveness probe must hit `/health`**, not `/livez` and not `/`. We standardised on `/health` returning a JSON `{ status: "ok" }`.
- **EADDRINUSE on rollout**: occasionally an old crashed pod still has the port held. `kubectl scale deployment/svc-partner --replicas=0`, wait 30 seconds, scale back to 1.
- **ImagePullBackOff**: the tag does not exist in the registry. `docker push` again, then re-roll.

Read [LECTURE-19-KUBERNETES-DEPLOYMENT-OPERATIONS.md](./LECTURE-19-KUBERNETES-DEPLOYMENT-OPERATIONS.md).

### 7.4 The svc-partner deployment fact

In Session 121 the audit found that svc-partner had been *extracted as code* from svc-organization (Session 120) but **had no deployment manifest**. It was a service that existed in the monorepo but did not run anywhere. We fixed this in Session 122: authored the manifest, wired up the secrets, the ingress rule, and the projector/outbox routing. This is a class of bug that is invisible from a code review of just the application code; it shows up only in an infra audit.

Lesson: when you extract a service, manifest changes are *part of the work*. They do not get to wait until "later".

---

## Phase 8: Observability, security, audit

The audit results in MOD-11 (Session 119 scorecard 91/100, Session 121 audit dropped to 53 then climbed back to 99/100 post-remediation in Session 122) made a few things very clear.

### 8.1 Observability is non-negotiable

Pillar 4 of the scorecard is observability. We scored 10/10. That was not luck. We adopted a fixed checklist:

- **OpenTelemetry**: every HTTP handler, use-case, and DB call has a span. Read [LECTURE-23-OPENTELEMETRY-OBSERVABILITY.md](./LECTURE-23-OPENTELEMETRY-OBSERVABILITY.md).
- **Prometheus exporters**: 5 of them (HTTP RED, Prisma latency, outbox lag, projector lag, JWT scope checks).
- **18 SLOs**: documented in [docs-dev-manazir/specs/OBSERVABILITY-CURRENT-STATE-2026-04-27.md](../specs/OBSERVABILITY-CURRENT-STATE-2026-04-27.md).
- **Grafana dashboards**: 5 (per service, per worker, per chain, per cluster).
- **Loki labels**: `service`, `tenantId`, `requestId`, `traceId`. The traceId is propagated end-to-end so you can grep one user's request across services.

When you ship a new use-case, the OTel span is a one-liner using the wrapper:

```typescript
import { tracer } from "@gx/core-logger";

async execute(input: SubmitProfitReportInput): Promise<SubmitProfitReportResult> {
  return tracer.startActiveSpan("partner.profit_report.submit", async (span) => {
    span.setAttribute("partner.id", input.partnerId);
    span.setAttribute("report.id", input.reportId);
    try {
      // ... use-case body ...
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

Add this on every public use-case method. It is cheap, it is uniform, and it pays dividends the first time you debug a slow request.

### 8.2 Security: the threat model and the audit findings

Read [docs-dev-manazir/specs/SECURITY-POSTURE-2026-04-27.md](../specs/SECURITY-POSTURE-2026-04-27.md). Read [.claude/rules/security-standards.md](../../.claude/rules/security-standards.md).

The Session 121 audit found 10 **P0** (must-fix-before-prod) items on MOD-11. They were:

| # | Finding | Fix |
|---|---------|-----|
| 1 | CORS fails-open on missing origin | Pin allowlist; reject by default |
| 2 | Rate limiting absent on admin routes | Add token-bucket middleware to all `(main)` paths |
| 3 | JWT algorithm not pinned (alg-confusion vector) | Explicit `algorithms: ['HS256']` whitelist |
| 4 | 3 missing projector handlers | Add `validateProfitReport`, `slashApproved`, `unstake` handlers |
| 5 | Outbox-submitter service-field fallback wrong | Strict matching, no fallback |
| 6 | 10 atomicOutbox use-cases missing service field | Audit all 19 commands; fix |
| 7 | svc-partner has no K8s deployment | Author manifest |
| 8 | Application layer imports from infrastructure | Extract port; rewire DI |
| 9 | Frontend proxy targets svc-organization not svc-partner | Update next.config.mjs |
| 10 | Multi-tenant BOLA (`findById` lacks `organizationId`) | Add tenant filter |

We fixed all 10 in Session 122. The remediation produced 215+ new tests (regression coverage so we never re-introduce the same bug class).

> **The lesson**: an audit is not optional. A spec is what you intend to build. An audit is what you actually built. They are usually different. Plan for the audit *before* you start.

### 8.3 The Super-Engineer Review Board

Before any plan touching production gets approved, we run a 3-wave, 9-agent review board (read the memory entry [feedback_super_engineer_review_board_full.md](../../../.claude/projects/-home-dev-manazir-code-vault/memory/feedback_super_engineer_review_board_full.md)). The waves are:

1. **Web research**: are there industry patterns or standards we are missing? (For MOD-11 this caught the EU Data Act portability requirement.)
2. **Live infrastructure verification**: does this plan actually match the cluster state? (Caught the missing svc-partner manifest.)
3. **Adversarial security hunt**: an adversarial agent tries to find ways to break the plan. (Caught the BOLA, the CORS fail-open, the JWT alg-confusion.)

This is not optional either. Skipping it is how you ship audit findings.

---

## Phase 9: Testing pyramid in practice

Read [LECTURE-18-TESTING-STRATEGIES.md](./LECTURE-18-TESTING-STRATEGIES.md), [LECTURE-25-TESTING-TROPHY-MODERN-TESTING.md](./LECTURE-25-TESTING-TROPHY-MODERN-TESTING.md), and [.claude/rules/testing-pyramid.md](../../.claude/rules/testing-pyramid.md).

For MOD-11 the test surfaces by layer:

| Layer | Type | Target | Actual |
|-------|------|--------|--------|
| Domain (entities, value-objects) | Unit (Vitest) | 95% | 96% |
| Application (use-cases) | Unit + Integration | 90% | 88% |
| Infrastructure (repos) | Integration (Testcontainers Postgres) | 70% | 72% |
| Interface (controllers, routes) | Integration | 60% | 58% |
| Solidity (PartnerFacet) | Hardhat | 50+ tests | 72+ tests |
| Frontend (PAC) | Vitest unit + Playwright E2E | Critical paths | 105 PAC tests (post-Session-122) |

### 9.1 Per-PR test requirements (from the bridge spec)

Every new use-case ships with at minimum:

1. **Domain validation test**: entity invariants enforced (try to create with bad input, expect throw).
2. **Outbox row content test**: command shape correct (assert payload fields and command type).
3. **Transaction atomicity test**: force throw mid-transaction, assert *both* read-model and outbox rolled back.
4. **Caller-distinct test** (where applicable): three-eyes enforced.

Sample atomicity test:

```typescript
describe("ApproveSettlementUseCase atomicity", () => {
  it("rolls back both read-model and outbox on error", async () => {
    // Arrange: prepare a valid input
    const input = makeValidInput();

    // Force the third write (audit) to throw
    auditRepo.create = jest.fn().mockRejectedValueOnce(new Error("DB down"));

    // Act
    await expect(useCase.execute(input)).rejects.toThrow("DB down");

    // Assert: nothing persisted
    const approval = await prisma.partnerSettlementApproval.findUnique({
      where: { id: input.approvalId },
    });
    const outbox = await prisma.outboxCommand.findFirst({
      where: { payload: { path: ["settlementId"], equals: input.settlementId } },
    });
    expect(approval?.approverId).toBeNull();
    expect(outbox).toBeNull();
  });
});
```

This test alone has caught half a dozen regressions where someone "simplified" a use-case by moving the outbox write outside the transaction.

### 9.2 The k6 load suite

Load tests live in `backend-core/test/load/`:

| Scenario | Profile | Target |
|----------|---------|--------|
| FSP burst settlement | 50 partners, 3 admins each | p95 < 90s, zero loss |
| Validator reward distribution | 21 validators concurrent | All complete < 60s, no double-credit |
| Partner directory read | 10k wallet users, 50 RPS | p95 < 200ms |
| EoI submission spike | 500 EoIs in 1 minute | All persist, zero loss |

```bash
cd backend-core/test/load
k6 run --vus 100 --duration 10m mod-11-projected-traffic.js
```

The load test was scaffolded by Session 119 but the live 100-vu, 10-min run against the cluster is operator carry-forward (in the MainNet-gate list).

### 9.3 Hardhat tests for the facet

Already covered in Phase 2. The aggregate is 72+ tests across six spec files. Run them with:

```bash
cd blockchain-besu
npx hardhat test
```

### 9.4 Frontend tests

Vitest unit tests for components, Playwright E2E for the critical participant flow:

1. Sign in to PAC
2. Issue an API key (scoped-auth gate)
3. Reveal the secret once
4. Configure a webhook
5. Trigger a test event, see delivery status
6. View settlement queue, approve a settlement (scoped-auth gate)
7. Sign out

Run:

```bash
cd partner-admin-center
npm run test           # Vitest
npx playwright test    # E2E
```

---

## Phase 10: Commit discipline, work records, and PR hygiene

This is the part new engineers most often skip. It is the part senior engineers most often look at first.

### 10.1 Commits: file by file, formal, descriptive

From [.claude/rules/git-workflow.md](../../.claude/rules/git-workflow.md):

```bash
# Bad
git add .
git commit -m "fix: various fixes"

# Good
git add backend-core/apps/svc-organization/src/application/use-cases/partner/settlement/submit-profit-report.use-case.ts
git commit -m "fix(partner): pre-validate on-chain balance before outbox write (D5)"

git add backend-core/apps/svc-organization/src/application/use-cases/partner/settlement/submit-profit-report.use-case.spec.ts
git commit -m "test(partner): add atomicity and balance-check tests for submit-profit-report"
```

Each file gets a separate commit with a descriptive message. The commit message format is:

```
type(scope): subject

[optional body explaining why]
```

Types: `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `perf`, `security`. Scopes for MOD-11: `partner`, `chaincode`, `bridge`, `pac`, `infra`, `db`. Imperative mood. Under 72 characters. No AI attribution. **No em dashes.**

### 10.2 Work records: storytelling, append only

Every session you work on this codebase you must update [docs-dev-manazir/workrecords/work-record-YYYY-MM-DD.md](../workrecords/). Read [.claude/rules/work-records.md](../../.claude/rules/work-records.md).

Work records are storytelling, not bullet lists. They answer:

- Why a decision was made
- What alternatives were considered
- What surprised us
- What patterns emerged

Sample (paraphrased from the actual MOD-11 work records):

> **Session 114, Track 0**: Container.ts decomposition. The audit had flagged the 391-LOC container as a single point of cognitive failure. Initial instinct was to split alphabetically. Stopped and asked: what is the actual axis of cohesion? Concluded: lifecycle vs staking vs settlement vs portal. Refactored along that axis. Result: 188 LOC plus four sub-modules. The container is now legible.

That paragraph is more useful to a future engineer than ten lines of "decomposed container.ts; reduced LOC". It tells you what was tried, what was rejected, and why.

**Append only.** Never overwrite a previous session's record. If a session record is incomplete, the next session adds a note: "Added in Session N: context limit prevented completion."

### 10.3 PR template

```markdown
## Summary
[1-3 bullet points about what changed and why]

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation

## Changes Made
- [Specific file or area]
- [Specific file or area]

## Closes
- D# defects
- L# decisions applied
- Audit findings closed

## Test plan
- [ ] Unit tests added or updated
- [ ] Integration tests updated
- [ ] Manual verification on DevNet
- [ ] OTel spans added for new use-cases

## Screenshots (if UI)

## Checklist
- [ ] Follows clean architecture rules
- [ ] No `any` types, no `@ts-ignore`, no silent fallbacks
- [ ] No em dashes in any new content
- [ ] Outbox writes atomic (if applicable)
- [ ] Self-review completed
```

---

## Phase 11: The Super-Engineer Review Board and audit remediation

I touched on this in Phase 8. Here is the full pattern.

### 11.1 The 3-wave review

Before any plan that touches production gets approved, we spawn nine agents across three waves:

**Wave 1: Industry research**
- Three agents, parallel
- Each researches a specific dimension (eg. "how do other partner platforms model FSP profit settlement?", "what does PCI DSS 4.0 require for audit-log retention?", "what is the EU Data Act portability obligation?")
- Output: list of patterns we should adopt or violations we should fix

**Wave 2: Live infrastructure verification**
- Three agents, parallel
- Each verifies one infra dimension (cluster state matches plan? secrets rotated? ingress wired?)
- Output: list of mismatches between intent and reality

**Wave 3: Adversarial security hunt**
- Three agents, parallel
- Each plays an adversary (insider threat, external attacker, supply-chain compromise)
- Output: list of attack paths

For MOD-11, the review board ran before Session 121 and produced the spec for the Session 121 audit. It also ran *after* Session 121 to validate the remediation plan. It is the difference between "we think this is secure" and "we have actively tried to break it".

### 11.2 Audit remediation methodology

When an audit finding lands on your desk:

1. **Classify**: P0 (must fix before prod), P1 (must fix this cycle), P2 (next cycle), P3 (backlog).
2. **Assign a defect number** so the fix-PR has something to close.
3. **Write the regression test first**. The test should fail against the current code.
4. **Fix the code**. The test should pass.
5. **Document in the work record**: "D# closed: [one-line description]".

For Session 122 we closed:
- 10 of 10 P0s
- 13 of 14 P1s
- 6 of 6 P2s
- 1 of 1 P3
- 215+ new tests added

The post-remediation scorecard climbed from 91/100 to 99/100.

### 11.3 What "done" looks like

You know a module is done when:

- All scorecard pillars score 8/10 or above
- All P0 audit findings closed
- The full E2E test suite passes
- A k6 load test at projected traffic (or a documented commitment to run it before MainNet) is in the file
- The work record for the final session has a "Carry-Forward" section that is either empty or explicitly deferred to MainNet
- The CLAUDE.md file is updated with the new module's status

Notice that "all bugs fixed" is *not* on this list. There are always bugs. The list is "all *known* bugs accounted for, classified, and either closed or explicitly deferred with a date".

---

## The Gotcha Registry (real bugs, real fixes)

A condensed list of bugs we hit during MOD-11 development, with the fix. Treat this as a checklist on your next module.

### Backend / Clean Architecture

1. **Application layer imports from infrastructure**: ESLint catches it. If you ever write `import { PrismaClient } from '@prisma/client'` inside `application/`, your build fails. Move the dep to a port + an infrastructure adapter.
2. **`(db as any).modelName` returns undefined at runtime**: never cast Prisma client. If TS does not see the model, `prisma generate` did not pick it up. Fix the schema or run `prisma generate`.
3. **`updateMany` matching zero rows is silent**: always check `result.count`. Log a warning if zero. This caught a projector bug where a typo in a where-clause made the handler a no-op.
4. **`.catch()` inside `$transaction()` does not prevent abort**: PostgreSQL aborts the transaction on the first error. A `.catch()` inside the callback does not save you. Use an existence-check before the operation.

### CQRS

5. **fabricUserId is not profileId**: chaincode events carry the 20-char human id; the database uses UUIDs. Always resolve via `UserProfile.fabricUserId`.
6. **Outbox written outside `$transaction`**: ESLint at error level catches it. `outboxRepository.create()` must always be inside `prisma.$transaction()`.
7. **Old field name aliases must stay**: blocks are immutable. If the chain emits `fromID` in legacy events and `senderId` in new events, your handler reads both: `const senderId = ev.args.senderId ?? ev.args.fromID`.
8. **Projector handler not idempotent**: every handler must `isTransactionAlreadyProcessed(event.transactionHash)` first. Replays are a fact of life.

### Database

9. **Multi-file Prisma schema not picked up by `prisma generate`**: monolithic `schema.prisma` is the source for `generate`. Multi-file is for `db push`. Keep your queryable models in `schema.prisma`.
10. **Migration recorded ≠ migration applied**: always verify column existence with `\d` or `information_schema.columns`.
11. **Missing index on `expiresAt`**: the sweeper scans the whole table without it. Add `@@index([expiresAt])`.

### Frontend

12. **`localStorage` access at module top in a `"use client"` component**: SSR crashes. Guard with `typeof window === "undefined"` or use `useEffect`.
13. **Scoped JWT comparison string vs number**: `parsed.expiresAt > Date.now()` is string-vs-number. Use `new Date(parsed.expiresAt).getTime()`.
14. **CSV formula injection**: cells starting with `=`, `+`, `-`, `@` are formulas in Excel. Sanitize with a leading apostrophe.
15. **Duplicate routes**: Next.js 16 errors on `/(root)/offline` and `/offline` coexisting. Pick one.

### Docker / K8s

16. **Missing transitive deps in production stage**: `core-http` needs `core-errors`. The build fails at `node dist/index.js` startup with `Cannot find module`. Always copy: `core-config`, `core-db`, `core-errors`, `core-http`, `core-logger`.
17. **`@gx/test-utils` in production build**: it should not be there. Remove from filter chain.
18. **Service `port` vs `targetPort` mismatch**: silent routing failure. Verify alignment.
19. **Liveness probe to wrong path**: standard is `/health`, returning JSON `{ status: "ok" }`.
20. **Image tag does not exist in registry**: `docker push` again.

### Auth

21. **JWT algorithm not pinned**: explicit `algorithms: ['HS256']` whitelist. Without it, alg-confusion attacks are possible.
22. **Scoped JWT 30-min TTL not enforced server-side**: client-side expiry checks are advisory. Server must reject.
23. **CC SSO needs `sso-exchange` endpoint in svc-admin**: bridges svc-auth OAuth token to admin session. Without it, CC users cannot log in via SSO.

### Security

24. **CORS fails-open**: pin allowlist; reject by default.
25. **Rate limiting absent on admin routes**: token bucket per route.
26. **BOLA (broken object-level auth)**: `findById` without tenant filter. Add `organizationId` parameter.
27. **PII in logs**: structure logs to never include passwords, secrets, full JWT.

This is not exhaustive. Read [.claude/rules/](../../.claude/rules/) for the full set of project rules.

---

## What is still incomplete on MOD-11

The user told me the module is "not 100% complete tho". Here is the precise truth, from [docs-dev-manazir/specs/MOD-11-FINAL-SCORECARD-2026-04-28.md](../specs/MOD-11-FINAL-SCORECARD-2026-04-28.md).

**MOD-11 is feature-complete on DevNet. It is operationally ready. Final scorecard 99/100 (post-Session-122 remediation).**

The 13 explicitly-deferred items are the **MainNet Cutover Gate**:

| # | Item | Why deferred |
|---|------|--------------|
| 1 | CLAUDE.md credentials purged, git history rewritten | DevNet dev-ergonomic; mainnet-sensitive |
| 2 | Postgres password, JWT_SECRET, admin password rotated | Same |
| 3 | Redis AUTH enabled | Coordinate with credential rotation |
| 4 | External smart-contract audit (Sherlock contest) GREEN | Scope drafted, contest scheduled post-MOD-11 |
| 5 | DPoP / FAPI 2.0 sender-constrained tokens (RFC 9449) | JWT HS256 to RS256/ES256 migration across 21 services |
| 6 | EU Data Act portability export (`/api/v1/partner/portability/export`, ISO 20022) | Spec drafted; 2-3 days impl |
| 7 | SOC 2 Type II observation window started | Control mapping done; observation starts post-MOD-11 |
| 8 | Diamond multi-sig upgrade (replace single-key with multi-sig + 7-day delay) | Spec designed; 1-2 weeks dev + audit |
| 9 | Outbox-submitter HA (leader election + multi-replica with shardKey) | Designed |
| 10 | Postgres replication (read-replica + WAL archive + PITR) | 3-4 days operator work |
| 11 | Audit log retention PCI DSS 4.0 floor (90d hot + 12mo archive) | Auto-purge scheduled |
| 12 | jsonwebtoken alg-confusion explicit-whitelist audit | 1-2 hours |
| 13 | OpenZeppelin v4 to v5 storage-deadlock verification | 4-6h manual review |

These are not "we forgot to ship". These are "we deliberately deferred to a later cutover". They live in the carry-forward registry until MainNet activation.

There is also a small set of post-audit cleanups (carry-forward, low-risk):

- D3 atomicity pattern applied to ~12 mutation paths (the rest of the codebase, not just MOD-11)
- 5 read-only helpers still direct-import `db` instead of going through a port (low-risk; the read-only path does not write to outbox)
- svc-loanpool `@ts-nocheck` count is 10 vs the baseline of 6 (we accumulated debt during MOD-11; need a cleanup PR)

These are tracked as P1/P2 carry-forward items, not blockers.

---

## Exercise: develop MOD-12 (Messaging) on your own

You should not just read this guide. You should use it. MOD-12 is the next pending module. The mandate paragraph for MOD-12, summarised:

> Participants need to receive notifications about events that affect them: KYC approved or rejected, transfer received, settlement approved, validator reward distributed, partner application advanced, etc. The protocol needs an email + SMS messaging service that templating, throttling, dead-letter handling, and observability of delivery status. It must integrate with the existing notifications-ui package for in-app notifications, and it must respect each participant's notification preferences (email yes/no, SMS yes/no, per category).

Your assignment, before you write any code:

1. **Phase 0**: write the mandate, the locked-decisions register (L1 to L?), and the scope/non-scope.
   - Hint: which of the 7 messaging providers (SendGrid, AWS SES, Twilio, etc.) do we adopt? Why?
   - Hint: where do delivery receipts live (which DB table)? What is the retention?
   - Hint: how do we throttle to avoid spam if a buggy projector emits 1000 events?

2. **Phase 1**: write the spec stack.
   - SPEC-MOD-12-IMPLEMENTATION-PLAN-V1.md
   - SPEC-MOD-12-MESSAGING-API-V1.md (if needed)
   - SPEC-MOD-12-NOTIFICATION-PREFERENCES-V1.md (likely)

3. **Phase 2**: chain surface. Probably none. (Notifications are off-chain.) But ask the question explicitly. If you decide no chain involvement, document the decision.

4. **Phase 3**: schema. New models for `MessageTemplate`, `MessageDelivery`, `NotificationPreferences`.

5. **Phase 4**: backend. The `svc-messaging` service exists today as a stub (port 3007, zero use-cases). Implement the use-cases.

6. **Phase 5**: CQRS. Probably no outbox-to-chain, but events from other services trigger notifications. Design the event-bus integration.

7. **Phase 6**: frontend. There is an existing `notifications-ui` package. How do you extend it for the new categories?

8. **Phase 7**: deploy. svc-messaging needs a Dockerfile and a K8s manifest. (It does not have one today.)

9. **Phase 8**: observability. SLOs for delivery success rate, retry rate, dead-letter rate.

10. **Phase 9**: tests. Mock the email provider in unit tests. Use a real SMTP testcontainer for integration.

11. **Phase 10**: commit, work record, PR.

12. **Phase 11**: review board, audit remediation.

If you can produce all of that, you have proven the lifecycle. Send the spec stack for review before you write a line of code. Senior engineers will read it, push back, and refine. That is the pattern.

---

## Required reading and lecture cross-references

Open these in the order listed. Each one is in [docs-dev-manazir/lectures/](.).

### Foundation
1. [LECTURE-02-INTRODUCTION-TO-GX-PROTOCOL.md](./LECTURE-02-INTRODUCTION-TO-GX-PROTOCOL.md): what the protocol is and why
2. [LECTURE-01-CORE-PACKAGES-DEEP-DIVE.md](./LECTURE-01-CORE-PACKAGES-DEEP-DIVE.md): shared packages, monorepo
3. [LECTURE-09-MONOREPO-TURBOREPO.md](./LECTURE-09-MONOREPO-TURBOREPO.md): how the monorepo holds together
4. [LECTURE-10-CONFIGURATION-ZOD.md](./LECTURE-10-CONFIGURATION-ZOD.md): Zod-validated config

### Architecture
5. [LECTURE-21-CLEAN-ARCHITECTURE-HEXAGONAL-PATTERN.md](./LECTURE-21-CLEAN-ARCHITECTURE-HEXAGONAL-PATTERN.md): the four layers
6. [LECTURE-26-BOUNDED-CONTEXTS-DATA-OWNERSHIP.md](./LECTURE-26-BOUNDED-CONTEXTS-DATA-OWNERSHIP.md): DDD in microservices
7. [LECTURE-22-POSTGRESQL-RLS-MULTI-TENANCY.md](./LECTURE-22-POSTGRESQL-RLS-MULTI-TENANCY.md): multi-tenant data isolation
8. [LECTURE-08-PRISMA-DATABASE-DESIGN.md](./LECTURE-08-PRISMA-DATABASE-DESIGN.md): schema, migrations, gotchas

### Blockchain
9. [LECTURE-03-HYPERLEDGER-FABRIC-BLOCKCHAIN.md](./LECTURE-03-HYPERLEDGER-FABRIC-BLOCKCHAIN.md): legacy chain (still present)
10. [LECTURE-11-SMART-CONTRACT-ARCHITECTURE.md](./LECTURE-11-SMART-CONTRACT-ARCHITECTURE.md): design principles
11. [BESU-SOLIDITY-DIAMOND-LECTURE.md](./BESU-SOLIDITY-DIAMOND-LECTURE.md): current chain (Besu Diamond)
12. [LECTURE-13-GENESIS-DISTRIBUTION-TOKENOMICS.md](./LECTURE-13-GENESIS-DISTRIBUTION-TOKENOMICS.md): Qirat, units, GX
13. [LECTURE-14-MULTI-SIGNATURE-TRANSACTIONS.md](./LECTURE-14-MULTI-SIGNATURE-TRANSACTIONS.md): three-eyes pattern
14. [LECTURE-15-ON-CHAIN-GOVERNANCE-VOTING.md](./LECTURE-15-ON-CHAIN-GOVERNANCE-VOTING.md): proposal/voting (MOD-15)
15. [LECTURE-16-LOAN-POOL-INTEREST-FREE-LENDING.md](./LECTURE-16-LOAN-POOL-INTEREST-FREE-LENDING.md): MOD-14 architecture

### CQRS / Outbox
16. [LECTURE-04-CQRS-PATTERN-DEEP-DIVE.md](./LECTURE-04-CQRS-PATTERN-DEEP-DIVE.md): the why of CQRS
17. [LECTURE-05-TRANSACTIONAL-OUTBOX-PATTERN.md](./LECTURE-05-TRANSACTIONAL-OUTBOX-PATTERN.md): atomicity guarantees
18. [LECTURE-06-EVENT-DRIVEN-PROJECTIONS.md](./LECTURE-06-EVENT-DRIVEN-PROJECTIONS.md): projector design
19. [LECTURE-07-FABRIC-SDK-CIRCUIT-BREAKERS.md](./LECTURE-07-FABRIC-SDK-CIRCUIT-BREAKERS.md): chain submission resilience

### Cross-cutting
20. [LECTURE-12-ATTRIBUTE-BASED-ACCESS-CONTROL.md](./LECTURE-12-ATTRIBUTE-BASED-ACCESS-CONTROL.md): RBAC + ABAC
21. [LECTURE-17-USER-IDENTITY-KYC-VERIFICATION.md](./LECTURE-17-USER-IDENTITY-KYC-VERIFICATION.md): identity stack
22. [jwt-authentication-and-rbac.md](./jwt-authentication-and-rbac.md): auth deep dive
23. [LECTURE-20-API-DESIGN-OPENAPI-VALIDATION.md](./LECTURE-20-API-DESIGN-OPENAPI-VALIDATION.md): HTTP + Zod
24. [LECTURE-23-OPENTELEMETRY-OBSERVABILITY.md](./LECTURE-23-OPENTELEMETRY-OBSERVABILITY.md): traces, metrics, logs
25. [LECTURE-24-RELIABILITY-PATTERNS.md](./LECTURE-24-RELIABILITY-PATTERNS.md): circuit breakers, idempotency
26. [LECTURE-19-KUBERNETES-DEPLOYMENT-OPERATIONS.md](./LECTURE-19-KUBERNETES-DEPLOYMENT-OPERATIONS.md): K3s, ingress, secrets
27. [LECTURE-18-TESTING-STRATEGIES.md](./LECTURE-18-TESTING-STRATEGIES.md): test types and coverage
28. [LECTURE-25-TESTING-TROPHY-MODERN-TESTING.md](./LECTURE-25-TESTING-TROPHY-MODERN-TESTING.md): Vitest, Testcontainers, Pact

### Operational
29. [ENTERPRISE-DEBUGGING-LECTURE.md](./ENTERPRISE-DEBUGGING-LECTURE.md): and parts 1-5: when something breaks
30. [WSL2-SETUP-GUIDE.md](./WSL2-SETUP-GUIDE.md): local environment

### Project rules (every PR enforced)
- [.claude/rules/clean-architecture.md](../../.claude/rules/clean-architecture.md)
- [.claude/rules/cqrs-outbox.md](../../.claude/rules/cqrs-outbox.md)
- [.claude/rules/docker-monorepo.md](../../.claude/rules/docker-monorepo.md)
- [.claude/rules/git-workflow.md](../../.claude/rules/git-workflow.md)
- [.claude/rules/language-specification.md](../../.claude/rules/language-specification.md)
- [.claude/rules/prisma-schema.md](../../.claude/rules/prisma-schema.md)
- [.claude/rules/production-grade-code.md](../../.claude/rules/production-grade-code.md)
- [.claude/rules/security-standards.md](../../.claude/rules/security-standards.md)
- [.claude/rules/testing-pyramid.md](../../.claude/rules/testing-pyramid.md)
- [.claude/rules/work-records.md](../../.claude/rules/work-records.md)

---

## Closing

You now have every piece of the puzzle. You have the map (Section 1), the definition of what we are building (Section 2), the lifecycle (Section 3), the eleven phases in order, the gotcha registry, the honest assessment of what is unfinished, and an exercise.

Three things to internalise:

1. **The order of phases is not arbitrary.** Specs before code. Chain before backend. Backend before frontend. Tests with code, not after. Deploy last. The order keeps the radius of mistakes small.

2. **Locked decisions save you.** Every minute spent in Phase 0 saves an hour in Phase 4. The L# register is the cheapest tool we have.

3. **Audit is part of the work.** A module is not done at "tests pass". It is done at "audit findings closed and either fixed or explicitly deferred". Plan for the audit.

When you ship MOD-12, send me the spec stack. We will sit down and walk through the L# register together.

Welcome to the team.

/ senior engineer
