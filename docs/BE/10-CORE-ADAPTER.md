# Corebridge — Core Adapter (ICS BANKS Integration)

This is the highest-risk module in the system. Everything downstream depends on it. This doc covers architecture, failure handling, and the impedance mismatch between our REST world and ICSFS SOAP.

---

## 1. What Core Adapter Does

```
Domain Services (clean TypeScript)
        │
        │ calls clean interfaces
        ▼
┌──────────────────────────────────────────────────┐
│                  CORE ADAPTER                     │
│                                                   │
│  ┌─────────────┐  ┌──────────┐  ┌─────────────┐ │
│  │  Interfaces  │  │ Mappers  │  │ SOAP Client │ │
│  │  (contract)  │→ │  (ACL)   │→ │  (transport)│ │
│  └─────────────┘  └──────────┘  └─────────────┘ │
│                                        │          │
│  ┌─────────────┐  ┌──────────────────┐ │          │
│  │  Saga        │  │ Circuit Breaker  │ │          │
│  │  Orchestrator│  │ + Retry + Pool   │ │          │
│  └─────────────┘  └──────────────────┘ │          │
└────────────────────────────┬─────────────────────┘
                             │ SOAP/HTTPS
                             ▼
                    ICS BANKS Core (ICSFS)
```

Domain services never see SOAP, WSDLs, XML, or ICSFS field names. They call typed interfaces. Core adapter handles everything else.

---

## 2. Layer Breakdown

### Interfaces (Contract Layer)

What domain services depend on. Pure TypeScript interfaces — no implementation details.

```typescript
// interfaces/customer.interface.ts
interface ICoreCustomerService {
  getCustomer(customerId: string): Promise<CoreCustomer>;
  createPreCustomer(data: PreCustomerData): Promise<PreCustomerResult>;
  createCustomer(data: CustomerData): Promise<CustomerResult>;
  updateCustomer(customerId: string, data: Partial<CustomerData>): Promise<void>;
  checkCustomerExists(phoneNumber: string): Promise<CustomerExistsResult>;
}

// interfaces/account.interface.ts
interface ICoreAccountService {
  getAccounts(customerId: string): Promise<CoreAccount[]>;
  getBalance(accountId: string): Promise<Balance>;
  getTransactions(accountId: string, params: TransactionParams): Promise<TransactionPage>;
  createAccount(customerId: string, data: AccountData): Promise<AccountResult>;
}

// interfaces/card.interface.ts
interface ICoreCardService {
  orderCard(data: CardOrderData): Promise<CardOrderResult>;
  getCardStatus(orderId: string): Promise<CardStatus>;
}

// interfaces/transaction.interface.ts
interface ICoreTransactionService {
  preCheckTransfer(data: TransferPreCheckData): Promise<TransferPreCheckResult>;  // returns requiresSelfie
  submitTransfer(data: TransferData): Promise<TransferResult>;
}

// interfaces/document.interface.ts
interface ICoreDocumentService {
  uploadDocument(customerId: string, doc: DocumentUpload): Promise<DocumentRef>;
  getDocument(customerId: string, docId: string): Promise<DocumentData>;
}
```

These interfaces are the only thing the rest of the app knows about. If ICSFS changes, only mappers and SOAP client change — interfaces stay stable.

### Mappers (Anti-Corruption Layer)

Translates between Corebridge domain models and ICSFS models. This is where the impedance mismatch gets handled.

```
Corebridge model          Mapper              ICSFS model
─────────────────     ──────────────     ─────────────────
customerId        →   CIF_NO
fullName          →   CUST_FULL_NAME
dateOfBirth       →   BIRTH_DATE (format: YYYYMMDD)
accountId         →   ACCT_NO
balance           ←   AVAIL_BAL (string → number)
currency          ←   ACCT_CCY
transactionDate   ←   TXN_DATE (format: YYYYMMDD → ISO 8601)
errorCode         ←   ERR_442 → ACCOUNT_FROZEN
```

Mappers handle:
- **Field renaming** — our names ↔ ICSFS names
- **Type conversion** — ICSFS strings → our typed models (dates, numbers, enums)
- **Format conversion** — ICSFS date formats (YYYYMMDD) → ISO 8601
- **Error normalization** — ICSFS error codes → our error codes (from contracts)
- **Null/missing handling** — ICSFS may return empty strings instead of null
- **Currency formatting** — ICSFS stores amounts as strings with varying decimal formats

```typescript
// mappers/customer.mapper.ts
class CustomerMapper {
  toCore(data: CustomerData): ICSFSCustomerRequest {
    return {
      CUST_FULL_NAME: data.fullName,
      BIRTH_DATE: formatICSDate(data.dateOfBirth),
      MOTHER_NAME: data.motherName,
      // ... all field mappings
    };
  }

  fromCore(icsfs: ICSFSCustomerResponse): CoreCustomer {
    return {
      customerId: icsfs.CIF_NO,
      fullName: icsfs.CUST_FULL_NAME,
      dateOfBirth: parseICSDate(icsfs.BIRTH_DATE),
      // ... all field mappings
    };
  }
}
```

### SOAP Client (Transport Layer)

Handles raw SOAP communication with ICSFS.

```
SOAP Client
├── wsdl/                    ← WSDL files from the bank
├── generated/               ← Auto-generated TS clients from WSDLs (wsdl-gen tool)
├── client.factory.ts        ← Creates SOAP clients with connection pooling
├── connection-pool.ts       ← Manages persistent SOAP connections
├── circuit-breaker.ts       ← Prevents cascading failures
├── retry.strategy.ts        ← Configurable retry with backoff
├── timeout.config.ts        ← Per-operation timeout settings
└── request-logger.ts        ← Logs all SOAP requests/responses (sanitized)
```

---

## 3. Failure Handling

### Timeouts

ICSFS is on-prem, often slow, sometimes unresponsive. Every SOAP call has a timeout.

| Operation Type | Timeout | Rationale |
|---------------|---------|-----------|
| Read (balance, transactions, customer lookup) | 5s | Should be fast, fail early |
| Write (create customer, create account) | 15s | Core processing takes longer |
| Document upload | 30s | Large payloads |
| Health check | 3s | Must be fast for monitoring |

If timeout fires:
- Read operations → return error to client, no retry (data might have changed)
- Write operations → **do not assume failure** — see saga section below

### Circuit Breaker

Prevents flooding a struggling core with requests.

```
States:
┌────────┐    5 failures     ┌────────┐    30s cooldown    ┌───────────┐
│ CLOSED │ ──────────────── → │  OPEN  │ ──────────────── → │ HALF-OPEN │
│ (OK)   │ ← ──────────────  │ (FAIL) │ ← ──────────────  │ (TESTING) │
└────────┘  success in        └────────┘  failure in        └───────────┘
            half-open                     half-open
```

- **CLOSED** — normal operation, requests pass through
- **OPEN** — core is down, all requests immediately fail with `CORE_UNAVAILABLE` error (no SOAP call made)
- **HALF-OPEN** — allow 1 request through to test if core recovered

Config:
- Failure threshold: 5 consecutive failures → open
- Cooldown: 30 seconds before trying again
- Per-service: separate circuit breakers for customer, account, card, document operations

When circuit is open, API returns:
```json
{
  "success": false,
  "error": {
    "code": "CORE_UNAVAILABLE",
    "message": "Banking services are temporarily unavailable. Please try again shortly.",
    "action": "RETRY_LATER"
  }
}
```

### Retry Strategy

Not all operations are retryable.

| Operation | Retryable | Strategy |
|-----------|-----------|----------|
| Get balance | Yes | 2 retries, 1s backoff |
| Get transactions | Yes | 2 retries, 1s backoff |
| Get customer | Yes | 2 retries, 1s backoff |
| Create pre-customer | **No** — see saga | — |
| Create customer | **No** — see saga | — |
| Create account | **No** — see saga | — |
| Upload document | Yes (idempotent by ref) | 3 retries, 2s backoff |
| Order card | **No** | — |

Write operations that change core state are **not blindly retried**. A timeout doesn't mean the operation failed — core might have processed it. Retrying could create duplicates. These go through the saga orchestrator.

---

## 4. Saga Orchestrator (Multi-Step Transactions)

### The Problem

Onboarding creates a customer in core through multiple SOAP calls:

```
Step 1: Create pre-customer   → returns preCIF
Step 2: Create customer        → returns CIF (uses preCIF)
Step 3: Create account         → returns accountId (uses CIF)
```

These are 3 separate SOAP calls. ICSFS has no distributed transactions. If step 2 succeeds but step 3 fails, you have a customer with no account.

### The Solution: Saga Pattern

Each multi-step core operation is orchestrated as a saga with compensating actions.

```
┌──────────────────────────────────────────────────────────────┐
│                     ONBOARDING SAGA                           │
│                                                               │
│  Step 1: createPreCustomer(data)                              │
│  ├── Success → store preCIF, continue                         │
│  ├── Failure → mark application FAILED, stop                  │
│  └── Timeout → query core "does this pre-customer exist?"     │
│               ├── YES → continue with returned preCIF         │
│               └── NO  → retry step 1 (safe, doesn't exist)   │
│                                                               │
│  Step 2: createCustomer(preCIF, data)                         │
│  ├── Success → store CIF, continue                            │
│  ├── Failure → log, mark application FAILED, stop             │
│  │            (pre-customer is inert, no compensation needed)  │
│  └── Timeout → query core "does customer exist for          │
│               national ID AND phone number?"                   │
│               ├── Both match → continue with returned CIF     │
│               ├── National ID match, phone differs →          │
│               │   transition to NEEDS_MANUAL_REVIEW, stop     │
│               │   (may be a branch-created customer record)    │
│               └── No match → retry step 2                     │
│                                                               │
│  Step 3: createAccount(CIF, accountData)                      │
│  ├── Success → store accountId, saga complete                 │
│  ├── Failure → log, mark application NEEDS_MANUAL_REVIEW      │
│  │            (customer exists but no account — backoffice     │
│  │             must resolve manually or retry)                 │
│  └── Timeout → query core "does account exist for CIF?"       │
│               ├── YES → continue, saga complete                │
│               └── NO  → retry step 3                          │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Key Principles

1. **Verify before retry.** On timeout, always check if the operation actually succeeded before retrying. Query core for existence.
2. **Idempotency where possible.** Use phone number / national ID as natural keys for existence checks.
3. **Compensating actions are limited.** ICSFS likely doesn't support "delete customer." So if step 2 succeeds but step 3 fails, you can't roll back — you escalate to backoffice.
4. **Every step is logged.** Audit trail records each saga step with its result for debugging and compliance.
5. **Manual resolution queue.** If a saga gets stuck in a partial state, it goes to a backoffice queue for manual resolution.

### Saga State Machine

```
┌─────────┐     ┌──────────────┐     ┌──────────────┐     ┌───────────┐
│ STARTED │ ──→ │ PRE_CUSTOMER │ ──→ │   CUSTOMER   │ ──→ │  ACCOUNT  │
│         │     │   CREATED    │     │   CREATED    │     │  CREATED  │
└─────────┘     └──────────────┘     └──────────────┘     └───────────┘
     │                │                    │                     │
     ▼                ▼                    ▼                     ▼
  FAILED           FAILED          NEEDS_MANUAL_REVIEW      COMPLETED
```

Saga state is stored in `onboarding_applications` across two separate columns:
- `saga_state` — tracks orchestration progress (saga runner reads/writes this)
- `status` — tracks application lifecycle for backoffice visibility (separate concern)

`pre_cif` is written to `onboarding_applications.pre_cif` immediately after step 1 succeeds. The saga reads this column on resume — no need to re-query core for the pre-customer reference.

---

## 5. Transfer Saga

`submitTransfer` is a write operation against core and follows the same verify-before-retry pattern as onboarding.

```
Transfer Saga:
│
├── POST /transfers/submit
│   └── Call core: executeTransfer(amount, fromAccount, toAccount, reference)
│       ├── Success → return transfer result to client, log to audit
│       ├── Failure → return error to client (core rejected the transfer)
│       └── Timeout → query core: "did transfer with this reference succeed?"
│           ├── YES → return success to client (transfer went through)
│           └── NO  → safe to retry (transfer was not processed)
```

A unique `reference` ID (generated by Corebridge before calling core) is passed with every transfer call. This reference is the idempotency key for the timeout recovery query.

---

## 6. Impedance Mismatch: REST ↔ SOAP

### Problem

Our API is REST with JSON. ICSFS is SOAP with XML. The mismatch goes deeper than protocol:

| Our API (REST) | ICSFS (SOAP) |
|----------------|-------------|
| JSON | XML |
| HTTP status codes (404, 400, 500) | Always HTTP 200, error in XML body |
| Typed error codes | String error codes like `ERR_442` |
| Cursor-based pagination | No pagination concept (returns all or fixed page) |
| ISO 8601 dates | `YYYYMMDD` strings |
| Null = field absent | Empty string = field absent |
| Decimal numbers | String amounts (`"1000.500"`) |
| UTF-8 everywhere | Potential encoding issues with Arabic |

### How Core Adapter Bridges This

**Error mapping:** ICSFS returns errors inside the SOAP response body, always HTTP 200. Core adapter:
1. Parses the XML response
2. Checks for ICSFS error fields
3. Maps ICSFS error code → Corebridge error code (from `contracts/errors/`)
4. Throws typed exception that NestJS exception filters handle

```typescript
// ICSFS returns:
// <ErrorCode>ERR_442</ErrorCode>
// <ErrorDesc>Account is frozen</ErrorDesc>

// Core adapter maps to:
throw new CoreException({
  code: 'ACCOUNT_FROZEN',
  httpStatus: 403,
  originalError: 'ERR_442',
  message: 'Account is frozen',
});
```

**Pagination bridge:** ICSFS may not support cursor pagination natively. Core adapter:
1. Requests transactions with date range from core
2. Internally manages cursor ↔ ICSFS query parameters
3. Returns cursor-paginated response to our API
4. Cursor encodes: last transaction date + ID for next page

**Date/Number conversion:** All done in mappers. ICSFS `"20260315"` → our `"2026-03-15T00:00:00Z"`. ICSFS `"1000.500"` → our `1000.5` (number).

**Arabic text:** Mappers validate and normalize Arabic text encoding from core responses. ICSFS may return Windows-1256 or other encodings — mappers ensure UTF-8.

---

## 7. Connection Pooling

SOAP connections to ICSFS are expensive to create. Connection pool manages reuse.

```
┌──────────────────────────────┐
│       Connection Pool         │
│                               │
│  Min connections: 5           │
│  Max connections: 20          │
│  Idle timeout: 60s            │
│  Connection lifetime: 300s    │
│                               │
│  ┌────┐ ┌────┐ ┌────┐       │
│  │conn│ │conn│ │conn│ ...    │
│  └────┘ └────┘ └────┘       │
└──────────────────────────────┘
```

- Pre-warms minimum connections on startup
- Health-checks idle connections periodically
- Rotates connections after max lifetime (prevents stale connections)
- If pool exhausted → queue requests (up to a limit, then reject with `CORE_BUSY`)

---

## 8. Observability

Every SOAP call is logged and metered:

```
Request log (sanitized — no PII):
{
  "operation": "GetCustomerByCIF",
  "duration_ms": 342,
  "status": "success",
  "soap_action": "urn:GetCustomerByCIF",
  "correlation_id": "req-uuid-123"
}

Metrics (Prometheus):
- core_adapter_request_duration_seconds (histogram, by operation)
- core_adapter_request_total (counter, by operation + status)
- core_adapter_circuit_breaker_state (gauge, by service)
- core_adapter_connection_pool_active (gauge)
- core_adapter_connection_pool_idle (gauge)
- core_adapter_saga_step_total (counter, by step + result)
```

Grafana dashboards show:
- Core response time distribution (p50, p95, p99)
- Error rate by operation
- Circuit breaker state timeline
- Connection pool utilization
- Saga success/failure/manual-review rates

---

## 9. Folder Structure

```
libs/core-adapter/
├── src/
│   ├── interfaces/
│   │   ├── customer.interface.ts
│   │   ├── account.interface.ts
│   │   ├── transaction.interface.ts
│   │   ├── card.interface.ts
│   │   ├── document.interface.ts
│   │   └── index.ts
│   ├── mappers/
│   │   ├── customer.mapper.ts
│   │   ├── account.mapper.ts
│   │   ├── transaction.mapper.ts
│   │   ├── card.mapper.ts
│   │   ├── error.mapper.ts
│   │   ├── date.utils.ts
│   │   ├── amount.utils.ts
│   │   └── index.ts
│   ├── soap-client/
│   │   ├── wsdl/                   ← Bank's WSDL files
│   │   ├── generated/              ← Auto-generated TS clients
│   │   ├── client.factory.ts
│   │   ├── connection-pool.ts
│   │   ├── circuit-breaker.ts
│   │   ├── retry.strategy.ts
│   │   ├── timeout.config.ts
│   │   └── request-logger.ts
│   ├── sagas/
│   │   ├── saga.orchestrator.ts    ← Base saga runner
│   │   ├── onboarding.saga.ts     ← Onboarding multi-step saga
│   │   └── saga.types.ts
│   ├── exceptions/
│   │   ├── core.exception.ts       ← Typed core exceptions
│   │   └── core-exception.filter.ts ← NestJS exception filter
│   └── core-adapter.module.ts      ← NestJS module wiring
└── index.ts
```

---

## 10. What's TBD (Needs Bank's WSDLs)

| Item | Blocked On |
|------|-----------|
| Exact ICSFS operation names | Bank's WSDL files |
| Field mappings | Bank's ICSFS configuration |
| Error code mapping (full list) | Bank's ICSFS error documentation |
| Pagination behavior | How their ICSFS handles transaction queries |
| Pre-customer → Customer flow specifics | Bank's onboarding procedures in core |
| Connection/auth to ICSFS | Their SOAP endpoint URL, credentials, certificates |
| Arabic encoding specifics | Testing against their actual responses |
