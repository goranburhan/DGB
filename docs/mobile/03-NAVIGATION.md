# Mobile App — Navigation

Expo Router with file-based routing. Native stack transitions.

---

## Route Groups

```
app/
├── _layout.tsx              ← Root: providers, fonts, splash screen
├── index.tsx                ← Entry: checks auth state → redirects
│
├── (auth)/                  ← Stack: no tab bar, no back gesture on login
├── (onboarding)/            ← Stack: step-by-step, custom back behavior
├── (pin)/                   ← Stack: PIN setup/unlock, no back
└── (main)/                  ← Tabs: home, accounts, cards, notifications, settings
```

---

## Navigation Flow

```
App Launch
    │
    ├── index.tsx checks:
    │   ├── Has refresh token in secure store?
    │   │   ├── NO → router.replace('/(auth)/login')
    │   │   └── YES → Has PIN set?
    │   │       ├── NO → router.replace('/(pin)/setup')
    │   │       └── YES → router.replace('/(pin)/unlock')
    │   │
    │   └── Has onboarding in progress? (check server)
    │       └── YES → router.replace('/(onboarding)/polling') or resume step
    │
    ├── After login (phone+pass+selfie) →
    │   ├── First time → /(pin)/setup → /(main)
    │   └── PIN already set → /(main)
    │
    ├── After PIN/biometric unlock → /(main)
    │
    └── After onboarding approved → /(auth)/login
```

---

## Deep Linking

Push notifications and external links route to specific screens.

```
corebridge://                         → index (auth check)
corebridge://accounts/:id             → account detail
corebridge://accounts/:id/transactions → transaction list
corebridge://notifications            → notification center
corebridge://cards/orders             → card orders
corebridge://onboarding/status        → onboarding polling
```

### Push Notification → Deep Link

```typescript
// Notification payload includes:
{
  "data": {
    "route": "/(main)/(accounts)/[id]",
    "params": { "id": "ACC-123" }
  }
}

// On tap → Expo Notifications listener → router.push(route, params)
// If app is locked → unlock first → then navigate
```

---

## Tab Bar

```
(main)/_layout.tsx

Tabs:
┌──────┬──────────┬───────┬───────────────┬──────────┐
│ Home │ Accounts │ Cards │ Notifications │ Settings │
└──────┴──────────┴───────┴───────────────┴──────────┘
```

- Badge on Notifications tab (unread count)
- RTL: tabs render right-to-left in Arabic/Kurdish
- Icons from a consistent icon set (Phosphor, Lucide, or custom)

---

## Onboarding Navigation

Linear step flow with custom back behavior:

```
phone-otp → id-scan → selfie-liveness → data-entry/document-data
    → data-entry/work-data → data-entry/income-data
    → data-entry/location-data → data-entry/bank-usage-data
    → review-submit → polling
```

- **Back button:** goes to previous step (not browser-style history)
- **Cannot skip steps** — enforced by checking `onboarding.currentStep` in slice
- **Polling screen:** no back button, user waits here
- **Resume:** if app killed, `index.tsx` checks server for in-progress onboarding and routes to correct step

---

## Auth Guards

```typescript
// Root _layout.tsx wraps protected routes
// (main) group requires: isAuthenticated && isUnlocked
// (onboarding) group requires: phone verified (or server-side session)
// (pin) group requires: isAuthenticated && !isUnlocked

// If guard fails → redirect to appropriate screen
```

---

## Screen Transitions

| Transition | Animation |
|-----------|-----------|
| Auth → Main | `replace` (no back) |
| Onboarding steps | `push` with slide (native stack) |
| PIN unlock → Main | `replace` (no back) |
| Tab switches | Default tab animation |
| Deep link | `push` on top of current stack |
