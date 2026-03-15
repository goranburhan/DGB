# Mobile App — Features

---

## Accounts

### Account List
- Fetch from core via `GET /accounts`
- Shows all customer accounts (current, savings, etc.)
- Each card shows: account type, account number (masked), balance
- Tap → account detail

### Account Detail
- Balance (real-time from core)
- Account info (type, number, status, currency)
- "View Transactions" button

### Transactions
- `GET /accounts/:id/transactions` — cursor paginated
- Filter by date range (date picker)
- Each row: date, description, amount (+/-), running balance
- Infinite scroll with cursor pagination
- Pull-to-refresh

---

## Transfers

### Transfer Form
- Select source account
- Select beneficiary type: own account / same-bank customer / inter-bank
- Enter amount (IQD)
- Optional: note / description

### Pre-Check
- `POST /transfers/pre-check` → returns `{ requiresSelfie, fee, limits }`
- Shows fee and transfer limits before submission
- If `requiresSelfie: true` → opens selfie verification flow (via `POST /security/verify-selfie` with `context: 'transfer'`) before allowing submission

### Submit
- `POST /transfers/submit` — includes optional `selfieToken` if selfie was completed
- On success: confirmation screen + push notification
- On failure: localized error from core (insufficient funds, account frozen, etc.)

### Transfer History
- Accessible from Account Detail screen
- `GET /transfers/history` — cursor-paginated, same row format as transactions
- Filter by date range

---

## Cards

### Card Orders List
- `GET /cards/orders` — list of all card orders
- Each row: card type, status (pending, approved, shipped, delivered), date
- Status badge (color-coded)

### Order Card
- `POST /cards/order`
- Select account, card type, delivery address
- Confirmation screen → submit
- Queued for backoffice review
- Push notification on status change

---

## Notifications

### Notification Center
- `GET /notifications` — paginated
- Grouped: unread on top, then read
- Each row: icon, title, body preview, time ago
- Tap → mark as read + deep link to relevant screen
- "Mark all as read" action

### Actionable Notifications
Each notification has an optional `route` in its data. On tap:

```
Notification types:
├── Onboarding approved     → /(auth)/login
├── Onboarding rejected     → /(onboarding)/polling (shows rejection)
├── Card order status        → /(main)/(cards)
├── Transaction alert        → /(main)/(accounts)/[id]/transactions
└── General announcement     → /(main)/(notifications)
```

### Push Notifications
- Register FCM token on login via `expo-notifications`
- Handle notification received (foreground): show in-app toast
- Handle notification tap (background/killed): deep link to route
- If app is locked → unlock first → then navigate

---

## Settings

### Language
- Switch between Arabic, Sorani Kurdish, English
- Updates `i18n` locale + app direction (RTL/LTR)
- Persisted locally + sent to server (`PATCH /settings`)

### Notification Preferences
- Toggle push notifications on/off
- Toggle SMS notifications on/off
- Per-category toggles (if needed later)

### Change Password
- Current password + new password + confirm
- `POST /auth/reset-password`
- On success → toast confirmation

### Deactivate Digital Access
- "Remove app access" → confirmation dialog
- Explains: "This removes your digital banking access. Your bank account remains open. To close your bank account, visit your nearest branch."
- Confirm → `POST /api/v1/settings/deactivate-digital-access`
- Server: revokes all sessions, deregisters FCM token, logs action to audit
- Client: clears all local storage → routes to login
- Bank account and all balances untouched in core

### FCM Token Lifecycle
- Registered on login: `POST /notifications/register-token` (sends `{ fcmToken, deviceId, platform }`)
- Refreshed automatically via `expo-notifications` token refresh listener (FCM can rotate tokens without notice)
- Deregistered on logout and on deactivate-digital-access
