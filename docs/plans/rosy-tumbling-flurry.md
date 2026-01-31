# Onboarding Page Redesign Plan

**Target**: `/register` page → Rename to "Onboarding" with 5 glass card account types
**Style**: Formal, professional, subtle luxury, enterprise standard
**Theme**: Consistent glass-morphism matching login page

---

## Current State Analysis

### Existing Implementation
- **File**: `gx-wallet-frontend/app/(auth)/register/page.tsx`
- **Current Title**: "Create Your Account"
- **Current Layout**: Single shadcn Card with 4 vertical stacked buttons
- **Current Options**: Individual, Business, NPO, Government
- **Styling**: Uses `backdrop-blur-sm bg-white/95` (not true glass card)
- **Colors**: Blue, Emerald, Pink, Slate (mixed, not brand-aligned)

### Reference: Login Page Glass Card
- **Component**: `GlassCard` from `components/auth/GlassCard.tsx`
- **Styling**: `bg-white/80 backdrop-blur-xl border-gray-200/50 rounded-3xl shadow-xl`
- **Ambient Glows**: Purple (#470A60/10) top-left, Green (#17A210/10) bottom-right
- **Brand Colors**: Primary Purple `#470A60`, Secondary Green `#17A210`

---

## Design Specifications

### Page Structure
```
┌─────────────────────────────────────────────────────────────────┐
│                         HEADER                                  │
│  GX Logo (centered)                                             │
│  Title: "Onboarding"                                            │
│  Subtitle: "Select your account type to begin"                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    CARD GRID (3+2 Layout)                       │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Individual │  │  Business   │  │  Partner    │             │
│  │  (Citizen)  │  │  Account    │  │  (Banks)    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│        ┌─────────────┐  ┌─────────────┐                        │
│        │  Government │  │  Non-Profit │                        │
│        │ Institution │  │ Organization│                        │
│        └─────────────┘  └─────────────┘                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         FOOTER                                  │
│  "Already have an account? Sign in"                             │
└─────────────────────────────────────────────────────────────────┘
```

### Card Design (Glass Card Selection)

Each card follows this anatomy:
```
┌──────────────────────────────────────────┐
│                                          │
│   [Icon]  24-28px, brand purple          │
│                                          │
│   Title                                  │
│   16px, font-semibold, text-gray-900     │
│                                          │
│   Muted description (2 lines max)        │
│   14px, text-gray-500, leading-relaxed   │
│                                          │
│                                          │
│   ───────────────────────────────────    │
│   [→ Get Started] on hover               │
└──────────────────────────────────────────┘

Dimensions: ~280px wide, ~200px tall (desktop)
Padding: 24px (p-6)
Border-radius: 16px (rounded-2xl)
```

### Card States

**Default State:**
```css
bg-white/75
backdrop-blur-xl
border border-gray-200/50
shadow-md
```

**Hover State:**
```css
bg-white/85
border border-[#470A60]/30
shadow-xl
transform: translateY(-2px)
/* Subtle purple glow */
```

**Active/Selected State (if implementing selection before navigation):**
```css
bg-white/90
border-2 border-[#470A60]
ring-2 ring-[#470A60]/20
```

### Responsive Layout

| Breakpoint | Layout | Card Width |
|------------|--------|------------|
| Mobile (< 640px) | 1 column, full width | 100% |
| Tablet (640-1024px) | 2 columns | ~280px each |
| Desktop (> 1024px) | 3+2 rows (3 top, 2 centered bottom) | ~280px each |

---

## 5 Account Type Cards

### 1. Individual (Citizen)
- **Icon**: `User` from lucide
- **Title**: "Individual"
- **Description**: "Personal wallet for everyday transactions and digital payments"
- **Route**: `/register/individual`
- **Accent**: Default (no special color)

### 2. Business
- **Icon**: `Building2` from lucide
- **Title**: "Business"
- **Description**: "Corporate accounts with multi-signature and approval workflows"
- **Route**: `/register/institution?type=BUSINESS`
- **Accent**: Default

### 3. Partner (Banks & Protocol Builders)
- **Icon**: `Handshake` from lucide
- **Title**: "Partner"
- **Description**: "For banks and institutions building on our protocol as infrastructure"
- **Route**: `/register/institution?type=PARTNER`
- **Accent**: Default
- **Use Case**: Banks operating with GX as central bank, businesses building on protocol APIs

### 4. Government Institution
- **Icon**: `Landmark` from lucide
- **Title**: "Government"
- **Description**: "Official accounts for agencies with compliance and audit features"
- **Route**: `/register/institution?type=GOVERNMENT`
- **Accent**: Default

### 5. Non-Profit Organization (NPO)
- **Icon**: `Heart` from lucide
- **Title**: "Non-Profit"
- **Description**: "Accounts for charities and foundations with impact tracking"
- **Route**: `/register/institution?type=NGO`
- **Accent**: Default

---

## Implementation Plan

### Files to Modify

1. **`gx-wallet-frontend/app/(auth)/register/page.tsx`**
   - Redesign entire page layout
   - Use GlassCard-style containers
   - Implement 5-card grid layout
   - Change title to "Onboarding"
   - Add professional typography hierarchy

2. **`gx-wallet-frontend/lib/icons.ts`** (if needed)
   - Export `Handshake` icon for Partner card

### New/Reusable Components

Consider creating:
- `OnboardingCard` - Reusable glass selection card component
- Or use existing `GlassCard` with custom card content

---

## Code Structure

### Page Component Pseudo-Structure

```tsx
// app/(auth)/register/page.tsx
'use client';

import { GlassCard } from '@/components/auth/GlassCard';

const accountTypes = [
  { id: 'individual', title: 'Individual', description: 'Personal wallet for everyday transactions and digital payments', icon: User, href: '/register/individual' },
  { id: 'business', title: 'Business', description: 'Corporate accounts with multi-signature and approval workflows', icon: Building2, href: '/register/institution?type=BUSINESS' },
  { id: 'partner', title: 'Partner', description: 'For banks and institutions building on our protocol as infrastructure', icon: Handshake, href: '/register/institution?type=PARTNER' },
  { id: 'government', title: 'Government', description: 'Official accounts for agencies with compliance and audit features', icon: Landmark, href: '/register/institution?type=GOVERNMENT' },
  { id: 'npo', title: 'Non-Profit', description: 'Accounts for charities and foundations with impact tracking', icon: Heart, href: '/register/institution?type=NGO' },
];

return (
  <div className="min-h-screen auth-page">
    <main className="min-h-screen flex flex-col items-center justify-center px-4 py-12">

      {/* Header Section */}
      <div className="text-center mb-10">
        <Logo />
        <h1 className="text-3xl font-bold text-[#470A60] mt-4">Onboarding</h1>
        <p className="text-gray-500 mt-2">Select your account type to begin</p>
      </div>

      {/* Cards Grid - 3+2 Layout */}
      <div className="w-full max-w-5xl">
        {/* Top row: 3 cards */}
        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6 justify-items-center">
          {accountTypes.slice(0, 3).map(renderCard)}
        </div>
        {/* Bottom row: 2 cards centered */}
        <div className="flex flex-col sm:flex-row gap-6 justify-center mt-6">
          {accountTypes.slice(3).map(renderCard)}
        </div>
      </div>

      {/* Footer Link */}
      <p className="text-center text-sm text-gray-600 mt-8">
        Already have an account? <Link href="/login">Sign in</Link>
      </p>

    </main>
  </div>
);
```

### Selection Card Component Styling

```tsx
<button
  onClick={() => router.push(option.href)}
  className={cn(
    // Glass effect
    "bg-white/75 backdrop-blur-xl",
    "border border-gray-200/50",
    "rounded-2xl p-6",
    "shadow-md",
    // Hover effects
    "hover:bg-white/85 hover:shadow-xl",
    "hover:border-[#470A60]/30",
    "hover:-translate-y-1",
    // Transitions
    "transition-all duration-300",
    // Layout
    "text-left w-full sm:w-[280px]",
    "group cursor-pointer"
  )}
>
  {/* Icon */}
  <div className="mb-4">
    <Icon className="h-7 w-7 text-[#470A60]" />
  </div>

  {/* Title */}
  <h3 className="font-semibold text-lg text-gray-900 mb-2">
    {title}
  </h3>

  {/* Description */}
  <p className="text-sm text-gray-500 leading-relaxed">
    {description}
  </p>

  {/* Hover indicator */}
  <div className="mt-4 flex items-center text-sm text-[#470A60] opacity-0 group-hover:opacity-100 transition-opacity">
    <span>Get started</span>
    <ArrowRight className="h-4 w-4 ml-1" />
  </div>
</button>
```

---

## Visual Consistency Checklist

- [ ] Use `bg-white/75` to `bg-white/85` (matching login GlassCard)
- [ ] Use `backdrop-blur-xl` (16px blur)
- [ ] Use `border-gray-200/50` (subtle border)
- [ ] Use `rounded-2xl` or `rounded-3xl` (consistent radius)
- [ ] Use `shadow-md` to `shadow-xl` (appropriate depth)
- [ ] Use `#470A60` for primary purple (icons, hover borders)
- [ ] Use `#17A210` for green accents (links)
- [ ] Use Assistant font (via font-brand or default)
- [ ] Subtle hover transitions (300ms ease)
- [ ] No garish per-card colors (unified brand palette)

---

## Verification Plan

### Visual Verification
1. Navigate to `http://localhost:3002/register`
2. Compare glass effect with login page at `/login`
3. Test all 5 cards for hover states
4. Test responsive layouts (mobile, tablet, desktop)
5. Verify all navigation routes work correctly

### Accessibility Check
- [ ] Keyboard navigation (Tab through cards)
- [ ] Focus visible states
- [ ] Color contrast (WCAG AA)
- [ ] Screen reader labels

### Mobile Testing
- [ ] Cards stack properly on mobile
- [ ] Touch targets are adequate (44px+)
- [ ] No horizontal scroll

---

## Questions Clarified

1. **Partner account type** - Banks and institutions building on GX Protocol as infrastructure (central bank relationship)
2. **Card arrangement** - 3+2 layout (3 cards top row, 2 cards centered bottom row)
3. **Colors** - Unified brand purple/green, no per-card color coding
4. **Navigation** - Direct navigation on click (no intermediate selection state)

---

## Files Changed Summary

| File | Action |
|------|--------|
| `app/(auth)/register/page.tsx` | Major rewrite - new layout, glass cards, 5 options |
| `lib/icons.ts` | Add `Handshake` export if not present |

**Estimated scope**: ~150-200 lines of code changes
