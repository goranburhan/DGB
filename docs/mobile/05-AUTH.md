# Mobile App — Authentication

---

## Auth Model

Two layers:

1. **Password** — used when fully logged out (no tokens). Proves identity.
2. **PIN / Biometric** — used when app is backgrounded or reopened. Proves device holder.

Tokens live in `expo-secure-store`. Redux only tracks boolean flags (isAuthenticated, isPINSet, isUnlocked).

---

## Login Flow (Logged Out)

```
User has no tokens (fresh install, or logged out)

1. Enter phone + password
   └── POST /auth/login → server verifies against core
   └── Server returns: { requiresSelfie: true }

2. Selfie + liveness check
   └── KYC SDK opens → selfie captured
   └── POST /auth/verify-selfie → server compares with onboarding selfie
   └── Match → server returns JWT pair (access + refresh)
   └── No match → error, retry selfie

3. Store tokens in expo-secure-store

4. First login ever?
   ├── YES → route to PIN setup → then home
   └── NO → route to home (PIN already set)
```

---

## PIN Setup

```
After first successful login:

1. "Create a PIN" screen
   └── PINInput component → 4-6 digit keypad
   └── User enters PIN

2. "Confirm your PIN" screen
   └── User re-enters PIN
   └── Must match

3. PIN hash stored in expo-secure-store
   └── Never sent to server, never in Redux

4. Ask to enable biometrics (if device supports)
   └── "Use fingerprint/Face ID to unlock?"
   └── YES → enable biometric flag in secure store
   └── NO → skip, PIN only

5. Route to /(main)
```

---

## App Unlock (Backgrounded / Reopened)

```
User has tokens + PIN set, app was backgrounded or killed

1. /(pin)/unlock screen appears

2. Option A: Enter PIN
   └── Compare hash with stored hash
   └── Match → dispatch isUnlocked = true → route to /(main)
   └── 3 wrong attempts → force password login (clear tokens)

3. Option B: Biometric (if enabled)
   └── expo-local-authentication prompt
   └── Success → dispatch isUnlocked = true → route to /(main)
   └── Fail → fall back to PIN entry
```

---

## Session Lifecycle

```
┌────────────────────────────────────────────────┐
│  Access Token (30 min)                          │
│  ├── Valid → API calls work                     │
│  └── Expired → auth middleware auto-refreshes   │
│                                                  │
│  Refresh Token (360 days)                       │
│  ├── Valid → can get new access token           │
│  └── Expired → user must login again (password) │
│                                                  │
│  PIN / Biometric                                 │
│  └── Only gates UI access, not API access       │
│      (tokens remain valid in background)         │
└────────────────────────────────────────────────┘
```

- **No auto-logout timer.** Security is handled by PIN/biometric on app reopen.
- **Token refresh** happens transparently via RTK Query auth middleware.
- **Multi-device allowed.** Each device has its own session. Revoking one doesn't affect others.

---

## Forgot PIN

```
1. User taps "Forgot PIN" on unlock screen
2. Route to password entry screen
3. Enter password → POST /auth/login (re-verify)
4. Success → route to PIN setup (create new PIN)
5. Back to unlock with new PIN
```

---

## Forgot Password

```
1. User taps "Forgot Password" on login screen
2. POST /auth/forgot-password (phone number)
3. OTP sent via SMS
4. Enter OTP → verified
5. Selfie + liveness check (security: prove it's really them)
6. POST /auth/reset-password (new password + selfie result)
7. Success → route to login with new password
```

---

## Secure Storage Layout

```
expo-secure-store keys:
├── access_token          ← JWT access token
├── refresh_token         ← JWT refresh token
├── pin_hash              ← bcrypt hash of PIN
├── biometric_enabled     ← "true" / "false"
├── customer_id           ← for quick lookups without API call
└── device_id             ← device fingerprint (persists across reinstalls if possible)
```

---

## Auth State in Redux

```typescript
interface AuthState {
  isAuthenticated: boolean;    // has valid tokens in secure store
  isPINSet: boolean;           // PIN exists in secure store
  isUnlocked: boolean;         // user passed PIN/biometric this session
  customerId: string | null;
  hasOnboardingInProgress: boolean;
}
```

Tokens are **never** in Redux. Auth slice only tracks flags derived from secure store state.
