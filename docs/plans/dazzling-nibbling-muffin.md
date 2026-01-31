# GX Admin Frontend Cleanup Plan

## Overview
Clean up the admin frontend template to be a focused, enterprise-standard GX Protocol admin application.

## Git Workflow
1. Create `base-temp` branch to preserve current code
2. Work on `dev` branch for cleanup

---

## Phase 1: Git Setup
```bash
cd /home/dev-manazir/prod-blockchain/gx-admin-frontend
git checkout -b base-temp
git push -u origin base-temp
git checkout -b dev
```

---

## Phase 2: Delete Auth Routes

**Remove these directories:**
- `src/app/(main)/auth/v1/` (entire directory)
- `src/app/(main)/auth/v2/` (entire directory)
- `src/app/(main)/auth/_components/register-form.tsx`
- `src/app/(main)/auth/_components/social-auth/` (entire directory)

---

## Phase 3: Delete Demo Pages

**Remove these directories:**
- `src/app/(main)/dashboard/default/` (demo with static JSON)
- `src/app/(main)/dashboard/finance/` (mock charts)
- `src/app/(main)/dashboard/crm/` (mock data)
- `src/app/(main)/dashboard/audit/` (coming soon placeholder)
- `src/app/(main)/dashboard/coming-soon/` (generic placeholder)

---

## Phase 4: Delete Theme Presets

**Remove these files:**
- `src/styles/presets/brutalist.css`
- `src/styles/presets/tangerine.css`
- `src/styles/presets/soft-pop.css`

**Keep:** `src/styles/presets/gx-protocol.css`

---

## Phase 5: Update Configuration Files

### 5.1 `src/app/globals.css`
Remove unused preset imports:
```css
/* DELETE these lines: */
@import "../styles/presets/brutalist.css";
@import "../styles/presets/soft-pop.css";
@import "../styles/presets/tangerine.css";
```

### 5.2 `src/types/preferences/theme.ts`
Remove brutalist, soft-pop, tangerine from `THEME_PRESET_OPTIONS`

### 5.3 `src/styles/presets/gx-protocol.css`
Update with correct GX branding:
- Primary Purple: `#470A60` (not #470A69)
- Secondary Green: `#17A210` (not #009A18)
- Update OKLCH values to match

### 5.4 `src/app/layout.tsx`
Add Assistant font as primary, Inter as fallback

---

## Phase 6: Update Redirects & Routes

### 6.1 `src/app/(external)/page.tsx`
```typescript
// Change: redirect("/dashboard/default")
// To: redirect("/dashboard")
```

### 6.2 `src/app/not-found.tsx`
```typescript
// Change: href="/dashboard/default"
// To: href="/dashboard"
```

### 6.3 `src/components/providers/auth-provider.tsx`
```typescript
// Remove v1/v2 from public routes
const publicRoutes = ["/auth/login", "/auth/mfa", "/unauthorized"];
```

---

## Phase 7: Update Dashboard Home

Ensure `/dashboard/page.tsx` shows stats overview with:
- User stats
- Transaction stats
- Pending approvals count
- System status

---

## Commit Strategy

| # | Scope | Message |
|---|-------|---------|
| 1 | auth | `chore(auth): remove legacy v1 and v2 authentication routes` |
| 2 | dashboard | `chore(dashboard): remove demo pages (default, finance, crm, audit, coming-soon)` |
| 3 | theme | `chore(theme): remove brutalist, tangerine, and soft-pop presets` |
| 4 | routing | `fix(routing): update redirects to /dashboard and clean public routes` |
| 5 | styles | `chore(styles): remove unused preset imports from globals.css` |
| 6 | theme | `feat(theme): update gx-protocol preset with correct brand colors` |
| 7 | typography | `feat(typography): add Assistant font for GX branding` |

---

## Verification Checklist

### Build Verification
- [ ] `npm run lint` passes
- [ ] `npx tsc --noEmit` passes
- [ ] `npm run build` succeeds

### Route Testing
| Route | Expected |
|-------|----------|
| `/` | Redirects to `/dashboard` |
| `/auth/login` | Login page |
| `/dashboard` | Stats overview |
| `/auth/v1/login` | 404 |
| `/dashboard/default` | 404 |
| `/dashboard/finance` | 404 |

### Feature Testing
- [ ] All core features load (Users, Admins, Approvals, Treasury, Supply, Roles, Deployments, Webhooks, Notifications, Security, Settings)
- [ ] All entity accounts load (Business, Government, NPO)
- [ ] Theme shows correct purple (#470A60) and green (#17A210)
- [ ] Light/dark mode works
- [ ] Sidebar navigation works

---

## Files Summary

**Delete (directories):**
- `src/app/(main)/auth/v1/`
- `src/app/(main)/auth/v2/`
- `src/app/(main)/auth/_components/social-auth/`
- `src/app/(main)/dashboard/default/`
- `src/app/(main)/dashboard/finance/`
- `src/app/(main)/dashboard/crm/`
- `src/app/(main)/dashboard/audit/`
- `src/app/(main)/dashboard/coming-soon/`

**Delete (files):**
- `src/app/(main)/auth/_components/register-form.tsx`
- `src/styles/presets/brutalist.css`
- `src/styles/presets/tangerine.css`
- `src/styles/presets/soft-pop.css`

**Modify:**
- `src/app/globals.css` - Remove preset imports
- `src/styles/presets/gx-protocol.css` - Update brand colors
- `src/types/preferences/theme.ts` - Remove preset options
- `src/app/layout.tsx` - Add Assistant font
- `src/app/(external)/page.tsx` - Update redirect
- `src/app/not-found.tsx` - Update link
- `src/components/providers/auth-provider.tsx` - Update public routes
