# Corebridge — Database Schema (PostgreSQL)

PostgreSQL is NOT the source of truth for banking data. It stores only: audit logs, sessions, notifications, onboarding state, and app configuration.

---

## Schema

```sql
-- ============================================
-- AUDIT (append-only, no UPDATE/DELETE)
-- ============================================
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    action          VARCHAR(100) NOT NULL,
    actor_type      VARCHAR(20) NOT NULL,      -- 'CUSTOMER', 'BACKOFFICE_USER', 'SYSTEM'
    actor_id        VARCHAR(100) NOT NULL,
    target_type     VARCHAR(50),
    target_id       VARCHAR(100),
    details         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_actor ON audit_logs (actor_id, created_at DESC);
CREATE INDEX idx_audit_action ON audit_logs (action, created_at DESC);
CREATE INDEX idx_audit_target ON audit_logs (target_type, target_id, created_at DESC);

-- ============================================
-- SESSIONS
-- ============================================
CREATE TABLE user_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     VARCHAR(100) NOT NULL,
    refresh_token   VARCHAR(500) NOT NULL UNIQUE,
    device_id       VARCHAR(200),
    device_info     JSONB,
    ip_address      INET,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ
);

CREATE INDEX idx_sessions_customer ON user_sessions (customer_id, is_active);
CREATE INDEX idx_sessions_token ON user_sessions (refresh_token) WHERE is_active = TRUE;

-- ============================================
-- ONBOARDING STATE
-- ============================================
CREATE TABLE onboarding_applications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number    VARCHAR(20) NOT NULL,
    current_step    VARCHAR(50) NOT NULL,
    -- Application lifecycle (backoffice visibility)
    status          VARCHAR(30) NOT NULL,      -- 'IN_PROGRESS', 'SUBMITTED', 'APPROVED', 'REJECTED', 'FAILED', 'NEEDS_MANUAL_REVIEW', 'EXPIRED'
    -- Core orchestration state (saga runner only — separate concern from status)
    saga_state      VARCHAR(40),               -- 'STARTED' | 'PRE_CUSTOMER_CREATED' | 'CUSTOMER_CREATED' | 'ACCOUNT_CREATED' | 'FAILED' | 'NEEDS_MANUAL_REVIEW' | 'COMPLETED'
    pre_cif         VARCHAR(100),              -- ICSFS pre-customer reference, written after saga step 1 succeeds, used by step 2 on resume
    personal_info   JSONB,                     -- Encrypted, temporary until sent to core
    documents_ref   JSONB,
    kyc_result      JSONB,
    sanctions_result JSONB,
    core_customer_id VARCHAR(100),
    rejection_reason TEXT,
    reviewed_by     VARCHAR(100),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL       -- Applications expire 90 days after creation if not completed
);

CREATE INDEX idx_onboarding_status ON onboarding_applications (status, created_at DESC);
CREATE INDEX idx_onboarding_phone ON onboarding_applications (phone_number);
-- Prevent two active applications for the same phone number
CREATE UNIQUE INDEX idx_onboarding_unique_active ON onboarding_applications (phone_number)
    WHERE status = 'IN_PROGRESS';

-- ============================================
-- PUSH TOKENS
-- ============================================
CREATE TABLE push_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     VARCHAR(100) NOT NULL,
    device_id       VARCHAR(200) NOT NULL,
    fcm_token       TEXT NOT NULL,
    platform        VARCHAR(10) NOT NULL,      -- 'ios' | 'android'
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (device_id)
);

CREATE INDEX idx_push_tokens_customer ON push_tokens (customer_id) WHERE is_active = TRUE;

-- ============================================
-- NOTIFICATIONS
-- ============================================
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     VARCHAR(100) NOT NULL,
    type            VARCHAR(50) NOT NULL,
    channel         VARCHAR(20) NOT NULL,      -- 'PUSH', 'SMS', 'IN_APP'
    title           JSONB NOT NULL,            -- { "ar": "...", "ku": "...", "en": "..." }
    body            JSONB NOT NULL,
    data            JSONB,
    is_read         BOOLEAN DEFAULT FALSE,
    sent_at         TIMESTAMPTZ,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_customer ON notifications (customer_id, is_read, created_at DESC);

-- ============================================
-- CARD ORDERS
-- ============================================
CREATE TABLE card_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     VARCHAR(100) NOT NULL,
    account_id      VARCHAR(100) NOT NULL,
    card_type       VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    delivery_address JSONB,
    reviewed_by     VARCHAR(100),
    reviewed_at     TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_card_orders_customer ON card_orders (customer_id, created_at DESC);
CREATE INDEX idx_card_orders_status ON card_orders (status, created_at DESC);

-- ============================================
-- APP CONFIGURATION + FEATURE FLAGS
-- ============================================
CREATE TABLE app_config (
    key             VARCHAR(200) PRIMARY KEY,
    value           JSONB NOT NULL,
    description     TEXT,
    updated_by      VARCHAR(100),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE feature_flags (
    key             VARCHAR(200) PRIMARY KEY,
    enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    description     TEXT,
    updated_by      VARCHAR(100),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- BACKOFFICE USERS
-- ============================================
CREATE TABLE backoffice_users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(100) NOT NULL UNIQUE,
    password_hash   VARCHAR(200) NOT NULL,     -- bcrypt
    role            VARCHAR(30) NOT NULL,       -- 'REVIEWER' | 'APPROVER' | 'AUDITOR'
    is_active       BOOLEAN DEFAULT TRUE,
    created_by      VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at   TIMESTAMPTZ
);

-- ============================================
-- LOGIN ATTEMPTS (for lockout tracking)
-- ============================================
CREATE TABLE login_attempts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number    VARCHAR(20) NOT NULL,
    ip_address      INET,
    success         BOOLEAN NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_login_attempts_phone ON login_attempts (phone_number, created_at DESC);

-- ============================================
-- OTP TRACKING
-- ============================================
CREATE TABLE otp_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number    VARCHAR(20) NOT NULL,
    otp_hash        VARCHAR(200) NOT NULL,     -- Never plaintext
    purpose         VARCHAR(30) NOT NULL,
    attempts        INT DEFAULT 0,
    max_attempts    INT DEFAULT 3,
    is_verified     BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_otp_phone ON otp_requests (phone_number, purpose, created_at DESC);
```

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| `customer_id` is VARCHAR, not FK | Customer lives in core, not our DB |
| `personal_info` is encrypted JSONB | Temporary during onboarding, cleared after core submission |
| Audit logs are append-only | DB user has INSERT only, no UPDATE/DELETE |
| Notifications store i18n in JSONB | One row, all languages. Frontend picks the right one |
| OTP stores hash only | Never plaintext |
| No `users` table | User data lives in core |

---

## Data Retention

| Table | Retention | Reason |
|-------|-----------|--------|
| audit_logs | 7 years minimum | CBI compliance |
| user_sessions | 90 days after expiry | Cleanup |
| onboarding_applications | 1 year after completion | Disputes. Expire (status → EXPIRED) 90 days after creation if not completed. Applications with non-null `pre_cif` at expiry flagged for backoffice cleanup. |
| notifications | 6 months | Only show recent |
| card_orders | 2 years | Disputes |
| otp_requests | 24 hours | Short-lived |
| push_tokens | Until deregistered | Deregistered on logout and deactivate-digital-access |
| backoffice_users | Until deactivated | is_active = false on offboarding |
| login_attempts | 90 days | Lockout window calculation |

---

## Encryption

`personal_info` contains PII temporarily. Application-level AES-256-GCM encryption before storing. Encryption key in environment config, not in database. Field cleared after successful core submission.
