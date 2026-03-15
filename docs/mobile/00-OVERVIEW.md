# Mobile App — Overview

---

## Tech Stack

| What | Choice |
|------|--------|
| Framework | React Native (Expo managed) |
| Navigation | Expo Router (file-based, native feel) |
| State management | Redux Toolkit (RTK Query for API) |
| Language | TypeScript (strict) |
| Styling | RTL-first, design tokens from Figma |
| Auth storage | expo-secure-store (PIN, tokens) |
| Biometrics | expo-local-authentication |
| Push notifications | expo-notifications + FCM |
| KYC SDK | Sumsub (or pluggable provider) |
| i18n | Arabic (default), Sorani Kurdish, English |

---

## Two User Types

### New Customer
No bank account. Goes through full onboarding → gets a new account.

```
Phone OTP → ID Scan → Selfie + Liveness → Data Entry (6+ sections) → Submit → Wait for Approval → PIN Setup → Home
```

### Existing Customer (Phase 2)
Already has a bank account. Lighter KYC → connects existing account to the app.

```
Phone match + ID scan + Selfie + Card verification → Connect account → PIN Setup → Home
```

---

## App States

```
App Launch
    │
    ├── No token → Login Screen
    │   └── Phone + Password → Selfie + Liveness → Home
    │
    ├── Has token, no PIN set → PIN Setup
    │
    ├── Has token, PIN set → PIN / Biometric unlock → Home
    │
    └── Has onboarding in progress → Resume Onboarding (polling screen if submitted)
```

---

## Feature Map

| Feature | Screens | Details Doc |
|---------|---------|-------------|
| Onboarding | OTP, ID scan, selfie, data entry (6+ forms), submit, polling | [04-ONBOARDING.md](./04-ONBOARDING.md) |
| Auth | Login, PIN setup, PIN unlock, biometric, forgot PIN/password | [05-AUTH.md](./05-AUTH.md) |
| Accounts | Account list, balance, transaction history (date filter) | [06-FEATURES.md](./06-FEATURES.md) |
| Cards | Order card, track order status | [06-FEATURES.md](./06-FEATURES.md) |
| Notifications | In-app center, actionable (deep link to screen) | [06-FEATURES.md](./06-FEATURES.md) |
| Settings | Language, notification prefs, change password, delete user | [06-FEATURES.md](./06-FEATURES.md) |

---

## Key Principles

- **Always online** — no offline mode, data always from core
- **RTL-first** — Arabic is default, LTR is the exception
- **Resumable** — onboarding survives app kill and reinstall (server-side state)
- **Contract-driven** — API client and validations auto-generated from `contracts/`
- **Reusable** — shared components, hooks, and patterns across all features
- **Secure** — no sensitive data on device except encrypted tokens in secure store
