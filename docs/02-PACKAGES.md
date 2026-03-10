# Corebridge — Package Breakdown (`@corebridge/*`)

Corebridge is distributed as private npm packages. Each bank's deployment imports these packages and extends them. This enables: independent versioning, AI agents working on isolated modules, and clean customization boundaries.

---

## Package Map

```
@corebridge/
├── contracts          ← OpenAPI spec, JSON schemas, validation rules, error codes
├── core-adapter       ← CoreAdapter module (interfaces + mappers + SOAP client)
├── auth               ← JWT auth, refresh tokens, OTP, selfie verification
├── onboarding         ← Digital onboarding flow, document upload, KYC orchestration
├── kyc                ← KYC provider integration (Sumsub/Regula/GBG)
├── sanctions          ← Sanctions screening integration
├── accounts           ← Balance, transactions, account switching
├── cards              ← Physical card ordering
├── notifications      ← FCM push, in-app center, SMS dispatch
├── backoffice         ← Manual review queues, 360 view, KYC approval
├── audit              ← Audit trail logging, queryable audit history
├── storage            ← Object storage abstraction (local + core dual-write)
├── i18n               ← Arabic, Sorani Kurdish, English translations
├── common             ← Shared types, decorators, guards, interceptors, utils
└── config             ← App configuration, feature flags, environment management
```

---

## Package Details

### `@corebridge/contracts`
Single source of truth. Not a NestJS module — just TypeScript types, JSON schemas, and validation rules. **Separate repo** (`corebridge-contracts`).

Contains: `openapi.yaml`, `schemas/`, `validations/`, `errors/`, `events/`

Consumed by: everything (backend, mobile, backoffice, AI agents)

---

### `@corebridge/core-adapter`
The bridge to ICS BANKS. Single external interface, internally separated.

```
core-adapter/
├── interfaces/
│   ├── customer.interface.ts      ← getCustomer(), createCustomer(), updateCustomer()
│   ├── account.interface.ts       ← getBalance(), getTransactions(), getAccounts()
│   ├── card.interface.ts          ← orderCard(), getCardStatus()
│   ├── document.interface.ts      ← uploadDocument(), getDocument()
│   └── core-adapter.interface.ts  ← Aggregates all interfaces
├── mappers/
│   ├── customer.mapper.ts         ← CIF_NO → customerId, maps all fields
│   ├── account.mapper.ts          ← Core account format → Corebridge format
│   ├── transaction.mapper.ts      ← Core transaction → clean transaction object
│   ├── error.mapper.ts            ← ERR_442 → ACCOUNT_FROZEN
│   └── index.ts
├── soap-client/
│   ├── wsdl/                      ← WSDL files from ICS BANKS
│   ├── generated/                 ← Auto-generated TS clients from WSDLs
│   ├── client.factory.ts          ← Creates SOAP clients with connection pooling
│   ├── circuit-breaker.ts         ← Prevents cascading failures to core
│   ├── retry.strategy.ts          ← Configurable retry with backoff
│   └── request-logger.ts          ← Logs all SOAP requests/responses
└── core-adapter.module.ts         ← NestJS module that wires it all together
```

Owner: Partner (primary). Customization point: Banks override `/mappers` when their ICSFS field names differ.

---

### `@corebridge/auth`
JWT access + refresh tokens, OTP generation/verification, selfie verification on login/password reset, session management, rate limiting, device fingerprinting.

Dependencies: `core-adapter`, `kyc`, `notifications`, `audit`

---

### `@corebridge/onboarding`
Orchestrates the full digital onboarding flow:

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

Flow steps are config-driven. Dependencies: `core-adapter`, `kyc`, `sanctions`, `notifications`, `storage`, `audit`

Customization: Banks add/remove/reorder steps, require additional documents.

---

### `@corebridge/kyc`
Abstraction over KYC providers. Provider-agnostic interface with implementations for Sumsub, Regula, GBG Acuant.

Customization: Config switch to pick provider, no code change.

---

### `@corebridge/sanctions`
Sanctions screening during onboarding and data updates. Provider-agnostic interface with implementations for Dow Jones, Refinitiv, ComplyAdvantage.

Customization: Bank may already have a provider. Plug in theirs or use built-in.

---

### `@corebridge/accounts`
Balance, transactions (cursor paginated), account switching, account type display. All reads from core, no local storage.

---

### `@corebridge/cards`
Physical card ordering — submit to backoffice queue, track status. Intentionally simple. PSP integration is a customization item.

---

### `@corebridge/notifications`
FCM push, SMS (bank's gateway), in-app center (PostgreSQL). Unified `send()` with channel selection and i18n templates.

Customization: SMS gateway is always bank-specific.

---

### `@corebridge/backoffice`
Backend module for backoffice API endpoints: review queues, 360 customer view, KYC approval/rejection, audit log viewer, customer search.

---

### `@corebridge/audit`
Append-only audit trail. Logs every mutation, auth event, and core banking call. Queryable by backoffice. Immutable — no UPDATE/DELETE. Uses `@Auditable()` decorator.

---

### `@corebridge/storage`
Dual-write: core + local object storage (MinIO or filesystem). File type validation. Configurable backend.

---

### `@corebridge/i18n`
Arabic, Sorani Kurdish, English. RTL/LTR utilities, date/number formatting per locale.

---

### `@corebridge/common`
Shared types, `@Auditable()` decorator, auth guards, interceptors, pagination utilities, error handling base classes, health check endpoints.

---

### `@corebridge/config`
Environment variable management, feature flags, bank-specific config schema, defaults.

---

## Dependency Graph

```
contracts (no deps — separate repo)
    ↑
common (contracts)
    ↑
core-adapter (contracts, common)
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
backoffice (core-adapter, kyc, audit, storage)
```

---

## Customization Matrix

| Package | Customizable? | What Changes |
|---------|--------------|--------------|
| contracts | Extended (not modified) | Bank adds endpoints, never removes base ones |
| core-adapter | **Heavily** | Field mappings differ per bank's ICSFS config |
| auth | Rarely | Usually works as-is |
| onboarding | **Heavily** | Steps, required docs, flow order |
| kyc | Provider swap | Pick Sumsub vs Regula vs GBG |
| sanctions | Provider swap | Pick provider or use bank's existing |
| accounts | Rarely | Display formatting at most |
| cards | Moderately | PSP integration is bank-specific |
| notifications | **Heavily** | SMS gateway, templates |
| backoffice | Moderately | Additional screens, custom workflows |
| audit | Never | Must remain unchanged for compliance |
| storage | Config only | Storage backend selection |
| i18n | Extended | Bank-specific terminology |
| common | Never | Shared infrastructure stays standard |
| config | Always | Every bank has different config values |
