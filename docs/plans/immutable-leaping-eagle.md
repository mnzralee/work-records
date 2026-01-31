# Fix KYC Step 2 Documents Upload - 400 Bad Request

## Overview

**Goal:** Fix the 400 Bad Request error on document upload and ensure full consistency from frontend to backend to database.

**Issue:** `POST http://localhost:3000/api/kyc/documents/upload 400 (Bad Request)`

---

## Root Cause Analysis

### The 400 Error Root Cause

**Backend uploadDocumentSchema** (lines 80-86 in `dto/request/index.ts`):
```typescript
export const uploadDocumentSchema = z.object({
  documentType: z.enum(['GOVERNMENT_ID', 'PASSPORT', 'DRIVER_LICENSE', ...]),
  documentSubType: z.string().max(50).optional(),
  fileName: z.string().min(1).max(255),     // REQUIRED - causes 400!
  mimeType: z.enum(['image/jpeg', ...]),    // REQUIRED - causes 400!
});
```

**Frontend sends** (lines 64-68 in `upload/route.ts`):
```typescript
outgoingFormData.append('file', file, file.name);
outgoingFormData.append('documentType', documentType);
outgoingFormData.append('side', side);
// MISSING: fileName, mimeType  <-- This causes 400!
```

**Why it fails:** Zod validation runs on `req.body` BEFORE the controller can extract `fileName`/`mimeType` from the multer file object. The controller (lines 53-54) does extract these, but too late:
```typescript
// This happens AFTER Zod validation fails:
fileName: file.originalname,
mimeType: file.mimetype,
```

### Additional Issues Found

| Issue | Location | Impact |
|-------|----------|--------|
| `fileName` required but extracted from file | Backend DTO | 400 error |
| `mimeType` required but extracted from file | Backend DTO | 400 error |
| `PROOF_OF_ADDRESS` missing from validDocTypes | Frontend route | 400 on PoA save |
| `DRIVERS_LICENSE` vs `DRIVER_LICENSE` naming | Both ends | Type mismatch |
| `side` field sent but not in backend schema | Frontend | Silently ignored |

---

## Implementation Plan

### Phase 1: Backend DTO Schema Fix (Root Cause)

**File:** `gx-protocol-backend/apps/svc-kyc/src/application/dto/request/index.ts`

Make `fileName` and `mimeType` optional since they're extracted from the file by the controller:

```typescript
export const uploadDocumentSchema = z.object({
  documentType: z.enum([
    'GOVERNMENT_ID',
    'PASSPORT',
    'DRIVER_LICENSE',
    'DRIVERS_LICENSE',  // Accept both naming conventions
    'PROOF_OF_ADDRESS',
    'UTILITY_BILL',
    'BANK_STATEMENT',
    'OTHER'
  ]),
  documentSubType: z.string().max(50).optional(),
  fileName: z.string().min(1).max(255).optional(),  // Optional - extracted from file
  mimeType: z.enum(['image/jpeg', 'image/png', 'application/pdf']).optional(),  // Optional
  side: z.enum(['FRONT', 'BACK']).optional(),  // Add side field for front/back tracking
});
```

---

### Phase 2: Frontend Upload Route Enhancement

**File:** `gx-wallet-frontend/app/api/kyc/documents/upload/route.ts`

Add `fileName` and `mimeType` to formData for defense-in-depth (lines 64-68):

```typescript
// Create new FormData for the backend request
const outgoingFormData = new FormData();
outgoingFormData.append('file', file, file.name);
outgoingFormData.append('documentType', documentType);
outgoingFormData.append('side', side);
// Add fileName and mimeType explicitly for schema validation
outgoingFormData.append('fileName', file.name);
outgoingFormData.append('mimeType', file.type);
```

---

### Phase 3: Frontend Document Type Validation Fix

**File:** `gx-wallet-frontend/app/api/kyc/applications/[id]/documents/[docType]/route.ts`

Add missing document types to validDocTypes (line 32):

```typescript
// Validate docType - accept all document types including PROOF_OF_ADDRESS
const validDocTypes = [
  'GOVERNMENT_ID',
  'PASSPORT',
  'DRIVERS_LICENSE',
  'DRIVER_LICENSE',  // Accept backend naming convention too
  'PROOF_OF_ADDRESS'
];
if (!validDocTypes.includes(docType.toUpperCase())) {
  return NextResponse.json(
    { success: false, error: `Invalid document type. Must be one of: ${validDocTypes.join(', ')}` },
    { status: 400 }
  );
}
```

---

### Phase 4: Backend Repository - Handle Front/Back Images

**File:** `gx-protocol-backend/apps/svc-kyc/src/application/ports/repositories/index.ts`

Add `side` to CreateDocumentInput:

```typescript
export interface CreateDocumentInput {
  applicationId: string;
  documentType: 'GOVERNMENT_ID' | 'PASSPORT' | 'DRIVER_LICENSE' | 'PROOF_OF_ADDRESS' | ...;
  documentSubType?: string;
  fileUrl: string;
  fileName: string;
  fileSize: number;
  mimeType: string;
  side?: 'FRONT' | 'BACK';  // NEW: Track which side
}
```

**File:** `gx-protocol-backend/apps/svc-kyc/src/infrastructure/repositories/prisma-kyc-document.repository.ts`

Update create method to handle front/back (around line 76):

```typescript
async create(input: CreateDocumentInput): Promise<KYCDocumentReadModel> {
  // ... existing code ...

  // Determine which image field to populate based on side
  const imageData: { frontImageId?: string; backImageId?: string } = {};
  if (input.side === 'BACK') {
    imageData.backImageId = input.fileUrl;
  } else {
    imageData.frontImageId = input.fileUrl;  // Default to front
  }

  const doc = await this.prisma.kYCApplicationDocument.create({
    data: {
      id: randomUUID(),
      tenantId: app.tenantId,
      applicationId: input.applicationId,
      docType: prismaDocType as any,
      ...imageData,  // Spread front/back image
      skipped: false,
    },
  });
  return this.toReadModel(doc);
}
```

---

### Phase 5: Backend Use Case - Pass Side Parameter

**File:** `gx-protocol-backend/apps/svc-kyc/src/application/use-cases/documents/index.ts`

Update UploadDocumentUseCase.execute() to pass side:

```typescript
const document = await this.documentRepository.create({
  applicationId: application.applicationId,
  documentType: dto.documentType,
  documentSubType: dto.documentSubType,
  fileUrl: uploadResult.fileUrl,
  fileName: uploadResult.fileName,
  fileSize: uploadResult.fileSize,
  mimeType: uploadResult.mimeType,
  side: dto.side,  // NEW: Pass side information
});
```

---

### Phase 6: Frontend UI/UX Mobile-First Improvements

**File:** `gx-wallet-frontend/app/onboarding/kyc/documents/page.tsx`

1. **Error Handling Enhancement:**
   - Show specific error messages for each validation failure
   - Add retry button for failed uploads
   - Show upload progress indicator

2. **Mobile-First Responsive Design:**
   - Tabs: Use horizontal scroll on small screens or convert to dropdown
   - Upload areas: Stack vertically on mobile, side-by-side on tablet+
   - Buttons: Full-width on mobile, inline on larger screens
   - Image previews: Smaller thumbnails on mobile

3. **Data Persistence:**
   - Save draft state to localStorage
   - Pre-fill from saved documents when returning to step
   - Show "Saved" indicator for completed documents

---

## Files to Modify

| Priority | File | Change |
|----------|------|--------|
| P0 | `backend/apps/svc-kyc/src/application/dto/request/index.ts` | Make fileName/mimeType optional, add side |
| P0 | `frontend/app/api/kyc/documents/upload/route.ts` | Add fileName/mimeType to formData |
| P1 | `frontend/app/api/kyc/applications/[id]/documents/[docType]/route.ts` | Add PROOF_OF_ADDRESS to validDocTypes |
| P1 | `backend/apps/svc-kyc/src/application/ports/repositories/index.ts` | Add side to CreateDocumentInput |
| P1 | `backend/apps/svc-kyc/src/infrastructure/repositories/prisma-kyc-document.repository.ts` | Handle front/back images |
| P1 | `backend/apps/svc-kyc/src/application/use-cases/documents/index.ts` | Pass side to repository |
| P2 | `frontend/app/onboarding/kyc/documents/page.tsx` | Mobile-first UI improvements |

---

## Verification Plan

### 1. Test Upload Endpoint
```bash
# Test GOVERNMENT_ID front image upload
curl -X POST http://localhost:3042/api/v1/kyc/documents \
  -H "Authorization: Bearer <token>" \
  -F "file=@test-id.jpg" \
  -F "documentType=GOVERNMENT_ID" \
  -F "side=FRONT"
```

### 2. Test Document Save
```bash
# Test saving document metadata
curl -X PUT http://localhost:3042/api/v1/kyc/applications/<id>/documents/GOVERNMENT_ID \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"documentNumber": "ABC123", "frontImageId": "file-id"}'
```

### 3. End-to-End Test
1. Login to wallet frontend
2. Start KYC application, complete Step 1
3. Upload Government ID (front + back)
4. Upload Proof of Address
5. Navigate away and back - verify data persists
6. Complete Step 2, verify Step 3 is accessible

### 4. Mobile Responsiveness
- Test on 320px, 375px, 414px, 768px, 1024px widths
- Verify all touch targets are at least 44px
- Verify text is readable without zooming

---

## Execution Order

1. **Backend DTO fix** - Make fileName/mimeType optional (fixes 400 error)
2. **Frontend upload route** - Add fileName/mimeType to formData
3. **Frontend docType validation** - Add PROOF_OF_ADDRESS
4. **Backend repository interface** - Add side field
5. **Backend repository** - Handle front/back images
6. **Backend use case** - Pass side to repository
7. **Build and deploy backend** - Test upload works
8. **Frontend UI/UX** - Mobile-first improvements
9. **Test end-to-end**
10. **Commit and deploy**

---

*Plan updated: 2026-01-22*
*Focus: Fix 400 Bad Request on document upload*
