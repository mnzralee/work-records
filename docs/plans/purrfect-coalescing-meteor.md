# Comprehensive Cleanup: Registration, Login, and KYC Flows

## Date: 2026-01-18

## Overview

Clean up and enhance the user registration, login, and KYC onboarding flows across all layers (Database, Backend, BFF, Frontend). Remove redundant code, enhance KYC Step 1 with additional fields, and create a clean Review page.

---

## Current State Analysis

### Registration Flow
- **Architecture**: Stateless 5-step flow (no PendingRegistration table)
- **Files**: `svc-identity/src/controllers/registration.controller.ts`, `registration.service.ts`
- **Issues Found**:
  - Code duplication in `checkEmail/resendEmailOtp` and `checkPhone/resendPhoneOtp`
  - Dual registration endpoints (`/auth/register` vs `/registration/complete`)
  - Liveness scores accepted but never persisted

### Login Flow
- **Architecture**: Lightweight JWT-based authentication
- **Files**: `svc-identity/src/controllers/auth.controller.ts`, `auth.service.ts`
- **Issues Found**:
  - Token revocation NOT implemented (logout has TODO comment)
  - No session tracking (unlike admin auth)
  - User status not validated on each request (suspended users can still use old tokens)

### KYC Flow
- **Current Step 1**: Only collects DOB and Gender
- **Required Step 1**: Name (pre-filled), DOB, Gender, Nationality, Address, Phone (pre-filled)
- **Files**: `svc-identity/src/controllers/kyc.controller.ts`, `kyc.service.ts`

### Schema Analysis (PendingRegistration - NOT USED)
The `PendingRegistration` model (lines 507-607 in schema.prisma) is NOT actively used:
- Registration service creates `UserProfile` directly
- Contains many outdated fields for KYC that now live in `KYCApplication`
- Should be deprecated/removed

---

## Implementation Plan

### Phase 1: Database Schema Cleanup

**Goal**: Remove redundant models and add missing fields to KYCApplication

#### 1.1 Deprecate PendingRegistration Model
The `PendingRegistration` model is unused. Mark as deprecated (don't delete yet for safety).

```prisma
/// @deprecated This model is not used. Registration is stateless.
/// All data is collected on frontend and UserProfile is created directly.
/// Kept for backward compatibility - can be removed in future migration.
model PendingRegistration {
  // ... existing fields
}
```

#### 1.2 Add Fields to KYCApplication for Enhanced Step 1

**Current KYCApplication Step 1 fields** (lines 890-894):
```prisma
dateOfBirth     DateTime? @db.Date
gender          Gender?
genesisEligible Boolean   @default(false)
```

**Add new fields for enhanced Step 1**:
```prisma
// Step 1: Personal Information (enhanced)
firstName              String?   // Captured/confirmed during KYC
lastName               String?   // Captured/confirmed during KYC
nationalityCountryCode String?   @db.Char(2)  // Selected during KYC
phoneNumber            String?   // Confirmed during KYC
dateOfBirth            DateTime? @db.Date
gender                 Gender?
genesisEligible        Boolean   @default(false)

// Address (inline for simplicity, not separate table)
addressLine1    String?
addressLine2    String?
city            String?
stateProvince   String?
postalCode      String?
addressCountry  String?   @db.Char(2)
```

**Migration File**: `db/prisma/migrations/YYYYMMDDHHMMSS_enhance_kyc_step1/migration.sql`

```sql
-- Add enhanced Step 1 fields to KYCApplication
ALTER TABLE "KYCApplication" ADD COLUMN "firstName" TEXT;
ALTER TABLE "KYCApplication" ADD COLUMN "lastName" TEXT;
ALTER TABLE "KYCApplication" ADD COLUMN "nationalityCountryCode" CHAR(2);
ALTER TABLE "KYCApplication" ADD COLUMN "phoneNumber" TEXT;
ALTER TABLE "KYCApplication" ADD COLUMN "addressLine1" TEXT;
ALTER TABLE "KYCApplication" ADD COLUMN "addressLine2" TEXT;
ALTER TABLE "KYCApplication" ADD COLUMN "city" TEXT;
ALTER TABLE "KYCApplication" ADD COLUMN "stateProvince" TEXT;
ALTER TABLE "KYCApplication" ADD COLUMN "postalCode" TEXT;
ALTER TABLE "KYCApplication" ADD COLUMN "addressCountry" CHAR(2);
```

---

### Phase 2: Backend Service Updates

#### 2.1 Update KYC Service - Personal Info DTO

**File**: `/home/dev-manazir/prod-blockchain/gx-protocol-backend/apps/svc-identity/src/services/kyc.service.ts`

**Update `KYCPersonalInfoDTO`**:
```typescript
interface KYCPersonalInfoDTO {
  // Required (pre-filled, user confirms)
  firstName: string;
  lastName: string;
  phoneNumber: string;

  // Required (user enters)
  dateOfBirth: string;  // ISO YYYY-MM-DD
  gender: 'MALE' | 'FEMALE';
  nationalityCountryCode: string;  // ISO 3166-1 alpha-2

  // Address (required)
  addressLine1: string;
  addressLine2?: string;
  city: string;
  stateProvince?: string;
  postalCode?: string;
  addressCountry: string;  // ISO 3166-1 alpha-2
}
```

**Update `updatePersonalInfo()` method**:
- Accept all new fields
- Validate country codes against Country table
- Calculate genesis eligibility from DOB
- Update step1Complete logic to check all required fields

**Update `formatApplication()` response**:
- Include all new fields in response
- Update `step1Complete` calculation:
```typescript
const step1Complete = Boolean(
  application.firstName &&
  application.lastName &&
  application.phoneNumber &&
  application.dateOfBirth &&
  application.gender &&
  application.nationalityCountryCode &&
  application.addressLine1 &&
  application.city &&
  application.addressCountry
);
```

#### 2.2 Update KYC Controller Validation

**File**: `/home/dev-manazir/prod-blockchain/gx-protocol-backend/apps/svc-identity/src/controllers/kyc.controller.ts`

Add Zod validation for enhanced personal info:
```typescript
const personalInfoSchema = z.object({
  firstName: z.string().min(2).max(50),
  lastName: z.string().min(2).max(50),
  phoneNumber: z.string().regex(/^\+[1-9]\d{6,14}$/),
  dateOfBirth: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  gender: z.enum(['MALE', 'FEMALE']),
  nationalityCountryCode: z.string().length(2).toUpperCase(),
  addressLine1: z.string().min(5).max(100),
  addressLine2: z.string().max(100).optional(),
  city: z.string().min(2).max(50),
  stateProvince: z.string().max(50).optional(),
  postalCode: z.string().max(20).optional(),
  addressCountry: z.string().length(2).toUpperCase(),
});
```

#### 2.3 Registration Service Cleanup (Optional)

**File**: `/home/dev-manazir/prod-blockchain/gx-protocol-backend/apps/svc-identity/src/services/registration.service.ts`

**Refactor duplicated OTP logic**:
```typescript
// Extract common logic into private helper methods
private async sendEmailOtp(email: string, isResend: boolean): Promise<{ message: string; otp?: string }> {
  // Shared logic for checkEmail and resendEmailOtp
}

private async sendPhoneOtp(phoneNum: string, isResend: boolean): Promise<{ message: string; otp?: string }> {
  // Shared logic for checkPhone and resendPhoneOtp
}
```

---

### Phase 3: Frontend Updates

#### 3.1 Update KYC Types

**File**: `/home/dev-manazir/prod-blockchain/gx-wallet-frontend/types/kyc.ts`

```typescript
export interface PersonalInfoRequest {
  // Pre-filled from registration (editable)
  firstName: string;
  lastName: string;
  phoneNumber: string;

  // User enters
  dateOfBirth: string;
  gender: KYCGender;
  nationalityCountryCode: string;

  // Address
  addressLine1: string;
  addressLine2?: string;
  city: string;
  stateProvince?: string;
  postalCode?: string;
  addressCountry: string;
}

export interface KYCApplication {
  // ... existing fields

  // Enhanced Step 1 fields
  firstName?: string;
  lastName?: string;
  phoneNumber?: string;
  nationalityCountryCode?: string;
  addressLine1?: string;
  addressLine2?: string;
  city?: string;
  stateProvince?: string;
  postalCode?: string;
  addressCountry?: string;
}
```

#### 3.2 Redesign Personal Info Page (Step 1)

**File**: `/home/dev-manazir/prod-blockchain/gx-wallet-frontend/app/onboarding/kyc/personal-info/page.tsx`

**New Layout (2-column grid on desktop)**:
```
┌─────────────────────────────────────────────────────────────┐
│ Personal Information                              Step 1/4  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌─────────────────────┐  ┌─────────────────────┐           │
│ │ First Name*         │  │ Last Name*          │           │
│ │ [John           ]   │  │ [Doe            ]   │           │
│ │ Pre-filled ✓        │  │ Pre-filled ✓        │           │
│ └─────────────────────┘  └─────────────────────┘           │
│                                                             │
│ ┌─────────────────────┐  ┌─────────────────────┐           │
│ │ Date of Birth*      │  │ Gender*             │           │
│ │ [1990-01-15    ]    │  │ ○ Male  ● Female    │           │
│ └─────────────────────┘  └─────────────────────┘           │
│                                                             │
│ ┌─────────────────────┐  ┌─────────────────────┐           │
│ │ Nationality*        │  │ Phone Number*       │           │
│ │ [United States ▼]   │  │ [+1 555 123 4567]   │           │
│ │                     │  │ Pre-filled ✓        │           │
│ └─────────────────────┘  └─────────────────────┘           │
│                                                             │
│ ─────────── Address ───────────                             │
│                                                             │
│ ┌───────────────────────────────────────────────┐          │
│ │ Address Line 1*                               │          │
│ │ [123 Main Street                          ]   │          │
│ └───────────────────────────────────────────────┘          │
│                                                             │
│ ┌───────────────────────────────────────────────┐          │
│ │ Address Line 2 (Optional)                     │          │
│ │ [Apt 4B                                   ]   │          │
│ └───────────────────────────────────────────────┘          │
│                                                             │
│ ┌─────────────────────┐  ┌─────────────────────┐           │
│ │ City*               │  │ State/Province      │           │
│ │ [New York       ]   │  │ [NY             ]   │           │
│ └─────────────────────┘  └─────────────────────┘           │
│                                                             │
│ ┌─────────────────────┐  ┌─────────────────────┐           │
│ │ Postal Code         │  │ Country*            │           │
│ │ [10001          ]   │  │ [United States ▼]   │           │
│ └─────────────────────┘  └─────────────────────┘           │
│                                                             │
│                               [Continue →]                  │
└─────────────────────────────────────────────────────────────┘
```

**Implementation Notes**:
- Pre-fill firstName, lastName, phoneNumber from user session/profile
- Show "Pre-filled ✓" badge on pre-filled fields
- Country selector with search (use existing country list)
- Validate all fields before enabling Continue button
- Show validation errors inline

#### 3.3 Redesign Review Page (Step 4)

**File**: `/home/dev-manazir/prod-blockchain/gx-wallet-frontend/app/onboarding/kyc/review/page.tsx`

**New Layout**:
```
┌─────────────────────────────────────────────────────────────┐
│ Review Your Application                           Step 4/4  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Personal Information                          [Edit ✎] │ │
│ ├─────────────────────────────────────────────────────────┤ │
│ │ Name           John Doe                                 │ │
│ │ Date of Birth  January 15, 1990 (34 years old)         │ │
│ │ Gender         Male                                     │ │
│ │ Nationality    United States                            │ │
│ │ Phone          +1 555 123 4567                          │ │
│ │ Address        123 Main Street, Apt 4B                  │ │
│ │                New York, NY 10001, United States        │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Identity Documents                            [Edit ✎] │ │
│ ├─────────────────────────────────────────────────────────┤ │
│ │ Document Type  Passport                                 │ │
│ │ Document #     AB1234567                                │ │
│ │ Expiry Date    December 31, 2028                        │ │
│ │ ┌─────────┐ ┌─────────┐                                 │ │
│ │ │ [Front] │ │ [Back]  │  ✓ Uploaded                    │ │
│ │ └─────────┘ └─────────┘                                 │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Face Verification                             [Edit ✎] │ │
│ ├─────────────────────────────────────────────────────────┤ │
│ │ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐                 │ │
│ │ │ [👤]  │ │ [←👤] │ │ [👤→] │ │ [📷]  │  ✓ Captured    │ │
│ │ │Front  │ │ Left  │ │ Right │ │Recent │                 │ │
│ │ └───────┘ └───────┘ └───────┘ └───────┘                 │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ ─────────── Terms & Conditions ───────────                  │
│                                                             │
│ ☐ I agree to the Terms of Service and Privacy Policy       │
│   I understand my data will be processed for identity       │
│   verification purposes in accordance with applicable laws. │
│                                                             │
│ [← Back]                          [Submit Application →]    │
│                                                             │
│ ⓘ Your application will be reviewed within 24-48 hours.    │
└─────────────────────────────────────────────────────────────┘
```

**Implementation Notes**:
- Clean, enterprise-style summary cards
- Edit buttons navigate to respective steps
- Single terms checkbox (combines terms, privacy, data consent)
- Submit button disabled until checkbox checked
- Success state shows "Under Review" message

---

### Phase 4: API Route Updates

#### 4.1 Update Personal Info API Route

**File**: `/home/dev-manazir/prod-blockchain/gx-wallet-frontend/app/api/kyc/applications/[id]/personal-info/route.ts`

Update to accept and forward all new fields:
```typescript
export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const body = await request.json();

  // Validate all required fields
  const {
    firstName, lastName, phoneNumber,
    dateOfBirth, gender, nationalityCountryCode,
    addressLine1, addressLine2, city, stateProvince, postalCode, addressCountry
  } = body;

  // Forward to backend
  const response = await fetch(
    `${IDENTITY_API_URL}/api/v1/kyc/applications/${params.id}/personal-info`,
    {
      method: 'PUT',
      headers: { ...authHeaders, 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    }
  );

  return NextResponse.json(await response.json(), { status: response.status });
}
```

---

## Files Summary

### Database (gx-protocol-backend)
| File | Change |
|------|--------|
| `db/prisma/schema.prisma` | Add enhanced Step 1 fields to KYCApplication, deprecate PendingRegistration |
| `db/prisma/migrations/*/migration.sql` | Add new columns |

### Backend (gx-protocol-backend/apps/svc-identity)
| File | Change |
|------|--------|
| `src/services/kyc.service.ts` | Update PersonalInfoDTO, updatePersonalInfo(), formatApplication(), step1Complete logic |
| `src/controllers/kyc.controller.ts` | Update validation schema for enhanced personal info |
| `src/services/registration.service.ts` | (Optional) Refactor duplicated OTP logic |

### Frontend (gx-wallet-frontend)
| File | Change |
|------|--------|
| `types/kyc.ts` | Add new fields to PersonalInfoRequest and KYCApplication |
| `app/onboarding/kyc/personal-info/page.tsx` | Full redesign with all new fields |
| `app/onboarding/kyc/review/page.tsx` | Redesign with clean summary cards and terms checkbox |
| `app/api/kyc/applications/[id]/personal-info/route.ts` | Forward all new fields |

---

## Execution Order

1. **Phase 1**: Database migration (add columns to KYCApplication)
2. **Phase 2**: Backend service updates (KYC service, validation)
3. **Phase 3**: Frontend updates (types, personal-info page, review page)
4. **Phase 4**: Deploy and test

---

## Verification

### Manual Testing
1. Start frontend: `npm run dev` in gx-wallet-frontend
2. Port-forward backend: `kubectl port-forward -n backend-dev-manazir svc/svc-identity 3041:80 &`
3. Navigate to `http://localhost:3000/onboarding/kyc/personal-info`
4. Verify:
   - Name and phone pre-filled from registration
   - All new fields visible and working
   - Validation errors show inline
   - Data persists when navigating back
5. Complete all steps to Review page
6. Verify:
   - All data displayed correctly in summary cards
   - Edit buttons navigate to correct steps
   - Terms checkbox enables submit button
   - Submit sends application for review

### API Testing
```bash
# Test enhanced personal info
curl -X PUT http://localhost:3041/api/v1/kyc/applications/<id>/personal-info \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Doe",
    "phoneNumber": "+15551234567",
    "dateOfBirth": "1990-01-15",
    "gender": "MALE",
    "nationalityCountryCode": "US",
    "addressLine1": "123 Main Street",
    "addressLine2": "Apt 4B",
    "city": "New York",
    "stateProvince": "NY",
    "postalCode": "10001",
    "addressCountry": "US"
  }'
```

---

## Out of Scope (Deferred)

The following issues were identified but deferred for future work:

### Login Flow Security Improvements
- Token revocation (logout doesn't invalidate tokens)
- Session tracking (no session table for users)
- Account status validation on each request

### Registration Flow Cleanup
- Dual registration endpoints (`/auth/register` vs `/registration/complete`)
- OTP code duplication refactoring
- Liveness score persistence

### Schema Cleanup
- Full removal of PendingRegistration model (marked deprecated only)
- Removal of unused fields in various models

These items require more extensive changes and should be addressed in a separate sprint.
