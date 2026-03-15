# Corebridge — Module Structure

No npm package distribution. Everything lives in one Nx monorepo as local libs.

---

## Module Map

```
libs/
├── core-adapter/       ← ICSFS SOAP integration (interfaces + mappers + client)
├── auth/               ← JWT, refresh tokens, OTP, selfie verification
├── onboarding/         ← Digital onboarding flow, document upload, KYC orchestration
├── kyc/                ← KYC provider integration (Sumsub/Regula/GBG)
├── sanctions/          ← Sanctions screening
├── accounts/           ← Balance, transactions, account switching
├── cards/              ← Physical card ordering
├── transfers/          ← Fund transfers (pre-check, submit, history)
├── notifications/      ← FCM push, in-app center, SMS dispatch
├── backoffice/         ← Review queues, 360 view, KYC approval
├── audit/              ← Append-only audit trail
├── storage/            ← Object storage abstraction (MinIO + core dual-write)
├── i18n/               ← Arabic, Sorani Kurdish, English
└── common/             ← Shared types, decorators, guards, interceptors, utils
```

---

## Module Details

### `core-adapter`
Bridge to ICS BANKS. Clean interfaces for domain services, SOAP details hidden inside.

```
core-adapter/
├── interfaces/         ← customer, account, card, document interfaces
├── mappers/            ← Corebridge models ←→ ICSFS models, error normalization
├── soap-client/        ← WSDL-generated clients, connection pooling, circuit breaker, retry
└── core-adapter.module.ts
```

### `auth`
JWT access + refresh tokens, OTP generation/verification, selfie verification, session management, rate limiting, device fingerprinting.

Dependencies: `core-adapter`, `kyc`, `notifications`, `audit`

### `onboarding`
Full digital onboarding flow:

```
Start → Phone verification (OTP)
     → Personal info entry
     → National ID upload (front + back)
     → Selfie + liveness check
     → Sanctions screening
     → Submit to core (create customer/account)
     → Queue for backoffice review
     → Approved / Rejected notification
```

Dependencies: `core-adapter`, `kyc`, `sanctions`, `notifications`, `storage`, `audit`

### `kyc`
Abstraction over KYC providers. Provider-agnostic interface. Pick Sumsub, Regula, or GBG via config (`KYC_PROVIDER`, `KYC_PROVIDER_DEPLOYMENT_MODE: cloud | on-prem`).

**Circuit breaker:** If the KYC provider returns errors on 100 consecutive requests, the circuit opens. Onboarding halts at the selfie step with a "try again later" error. Ops alerted via Prometheus alert rule. Circuit resets after 60s cooldown.

### `sanctions`
Sanctions screening during onboarding submission. Provider-agnostic API calls — configure via `SANCTIONS_API_KEY` and `SANCTIONS_PROVIDER_URL`. Screens applicant full name + national ID number against configured lists (UN Security Council, OFAC SDN, EU Consolidated, and any CBI-mandated domestic list). Matching algorithm: exact + fuzzy (score threshold configurable). Hit → application flagged for manual review, not auto-rejected.

### `accounts`
Balance, transactions (cursor paginated), account switching. All reads from core, no local storage.

### `cards`
Physical card ordering — submit to backoffice queue, track status.

### `transfers`
Fund transfers — pre-check (returns selfie requirement, fee, limits), submit (executes via core), transfer history. Selfie authorization rule applied here (250K IQD threshold / first-receiver-today logic).

Dependencies: `core-adapter`, `audit`, `notifications`

Phase 1 scope.

### `notifications`
FCM push, SMS (bank's gateway), in-app center (PostgreSQL). Unified `send()` with channel selection and i18n templates.

### `backoffice`
Backend endpoints for backoffice app: review queues, 360 customer view, KYC approval/rejection, audit log viewer, customer search.

### `audit`
Append-only audit trail. Logs every mutation, auth event, and core call. Immutable — no UPDATE/DELETE. `@Auditable()` decorator.

### `storage`
Dual-write: core + local object storage (MinIO). File type validation.

### `i18n`
Arabic, Sorani Kurdish, English. RTL/LTR utilities, date/number formatting per locale.

### `common`
Shared types, `@Auditable()` decorator, auth guards, interceptors, pagination utilities, error handling base classes, health check endpoints.

---

## Dependency Graph

```
common (no deps)
    ↑
core-adapter (common)
    ↑
audit (common)
    ↑
┌────┼──────────┬──────────┬──────────┐
│    │          │          │          │
kyc  sanctions  storage   notifications
│    │          │          │
└────┼──────────┴──────────┘
     ↑
onboarding (core-adapter, kyc, sanctions, storage, notifications, audit)
auth (core-adapter, kyc, notifications, audit)
accounts (core-adapter, audit)
cards (core-adapter, notifications, audit)
transfers (core-adapter, audit, notifications)
backoffice (core-adapter, kyc, audit, storage)
```
