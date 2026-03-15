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
| JWT expiry | Access: 30 min, Refresh: 30 days (rotated on use; idle >30 days → revoked on next use) |
| OTP expiry | 5 minutes, max 3 attempts |
| Password hashing | bcrypt (12 rounds) |
| Password policy | Min 8 chars, at least 1 letter + 1 digit, top-1000 common passwords rejected |
| Account lockout | 5 failed login attempts → 15-min lockout. 10 further failures → account locked, backoffice unlock required. Tracked in `login_attempts` table. |
| Refresh token rotation | Old token invalidated on each use. Reuse of a rotated token revokes the entire session (theft indicator). |
| PII encryption | AES-256-GCM at application level |
| HTTPS | TLS 1.2+ enforced |
| CORS | Whitelist bank's domains only |
| Rate limiting | Per-user, per-endpoint group |
| Input validation | Contract-driven, all inputs validated |
| SQL injection | TypeORM parameterized queries |
| Audit logging | Every sensitive operation logged |
| Session management | Single active session per device, revocable |
| Device attestation | Play Integrity (Android) + App Attest (iOS) verified at login before JWT is issued. Jailbroken/rooted devices blocked. |
| Certificate pinning | TLS cert pinned in mobile app. Rotation procedure: deploy new pin 30 days before cert expiry, remove old pin after expiry window. |
| Backoffice sessions | Access token: 8h (full workday). No refresh token. Re-authentication required per session. |
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
