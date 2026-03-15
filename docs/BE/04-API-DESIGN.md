# Corebridge — API Design Conventions

---

## URL Structure

```
Base:    /api/v1
Auth:    /api/v1/auth/*
Mobile:  /api/v1/*              (used by mobile app)
Back:    /api/v1/backoffice/*   (used by backoffice app, separate auth role)
```

---

## Endpoints

```
# Auth
POST   /api/v1/auth/register              ← Start onboarding (phone + OTP)
POST   /api/v1/auth/verify-otp            ← Verify OTP code
POST   /api/v1/auth/login                 ← Login with credentials
POST   /api/v1/auth/refresh               ← Refresh access token
POST   /api/v1/auth/logout                ← Invalidate session
POST   /api/v1/auth/forgot-password       ← Initiate password reset
POST   /api/v1/auth/reset-password        ← Complete password reset with selfie
POST   /api/v1/auth/verify-selfie         ← Selfie verification for sensitive ops

# Onboarding
GET    /api/v1/onboarding/status          ← Current onboarding step
POST   /api/v1/onboarding/personal-info   ← Submit personal information
POST   /api/v1/onboarding/documents       ← Upload ID documents
POST   /api/v1/onboarding/selfie          ← Upload selfie + liveness
POST   /api/v1/onboarding/submit          ← Final submission to core
GET    /api/v1/onboarding/result          ← Approval/rejection status

# Accounts
GET    /api/v1/accounts                   ← List customer accounts
GET    /api/v1/accounts/:id               ← Account details + balance
GET    /api/v1/accounts/:id/transactions  ← Transaction history (cursor paginated)

# Cards
POST   /api/v1/cards/order                ← Submit card order
GET    /api/v1/cards/orders               ← List card orders + status

# Notifications
GET    /api/v1/notifications              ← In-app notification center (paginated)
PATCH  /api/v1/notifications/:id/read     ← Mark as read
POST   /api/v1/notifications/read-all     ← Mark all as read

# Settings
GET    /api/v1/settings                   ← User settings (language, notifications)
PATCH  /api/v1/settings                   ← Update settings
POST   /api/v1/account/close              ← Request account closure

# Backoffice
GET    /api/v1/backoffice/queue/onboarding         ← Pending onboarding reviews
GET    /api/v1/backoffice/queue/cards               ← Pending card orders
GET    /api/v1/backoffice/customers/:id             ← 360 customer view
GET    /api/v1/backoffice/customers/:id/documents   ← KYC documents
POST   /api/v1/backoffice/customers/:id/approve     ← Approve onboarding
POST   /api/v1/backoffice/customers/:id/reject      ← Reject with reason
GET    /api/v1/backoffice/customers/search           ← Customer search
GET    /api/v1/backoffice/audit                      ← Audit log (filtered, paginated)
```

---

## Request/Response Format

**Success:**
```json
{
  "success": true,
  "data": { ... },
  "meta": { "timestamp": "2026-03-10T12:00:00Z", "requestId": "uuid" }
}
```

**Paginated (cursor-based):**
```json
{
  "success": true,
  "data": [ ... ],
  "pagination": { "nextCursor": "eyJpZCI6MTAwfQ==", "hasMore": true, "limit": 20 },
  "meta": { "timestamp": "2026-03-10T12:00:00Z", "requestId": "uuid" }
}
```

**Error:**
```json
{
  "success": false,
  "error": { "code": "ONBOARD_001", "message": "ID photo is not clear", "action": "RETRY_UPLOAD", "details": {} },
  "meta": { "timestamp": "2026-03-10T12:00:00Z", "requestId": "uuid" }
}
```

---

## Pagination

Cursor-based for all list endpoints. Cursor is an opaque base64 string — client never parses it, just passes it back. Works with real-time core data where offset would break.

---

## Versioning

URL path versioning (`/api/v1/`). Breaking changes get `/api/v2/` for affected endpoints only.

---

## Swagger / OpenAPI

Every endpoint documented via NestJS Swagger decorators. Generated spec must match `contracts/openapi.yaml`. CI validates drift.

Swagger UI at `/api/docs` in dev/staging. Disabled in production unless opted in.

---

## Headers

```
Authorization: Bearer <access_token>
X-Request-Id: <uuid>              ← Client-generated, for tracing
X-Device-Id: <device_fingerprint> ← Mobile app only
Accept-Language: ar | ku | en     ← Determines response language
```

---

## Rate Limiting

| Endpoint Group | Limit | Window |
|----------------|-------|--------|
| Auth (login, OTP) | 5 requests | per minute |
| Auth (refresh) | 30 requests | per minute |
| Onboarding | 10 requests | per minute |
| Accounts (read) | 60 requests | per minute |
| Backoffice | 120 requests | per minute |

Per-user (by JWT subject).
