# Mobile App — Onboarding

---

## Flow

```
1. Phone OTP
   └── Enter phone → receive SMS → enter code → verified

2. ID Scan (KYC SDK — Sumsub or similar)
   └── Camera opens → scan national ID front + back → OCR extracts data

3. Selfie + Liveness
   └── Camera opens → take selfie → liveness check → match against ID photo

4. Data Entry (6+ sections, multi-screen)
   ├── 4a. Document Data (pre-filled from OCR)
   │   └── Full name, DOB, father name, mother name, expiry date, issue date, location, etc.
   ├── 4b. Work Data
   │   └── Employment status, employer, job title, sector, etc.
   ├── 4c. Income & Source of Funds
   │   └── Monthly income, income source, source of funds, etc.
   ├── 4d. Location
   │   └── Current address, city, governorate, etc.
   ├── 4e. Bank Usage
   │   └── Purpose of account, expected transaction volume, etc.
   └── 4f. (More sections TBD)

5. Review & Submit
   └── User reviews all data → submit → application sent to backend → queued for backoffice

6. Polling / Waiting
   └── Animated waiting screen → polls server → push notification on decision
   └── Approved → redirect to login
   └── Rejected → show reason → "Try Again" button → restart from step 1
```

---

## Step State Machine

```
PHONE_OTP → ID_SCAN → SELFIE_LIVENESS → DATA_DOCUMENT → DATA_WORK
    → DATA_INCOME → DATA_LOCATION → DATA_BANK_USAGE → REVIEW → SUBMITTED
    → APPROVED / REJECTED
```

Server tracks `current_step` and `status`. Mobile reads from server on resume.

---

## Resume Logic

User can close app at any point and come back:

```
App launch → GET /onboarding/status
    │
    ├── No application → start fresh (phone OTP)
    ├── IN_PROGRESS → route to current_step
    ├── SUBMITTED → route to polling screen
    ├── MANUAL_REVIEW_REQUIRED → route to polling screen (awaiting manual backoffice review)
    ├── APPROVED → route to login
    └── REJECTED → route to rejection screen (with retry)
```

This works even after app reinstall — state is server-side, identified by phone number + device.

---

## KYC SDK Failure Fallback

If the KYC SDK fails to produce a valid result after **2 attempts** (degraded document, poor camera, liveness check failed twice):

```
1. App exits the KYC SDK
2. User is shown a manual entry screen:
   └── Upload any image of the document (no liveness required)
   └── Fill in document data manually (no OCR pre-fill)
3. Application is flagged MANUAL_REVIEW_REQUIRED in backend
4. Backoffice manually reviews uploaded image + submitted data before approval
```

This ensures customers with degraded documents or older hardware are not permanently blocked from onboarding. All manual-entry applications go to a dedicated backoffice queue.

---

## KYC SDK Integration

The ID scan and selfie/liveness steps use a third-party SDK (Sumsub or similar).

```
┌─────────────────┐     ┌─────────────────┐     ┌──────────────┐
│   Corebridge     │────►│   KYC SDK        │────►│  KYC Server  │
│   Mobile App     │◄────│   (native view)  │◄────│  (Sumsub)    │
└─────────────────┘     └─────────────────┘     └──────────────┘
        │                                               │
        │           OCR data + match result             │
        └───────────────── backend ◄────────────────────┘
```

- SDK opens a native camera view (not our UI)
- On completion, SDK returns: OCR data, face match score, liveness result
- Backend validates the result with the KYC provider's server
- Mobile receives: extracted document data (for pre-filling forms) + pass/fail

---

## Data Entry Forms

Each data entry screen is a form backed by:
- **Validation rules** from `contracts/validations/` (auto-generated)
- **Redux slice** stores partial form data (survives step navigation)
- **Shared `FormField` component** renders each field (RTL-aware, error display)

### Document Data (pre-filled from OCR)

| Field | Type | Pre-filled | Editable |
|-------|------|-----------|----------|
| Full name (Arabic) | text | Yes (OCR) | Yes |
| Full name (English) | text | Yes (OCR) | Yes |
| Date of birth | date | Yes (OCR) | Yes |
| Father name | text | Yes (OCR) | Yes |
| Mother name | text | No | Yes |
| National ID number | text | Yes (OCR) | No (locked) |
| ID issue date | date | Yes (OCR) | No (locked) |
| ID expiry date | date | Yes (OCR) | No (locked) |
| Place of birth | text | Yes (OCR) | Yes |
| Gender | dropdown | Yes (OCR) | Yes |

### Work Data

| Field | Type |
|-------|------|
| Employment status | dropdown (employed, self-employed, student, retired, unemployed) |
| Employer name | text (conditional: if employed) |
| Job title | text (conditional: if employed) |
| Sector | dropdown |

### Income & Source of Funds

| Field | Type |
|-------|------|
| Monthly income range | dropdown (ranges) |
| Source of income | dropdown (salary, business, investment, etc.) |
| Source of funds | dropdown |
| Expected monthly deposits | dropdown (ranges) |

### Location

| Field | Type |
|-------|------|
| Governorate | dropdown |
| City/District | dropdown (depends on governorate) |
| Neighborhood | text |
| Street | text |
| Nearest landmark | text |

### Bank Usage

| Field | Type |
|-------|------|
| Purpose of account | multi-select (salary, savings, business, transfers, etc.) |
| Expected transaction frequency | dropdown |
| Will receive international transfers | boolean |

---

## Polling Screen

After submission, user sees a waiting screen:

- Animated illustration (processing / reviewing)
- Status text: "Your application is being reviewed"
- Polls `GET /onboarding/result` every 30 seconds
- Also listens for push notification (faster than polling)
- No back button — user can only close app
- On return to app → same polling screen (server says SUBMITTED)

---

## Rejection & Retry

If rejected:
- Show reason from backoffice (e.g., "ID photo unclear", "Information mismatch")
- "Try Again" button → clears onboarding slice → starts from step 1
- Backend creates a new `onboarding_application` for the same phone number
- Previous application kept for audit

---

## API Endpoints Used

```
POST /api/v1/auth/register          ← Send OTP
POST /api/v1/auth/verify-otp        ← Verify OTP
GET  /api/v1/onboarding/status      ← Current step (for resume)
POST /api/v1/onboarding/personal-info  ← Submit all form data
POST /api/v1/onboarding/documents   ← Upload documents (from KYC SDK)
POST /api/v1/onboarding/selfie      ← Upload selfie result
POST /api/v1/onboarding/submit      ← Final submission
GET  /api/v1/onboarding/result      ← Poll for decision
```
