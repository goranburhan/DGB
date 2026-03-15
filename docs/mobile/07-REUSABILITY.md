# Mobile App — Reusability & Patterns

---

## Component Layers

```
┌─────────────────────────────────────────┐
│  Screen (app/ route files)              │  ← Thin, composes everything below
│  Max ~150 lines. No business logic.     │
├─────────────────────────────────────────┤
│  Hooks (src/hooks/)                     │  ← Business logic, state, side effects
├─────────────────────────────────────────┤
│  Components (src/components/)           │  ← Pure UI, props-driven
│  ├── ui/       ← Design system          │
│  ├── forms/    ← Form inputs            │
│  ├── layout/   ← Screen wrappers        │
│  └── feedback/ ← Status & errors        │
├─────────────────────────────────────────┤
│  Theme (src/theme/)                     │  ← Design tokens, no logic
└─────────────────────────────────────────┘
```

---

## Screen Pattern

Every screen follows the same structure:

```typescript
// app/(main)/(accounts)/index.tsx
import { Screen } from '@components/layout/Screen';
import { useGetAccountsQuery } from '@api/endpoints/accounts.api';
import { AccountCard } from './components/AccountCard';
import { ErrorBanner } from '@components/feedback/ErrorBanner';
import { Spinner } from '@components/ui/Spinner';
import { EmptyState } from '@components/feedback/EmptyState';

export default function AccountsScreen() {
  const { data, isLoading, error, refetch } = useGetAccountsQuery();

  if (isLoading) return <Screen><Spinner /></Screen>;
  if (error) return <Screen><ErrorBanner error={error} onRetry={refetch} /></Screen>;
  if (!data?.length) return <Screen><EmptyState message="accounts.empty" /></Screen>;

  return (
    <Screen>
      {data.map(account => (
        <AccountCard key={account.id} account={account} />
      ))}
    </Screen>
  );
}
```

Rules:
- Screen files are thin composers — no inline styles, no API calls, no complex logic
- All data fetching via RTK Query hooks
- All business logic in custom hooks
- All UI from shared components

---

## Shared UI Components

### Screen

```typescript
// Wraps every screen: SafeArea + ScrollView + KeyboardAvoiding + RTL direction
<Screen
  scrollable={true}           // default true
  keyboardAware={true}        // default true for form screens
  padding="default"           // uses spacing tokens
  header={<Header title="accounts.title" />}
>
  {children}
</Screen>
```

### Button

```typescript
<Button
  label="onboarding.next"     // i18n key
  variant="primary"            // primary | secondary | danger | text
  onPress={handleNext}
  loading={isSubmitting}
  disabled={!isValid}
  fullWidth                    // default true
/>
```

### FormField

```typescript
// Label + Input + Error message, RTL-aware
<FormField
  label="onboarding.fullName"  // i18n key
  value={name}
  onChangeText={setName}
  error={errors.fullName}      // from validation
  required
/>
```

### OTPInput / PINInput

```typescript
// Auto-focus, auto-advance, paste support
<OTPInput
  length={6}
  onComplete={handleVerify}
  error={otpError}
/>

<PINInput
  length={4}
  onComplete={handlePINEntry}
  biometricAvailable={hasBiometric}
  onBiometricPress={handleBiometric}
/>
```

---

## Custom Hooks

### useForm

```typescript
// Generic form hook — works with any validation schema from contracts
const { values, errors, handleChange, handleSubmit, isValid, isSubmitting } = useForm({
  schema: documentDataSchema,   // from generated validations
  initialValues: prefilled,     // OCR data for document form
  onSubmit: async (data) => {
    dispatch(setDocumentData(data));
    router.push('/(onboarding)/data-entry/work-data');
  },
});
```

### useAuth

```typescript
const {
  login,           // phone + password → selfie → tokens
  logout,          // clear everything, route to login
  refreshToken,    // called by middleware, transparent
  isAuthenticated,
} = useAuth();
```

### useOnboarding

```typescript
const {
  currentStep,
  goToNextStep,
  goToPreviousStep,
  formData,
  updateFormData,
  submitApplication,
  resumeOnboarding,  // check server, route to correct step
  applicationStatus,
} = useOnboarding();
```

### useBiometric

```typescript
const {
  isAvailable,     // device supports biometric
  isEnabled,       // user opted in
  authenticate,    // trigger prompt
  enable,          // opt in
  disable,         // opt out
} = useBiometric();
```

---

## Form Validation Pattern

```
contracts/validations/          ← Source of truth (JSON)
    │
    ▼ (code generation)
src/validations/generated/      ← TypeScript validation schemas
    │
    ▼ (useForm hook)
FormField component             ← Shows errors inline, i18n messages
```

- Validations are **never** hardcoded in components
- Same rules on mobile and backend (generated from same source)
- Error messages are i18n keys — `FormField` resolves them to current language

---

## RTL Pattern

```typescript
// useRTL hook
const { isRTL, direction, flipStyle } = useRTL();

// Components use it internally:
// - Text alignment flips
// - Flex direction flips
// - Padding/margin flips
// - Icons that indicate direction (arrows, chevrons) flip

// Theme provides RTL-aware spacing:
// paddingStart / paddingEnd instead of paddingLeft / paddingRight
```

All components are RTL by default. LTR is handled as the exception.

---

## Error Handling Pattern

```
API error (RTK Query rejection)
    │
    ├── 401 → auth middleware handles (refresh or logout)
    │
    ├── Known error code (from contracts/errors/)
    │   └── ErrorBanner shows localized message + action
    │       e.g., "ONBOARD_001" → "ID photo is not clear" → "Retry Upload" button
    │
    └── Unknown error
        └── Generic error message + "Try Again"
```

---

## File Naming

| Type | Convention | Example |
|------|-----------|---------|
| Route files | kebab-case | `phone-otp.tsx`, `id-scan.tsx` |
| Components | PascalCase | `Button.tsx`, `FormField.tsx` |
| Hooks | camelCase with `use` prefix | `useAuth.ts`, `useForm.ts` |
| Slices | camelCase with `.slice` suffix | `auth.slice.ts` |
| API endpoints | camelCase with `.api` suffix | `accounts.api.ts` |
| Types | camelCase with `.types` suffix | `api.types.ts` |
| Utils | camelCase | `formatting.ts`, `errors.ts` |
