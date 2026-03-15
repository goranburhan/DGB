# Mobile App — Folder Structure

Expo managed workflow with file-based routing (Expo Router).

---

## Project Root

```
apps/mobile/
├── app/                            ← Expo Router (file-based routes)
│   ├── _layout.tsx                 ← Root layout (providers, fonts, splash)
│   ├── index.tsx                   ← Entry redirect (check auth state → route)
│   │
│   ├── (auth)/                     ← Auth group (no tab bar)
│   │   ├── _layout.tsx
│   │   ├── login.tsx
│   │   ├── forgot-password.tsx
│   │   ├── reset-password.tsx
│   │   └── selfie-verify.tsx
│   │
│   ├── (onboarding)/               ← Onboarding group (no tab bar)
│   │   ├── _layout.tsx
│   │   ├── phone-otp.tsx
│   │   ├── id-scan.tsx
│   │   ├── selfie-liveness.tsx
│   │   ├── data-entry/
│   │   │   ├── document-data.tsx
│   │   │   ├── work-data.tsx
│   │   │   ├── income-data.tsx
│   │   │   ├── location-data.tsx
│   │   │   ├── bank-usage-data.tsx
│   │   │   └── [step].tsx          ← Dynamic for future extra steps
│   │   ├── review-submit.tsx
│   │   └── polling.tsx
│   │
│   ├── (pin)/                      ← PIN/Biometric group
│   │   ├── _layout.tsx
│   │   ├── setup.tsx               ← First-time PIN creation
│   │   └── unlock.tsx              ← PIN entry or biometric
│   │
│   └── (main)/                     ← Main app (with tab bar)
│       ├── _layout.tsx             ← Tab navigator layout
│       ├── (home)/
│       │   ├── _layout.tsx
│       │   └── index.tsx           ← Accounts overview / balance
│       ├── (accounts)/
│       │   ├── _layout.tsx
│       │   ├── index.tsx           ← Account list
│       │   └── [id]/
│       │       ├── index.tsx       ← Account detail + balance
│       │       └── transactions.tsx
│       ├── (cards)/
│       │   ├── _layout.tsx
│       │   ├── index.tsx           ← Card orders list
│       │   └── order.tsx           ← Order new card
│       ├── (notifications)/
│       │   ├── _layout.tsx
│       │   └── index.tsx           ← Notification center
│       └── (settings)/
│           ├── _layout.tsx
│           ├── index.tsx           ← Settings menu
│           ├── language.tsx
│           ├── notification-prefs.tsx
│           ├── change-password.tsx
│           └── delete-account.tsx
│
├── src/
│   ├── api/
│   │   ├── generated/              ← Auto-generated typed API client from contracts
│   │   ├── client.ts               ← Axios instance with auth interceptor
│   │   └── endpoints/              ← RTK Query endpoint definitions
│   │       ├── auth.api.ts
│   │       ├── onboarding.api.ts
│   │       ├── accounts.api.ts
│   │       ├── cards.api.ts
│   │       ├── notifications.api.ts
│   │       └── settings.api.ts
│   │
│   ├── store/
│   │   ├── index.ts                ← Store configuration
│   │   ├── hooks.ts                ← Typed useAppSelector, useAppDispatch
│   │   ├── middleware.ts           ← Custom middleware (auth, error handling)
│   │   └── slices/
│   │       ├── auth.slice.ts       ← Auth state (token, user, PIN status)
│   │       ├── onboarding.slice.ts ← Onboarding progress, form data
│   │       ├── ui.slice.ts         ← Loading states, modals, toasts
│   │       └── app.slice.ts        ← Language, feature flags, app config
│   │
│   ├── components/
│   │   ├── ui/                     ← Pure UI primitives (design system)
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Text.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Toast.tsx
│   │   │   ├── Spinner.tsx
│   │   │   ├── Badge.tsx
│   │   │   ├── Divider.tsx
│   │   │   ├── Avatar.tsx
│   │   │   └── index.ts           ← Barrel export
│   │   ├── forms/                  ← Form-specific components
│   │   │   ├── FormField.tsx       ← Label + input + error (RTL-aware)
│   │   │   ├── PhoneInput.tsx      ← +964 prefix, numeric
│   │   │   ├── OTPInput.tsx        ← 4-6 digit code input
│   │   │   ├── PINInput.tsx        ← PIN dots + keypad
│   │   │   ├── DatePicker.tsx
│   │   │   ├── Dropdown.tsx
│   │   │   └── DocumentScanner.tsx ← Wraps KYC SDK
│   │   ├── layout/                 ← Layout components
│   │   │   ├── Screen.tsx          ← SafeArea + scroll + keyboard aware
│   │   │   ├── Header.tsx
│   │   │   ├── TabBar.tsx
│   │   │   └── StepIndicator.tsx   ← Onboarding progress bar
│   │   └── feedback/               ← Status / feedback components
│   │       ├── ErrorBanner.tsx
│   │       ├── EmptyState.tsx
│   │       ├── PollingStatus.tsx   ← Animated waiting screen
│   │       └── StatusBadge.tsx
│   │
│   ├── hooks/
│   │   ├── useAuth.ts              ← Login, logout, token refresh, biometric check
│   │   ├── usePIN.ts               ← PIN setup, verify, forgot
│   │   ├── useBiometric.ts         ← Check availability, authenticate
│   │   ├── useOnboarding.ts        ← Step navigation, progress, resume
│   │   ├── useForm.ts              ← Form state, validation, submission
│   │   ├── useNotifications.ts     ← Push registration, permission, deep link
│   │   ├── useRTL.ts               ← RTL detection, flip styles
│   │   └── useSecureStore.ts       ← Wrapper around expo-secure-store
│   │
│   ├── theme/
│   │   ├── tokens.ts               ← Imports design/tokens/*.json
│   │   ├── colors.ts
│   │   ├── typography.ts
│   │   ├── spacing.ts
│   │   ├── shadows.ts
│   │   └── index.ts                ← Unified theme export
│   │
│   ├── i18n/
│   │   ├── index.ts                ← i18n setup (i18next or similar)
│   │   ├── ar.json                 ← Arabic translations
│   │   ├── ku.json                 ← Sorani Kurdish translations
│   │   └── en.json                 ← English translations
│   │
│   ├── validations/
│   │   ├── generated/              ← Auto-generated from contracts/validations/
│   │   └── helpers.ts              ← Validation runner, error formatting
│   │
│   ├── utils/
│   │   ├── formatting.ts           ← Currency, date, number formatting (locale-aware)
│   │   ├── navigation.ts           ← Deep link helpers, route constants
│   │   ├── errors.ts               ← Error code lookup from contracts, user-facing messages
│   │   └── constants.ts            ← App-wide constants
│   │
│   └── types/
│       ├── api.types.ts            ← API response/request types (from generated)
│       ├── navigation.types.ts     ← Route param types
│       └── common.types.ts         ← Shared app types
│
├── design/                         ← Figma exports (tokens + screen specs)
│   ├── tokens/
│   │   ├── colors.json
│   │   ├── typography.json
│   │   ├── spacing.json
│   │   └── shadows.json
│   ├── screens/
│   │   ├── onboarding/
│   │   ├── auth/
│   │   ├── accounts/
│   │   ├── cards/
│   │   ├── notifications/
│   │   └── settings/
│   └── components/
│
├── assets/
│   ├── fonts/
│   ├── images/
│   └── icons/
│
├── app.json                        ← Expo config
├── eas.json                        ← EAS Build config
├── tsconfig.json
├── babel.config.js
└── package.json
```

---

## Key Conventions

| Convention | Rule |
|-----------|------|
| Routes | `app/` directory only — Expo Router file-based |
| Business logic | `src/hooks/` and `src/store/slices/` — never in route files |
| API calls | `src/api/endpoints/` via RTK Query — never raw fetch in components |
| UI components | `src/components/ui/` — pure, no business logic, design-token-driven |
| Form components | `src/components/forms/` — reusable form inputs with built-in validation |
| Validation | `src/validations/generated/` — auto-generated, never hardcoded |
| Translations | `src/i18n/*.json` — all user-facing text, no hardcoded strings |
| Sensitive storage | `expo-secure-store` only — never AsyncStorage for tokens/PIN |
| Screen files | Thin — compose from components + hooks, max ~150 lines |

---

## Import Aliases

```json
// tsconfig.json paths
{
  "@api/*": ["src/api/*"],
  "@store/*": ["src/store/*"],
  "@components/*": ["src/components/*"],
  "@hooks/*": ["src/hooks/*"],
  "@theme/*": ["src/theme/*"],
  "@i18n/*": ["src/i18n/*"],
  "@utils/*": ["src/utils/*"],
  "@types/*": ["src/types/*"],
  "@validations/*": ["src/validations/*"]
}
```
