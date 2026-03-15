# Mobile App — State Management

Redux Toolkit with RTK Query for all server state. No mixing patterns.

---

## Architecture

```
┌─────────────────────────────────────────────┐
│                 Redux Store                  │
│                                              │
│  ┌──────────────┐  ┌──────────────────────┐ │
│  │   Slices      │  │    RTK Query APIs     │ │
│  │               │  │                       │ │
│  │  auth         │  │  authApi              │ │
│  │  onboarding   │  │  onboardingApi        │ │
│  │  ui           │  │  accountsApi          │ │
│  │  app          │  │  cardsApi             │ │
│  │               │  │  notificationsApi     │ │
│  │               │  │  settingsApi          │ │
│  └──────────────┘  └──────────────────────┘ │
│                                              │
│  ┌──────────────────────────────────────────┐│
│  │  Middleware                               ││
│  │  - RTK Query middleware                   ││
│  │  - Auth middleware (token refresh, 401)   ││
│  │  - Error middleware (global error toast)  ││
│  └──────────────────────────────────────────┘│
└─────────────────────────────────────────────┘
```

---

## What Goes Where

| State Type | Where | Example |
|-----------|-------|---------|
| Server data (API responses) | RTK Query cache | Accounts, transactions, notifications |
| Auth state | `auth.slice.ts` | Token presence, PIN configured, user info |
| Onboarding progress | `onboarding.slice.ts` | Current step, form data across steps |
| UI state | `ui.slice.ts` | Loading overlays, toast messages, modal visibility |
| App config | `app.slice.ts` | Current language, feature flags |
| Sensitive values | `expo-secure-store` (NOT Redux) | JWT tokens, refresh token, PIN hash |

---

## Store Setup

```typescript
// src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import { setupListeners } from '@reduxjs/toolkit/query';
import { authApi } from '@api/endpoints/auth.api';
import { onboardingApi } from '@api/endpoints/onboarding.api';
import { accountsApi } from '@api/endpoints/accounts.api';
import { cardsApi } from '@api/endpoints/cards.api';
import { notificationsApi } from '@api/endpoints/notifications.api';
import { settingsApi } from '@api/endpoints/settings.api';
import authReducer from './slices/auth.slice';
import onboardingReducer from './slices/onboarding.slice';
import uiReducer from './slices/ui.slice';
import appReducer from './slices/app.slice';
import { authMiddleware } from './middleware';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    onboarding: onboardingReducer,
    ui: uiReducer,
    app: appReducer,
    [authApi.reducerPath]: authApi.reducer,
    [onboardingApi.reducerPath]: onboardingApi.reducer,
    [accountsApi.reducerPath]: accountsApi.reducer,
    [cardsApi.reducerPath]: cardsApi.reducer,
    [notificationsApi.reducerPath]: notificationsApi.reducer,
    [settingsApi.reducerPath]: settingsApi.reducer,
  },
  middleware: (getDefault) =>
    getDefault()
      .concat(authApi.middleware)
      .concat(onboardingApi.middleware)
      .concat(accountsApi.middleware)
      .concat(cardsApi.middleware)
      .concat(notificationsApi.middleware)
      .concat(settingsApi.middleware)
      .concat(authMiddleware),
});

setupListeners(store.dispatch);

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

---

## Typed Hooks

```typescript
// src/store/hooks.ts
import { useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './index';

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

---

## Slice Examples

### Auth Slice

```typescript
// src/store/slices/auth.slice.ts
interface AuthState {
  isAuthenticated: boolean;
  isPINSet: boolean;
  isUnlocked: boolean;          // app is unlocked (PIN/biometric passed)
  customerId: string | null;
  hasOnboardingInProgress: boolean;
}

// Tokens stored in expo-secure-store, NOT in Redux
```

### Onboarding Slice

```typescript
// src/store/slices/onboarding.slice.ts
interface OnboardingState {
  currentStep: OnboardingStep;
  phoneNumber: string | null;
  phoneVerified: boolean;
  idScanned: boolean;
  selfieCompleted: boolean;
  formData: {
    documentData: Partial<DocumentDataForm>;
    workData: Partial<WorkDataForm>;
    incomeData: Partial<IncomeDataForm>;
    locationData: Partial<LocationDataForm>;
    bankUsageData: Partial<BankUsageDataForm>;
  };
  applicationId: string | null;
  status: 'idle' | 'in_progress' | 'submitted' | 'approved' | 'rejected';
}
```

---

## RTK Query Pattern

```typescript
// src/api/endpoints/accounts.api.ts
import { createApi } from '@reduxjs/toolkit/query/react';
import { baseQuery } from '../client';

export const accountsApi = createApi({
  reducerPath: 'accountsApi',
  baseQuery,
  tagTypes: ['Accounts', 'Transactions'],
  endpoints: (builder) => ({
    getAccounts: builder.query<AccountsResponse, void>({
      query: () => '/accounts',
      providesTags: ['Accounts'],
    }),
    getAccountDetail: builder.query<AccountDetail, string>({
      query: (id) => `/accounts/${id}`,
      providesTags: (result, error, id) => [{ type: 'Accounts', id }],
    }),
    getTransactions: builder.query<TransactionsResponse, TransactionsParams>({
      query: ({ accountId, cursor, dateFrom, dateTo }) => ({
        url: `/accounts/${accountId}/transactions`,
        params: { cursor, dateFrom, dateTo },
      }),
      providesTags: ['Transactions'],
    }),
  }),
});

export const {
  useGetAccountsQuery,
  useGetAccountDetailQuery,
  useGetTransactionsQuery,
} = accountsApi;
```

---

## Auth Middleware

Handles 401 responses globally — attempts token refresh, if refresh fails → force logout.

```typescript
// src/store/middleware.ts
// Intercepts all RTK Query rejections with status 401
// 1. Pause the failed request
// 2. Attempt token refresh via /auth/refresh
// 3. If refresh succeeds → retry original request
// 4. If refresh fails → dispatch logout, clear secure store, redirect to login
```

---

## Rules

- **Never** put API calls in components directly — always RTK Query
- **Never** store tokens or PIN in Redux — always `expo-secure-store`
- **Never** duplicate server state in slices — RTK Query cache is the source
- Onboarding form data lives in the slice so it survives navigation between steps
- If the app is killed, onboarding state is recovered from the server (not Redux)
- Keep slices flat — no deeply nested state
