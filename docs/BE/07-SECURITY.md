# Corebridge — Security & Performance

---

## Authentication Flow

```
Mobile App                     API                        Core
    │                           │                           │
    ├── POST /auth/login ──────►│                           │
    │   (phone + password)      ├── Verify customer ───────►│
    │                           │◄── Customer exists ───────┤
    │                           ├── Generate OTP            │
    │                           ├── Send OTP via SMS ──────►│ (bank's gateway)
    │◄── OTP sent ──────────────┤                           │
    │                           │                           │
    ├── POST /auth/verify-otp ─►│                           │
    │   (OTP code)              ├── Verify OTP hash         │
    │                           ├── Generate JWT pair       │
    │                           ├── Store session in DB     │
    │◄── { accessToken,         │                           │
    │      refreshToken } ──────┤                           │
    │                           │                           │
    ├── GET /accounts ─────────►│                           │
    │   (Bearer accessToken)    ├── Validate JWT            │
    │                           ├── Get accounts ──────────►│
    │                           │◄── Account data ──────────┤
    │◄── { accounts } ─────────┤                           │
```

---

## Security Measures

| Measure | Implementation |
|---------|---------------|
| JWT expiry | Access: 30 min, Refresh: 360 days |
| OTP expiry | 5 minutes, max 3 attempts |
| Password hashing | bcrypt (12 rounds) |
| PII encryption | AES-256-GCM at application level |
| HTTPS | TLS 1.2+ enforced |
| CORS | Whitelist bank's domains only |
| Rate limiting | Per-user, per-endpoint group |
| Input validation | Contract-driven, all inputs validated |
| SQL injection | TypeORM parameterized queries |
| Audit logging | Every sensitive operation logged |
| Session management | Single active session per device, revocable |
| Selfie verification | Dynamic, compliance-driven. Required on: (1) password reset, (2) transactions where amount ≥ 250K IQD, (3) transactions to a receiver not paid today. Skipped for low-value familiar transfers (< 250K IQD to a receiver already paid today). |

---

## Performance Targets

- 10K-30K concurrent requests per minute
- < 500ms for read operations
- < 2s for core-dependent operations

---

## Core SOAP Bottleneck Mitigations

- **Connection pooling** in SOAP client
- **Circuit breaker** (fail fast if core is down)
- **Request timeout** (5s default, configurable)
- **No caching of banking data** (compliance: always real-time)
- **Async where possible** (onboarding submission queued)

---

## NestJS Scaling

- 2+ API instances behind load balancer
- Stateless design (JWT, no server-side sessions in memory)
- PostgreSQL for session storage (Redis optional for high traffic)
- Node.js cluster mode if single-machine

---

## Database Performance

- Cursor-based pagination (no OFFSET)
- Proper indexes on all query patterns
- Audit log partitioned by month after 1 year
- Connection pooling via `pg-pool` (20 connections default)
