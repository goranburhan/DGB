# Corebridge — Monitoring & Alerting

Logs are not enough. Failures must be visible, categorized, and pushed to the right people.

---

## 1. Stack

```
┌──────────────────────────────────────────────────────────┐
│                     DATA COLLECTION                       │
│                                                           │
│  NestJS App ──► Prometheus (metrics scraping, every 15s)  │
│  NestJS App ──► Loki (structured logs, via Promtail)      │
└──────────────────────┬────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                     VISUALIZATION                         │
│                                                           │
│  Grafana                                                  │
│  ├── Dashboards (real-time metrics)                       │
│  ├── Log explorer (Loki queries)                          │
│  └── Alert rules (evaluate every 1-5 min)                 │
└──────────────────────┬────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                     ALERTING                              │
│                                                           │
│  Grafana Alerting ──► Contact Points:                     │
│  ├── Telegram (bot → group chat)                          │
│  ├── Email (SMTP)                                         │
│  └── Slack or Teams (webhook)                             │
│                                                           │
│  Each alert has: severity, description, affected system,  │
│  current value, threshold, runbook link                   │
└──────────────────────────────────────────────────────────┘
```

Why this stack:

- **Prometheus + Grafana + Loki** — already in the deployment (06-DEPLOYMENT.md). No new infra.
- **Grafana Alerting** — built into Grafana, supports Telegram/Email/Slack/Teams natively. No extra service.
- **No backoffice development needed.** Grafana is the dashboard. Add/edit alerts via Grafana UI, not code.

---

## 2. What Gets Monitored

### Core Adapter (Highest Priority)

| Metric                              | Alert When                         | Severity |
| ----------------------------------- | ---------------------------------- | -------- |
| `core_request_duration_seconds` p95 | > 5s for 3 min                     | Warning  |
| `core_request_duration_seconds` p95 | > 10s for 3 min                    | Critical |
| `core_request_errors_total` rate    | > 10% error rate for 5 min         | Critical |
| `core_circuit_breaker_state`        | any circuit opens                  | Critical |
| `core_connection_pool_available`    | 0 available connections for 1 min  | Critical |
| `core_saga_stuck_total`             | any saga in partial state > 15 min | Critical |
| `core_saga_manual_review_total`     | any new manual review needed       | Warning  |

### Onboarding

| Metric                                 | Alert When                          | Severity |
| -------------------------------------- | ----------------------------------- | -------- |
| `onboarding_submission_errors_total`   | any submission fails                | Warning  |
| `onboarding_kyc_provider_errors_total` | KYC provider errors > 3 in 5 min    | Critical |
| `onboarding_pending_review_count`      | > 50 pending reviews (queue backup) | Warning  |
| `onboarding_avg_approval_time_hours`   | > 24 hours average                  | Warning  |

### Auth

| Metric                                   | Alert When                       | Severity |
| ---------------------------------------- | -------------------------------- | -------- |
| `auth_login_failures_total` rate         | > 20 failures/min (brute force?) | Critical |
| `auth_otp_delivery_failures_total`       | SMS delivery failing             | Critical |
| `auth_token_refresh_failures_total` rate | > 5% refresh failures            | Warning  |

### API Health

| Metric                               | Alert When               | Severity        |
| ------------------------------------ | ------------------------ | --------------- |
| `http_request_duration_seconds` p95  | > 2s for 5 min           | Warning         |
| `http_request_errors_total` 5xx rate | > 1% for 5 min           | Critical        |
| `http_request_errors_total` 5xx rate | > 5% for 2 min           | Critical (page) |
| `/health/detailed`                   | any dependency unhealthy | Critical        |

### Infrastructure

| Metric                      | Alert When              | Severity |
| --------------------------- | ----------------------- | -------- |
| `node_cpu_usage_percent`    | > 80% for 10 min        | Warning  |
| `node_memory_usage_percent` | > 85% for 5 min         | Critical |
| `pg_connections_active`     | > 80% of pool for 5 min | Warning  |
| `pg_disk_usage_percent`     | > 80%                   | Warning  |
| `pg_disk_usage_percent`     | > 90%                   | Critical |
| Container restart count     | any container restarted | Warning  |

---

## 3. Alert Routing

### Severity Levels

| Severity     | Meaning                         | Channels                       | Response Time |
| ------------ | ------------------------------- | ------------------------------ | ------------- |
| **Critical** | System broken, users affected   | Telegram + Email + Slack/Teams | Immediate     |
| **Warning**  | Degraded, not broken yet        | Email + Slack/Teams            | Within hours  |
| **Info**     | Notable event, no action needed | Slack/Teams only               | Review daily  |

### Contact Points (Grafana Config)

```yaml
# Grafana alerting contact points
contact_points:
  - name: telegram-critical
    type: telegram
    settings:
      botToken: ${TELEGRAM_BOT_TOKEN}
      chatId: ${TELEGRAM_CHAT_ID} # Group chat for ops team

  - name: email-ops
    type: email
    settings:
      addresses: ${ALERT_EMAIL_LIST} # Comma-separated

  - name: slack-ops
    type: slack
    settings:
      webhookUrl: ${SLACK_WEBHOOK_URL}
      channel: "#corebridge-alerts"

  # OR for Teams:
  - name: teams-ops
    type: teams
    settings:
      webhookUrl: ${TEAMS_WEBHOOK_URL}
```

### Notification Policies

```yaml
# Grafana notification policies
policies:
  - match:
      severity: critical
    contact_points: [telegram-critical, email-ops, slack-ops]
    repeat_interval: 15m # re-alert every 15 min if unresolved

  - match:
      severity: warning
    contact_points: [email-ops, slack-ops]
    repeat_interval: 1h

  - match:
      severity: info
    contact_points: [slack-ops]
    repeat_interval: 4h
```

### Who Gets Alerted

Managed via Grafana notification policies. Add/remove people by:

- Adding them to the Telegram group
- Adding their email to the contact point
- Adding them to the Slack/Teams channel

No code changes needed. Anyone you choose gets added through Grafana UI or config.

---

## 4. Alert Message Format

Every alert includes actionable context:

```
🔴 CRITICAL: Core Circuit Breaker OPEN

System: core-adapter / customer-service
Status: Circuit breaker opened after 25 consecutive failures
Since: 2026-03-15 14:32 UTC
Duration: 3 minutes

Last error: SOAP timeout after 15s on CreateCustomer
Error rate: 100% (5/5 requests failed)

Impact: All customer creation requests failing.
        Onboarding submissions will fail.

Dashboard: https://grafana.internal/d/core-adapter
Runbook: https://docs.internal/runbooks/circuit-breaker-open
```

```
🟡 WARNING: Onboarding Saga Stuck in Partial State

System: core-adapter / onboarding-saga
Application ID: app-uuid-123
Phone: +964XXXXXXXX (masked)
Stuck at: CUSTOMER_CREATED (account creation failed)
Since: 2026-03-15 13:15 UTC
Duration: 1 hour 17 minutes

Action required: Manual review needed.
                 Customer created in core (CIF: 12345) but account creation failed.
                 Check core status and create account manually or retry.

Dashboard: https://grafana.internal/d/sagas
```

---

## 5. Dashboards

### Core Adapter Dashboard

```
┌─────────────────────────────────────────────────────────┐
│ Core Adapter Health                                      │
│                                                          │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐ │
│ │ Status: OK   │ │ Error Rate:  │ │ Circuit Breakers:│ │
│ │              │ │ 0.2%         │ │ All CLOSED       │ │
│ └──────────────┘ └──────────────┘ └──────────────────┘ │
│                                                          │
│ Response Time (p50, p95, p99) ─── [line chart over time] │
│                                                          │
│ Errors by Operation ──────────── [bar chart]             │
│                                                          │
│ Connection Pool ──────────────── [gauge: 3/20 active]    │
│                                                          │
│ Saga Status ──────────────────── [table: active sagas]   │
│ │ app-123 │ CUSTOMER_CREATED │ stuck 1h │ ⚠️            │
│ │ app-456 │ COMPLETED        │ 3s       │ ✓             │
└─────────────────────────────────────────────────────────┘
```

### Onboarding Dashboard

```
┌─────────────────────────────────────────────────────────┐
│ Onboarding Pipeline                                      │
│                                                          │
│ ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│ │ In Progress│ │ Pending    │ │ Approved   │           │
│ │ 12         │ │ Review: 8  │ │ Today: 23  │           │
│ └────────────┘ └────────────┘ └────────────┘           │
│                                                          │
│ Submissions/hour ─────────── [line chart]                │
│ KYC Provider Response Time ── [line chart]               │
│ Failure Reasons ──────────── [pie chart]                 │
│ Avg Time to Approval ─────── [stat: 4.2 hours]          │
└─────────────────────────────────────────────────────────┘
```

### API Overview Dashboard

```
┌─────────────────────────────────────────────────────────┐
│ API Health                                               │
│                                                          │
│ Request Rate ─────────────── [line chart]                │
│ Error Rate (4xx, 5xx) ────── [line chart]                │
│ Response Time by Endpoint ── [heatmap]                   │
│ Active Sessions ──────────── [stat]                      │
│ Top Errors ───────────────── [table: code, count, last]  │
└─────────────────────────────────────────────────────────┘
```

---

## 6. Structured Logging (Loki)

Not just text logs. Structured JSON logs queryable in Grafana via Loki.

```typescript
// NestJS logger outputs structured JSON:
{
  "level": "error",
  "timestamp": "2026-03-15T14:32:00Z",
  "service": "core-adapter",
  "operation": "createCustomer",
  "correlationId": "req-uuid-123",
  "duration_ms": 15000,
  "error": "SOAP_TIMEOUT",
  "sagaId": "saga-uuid-456",
  "sagaStep": "CREATE_CUSTOMER",
  "message": "SOAP call timed out after 15s"
}
```

Loki queries in Grafana:

- `{service="core-adapter"} |= "error"` — all core adapter errors
- `{service="core-adapter"} | json | operation="createCustomer" | duration_ms > 5000` — slow customer creation
- `{service="core-adapter"} | json | sagaStep="CREATE_ACCOUNT" | error != ""` — failed account creation sagas

Logs feed into alerts too. Grafana can alert on log patterns (e.g., "more than 5 `SOAP_TIMEOUT` logs in 5 minutes").

---

## 7. Implementation Checklist

### NestJS Side

- [ ] Add `prom-client` for Prometheus metrics
- [ ] Create metrics module exposing `/metrics` endpoint
- [ ] Add structured JSON logger (Winston or Pino with JSON transport)
- [ ] Instrument core-adapter: request duration histogram, error counter, circuit breaker gauge
- [ ] Instrument sagas: step counter, stuck saga gauge
- [ ] Instrument auth: login failure counter, OTP delivery counter
- [ ] Add correlation ID to all logs (from `X-Request-Id` header)

### Infrastructure Side

- [ ] Configure Prometheus to scrape `/metrics` from API instances
- [ ] Deploy Promtail (log shipper) → Loki
- [ ] Add Loki as data source in Grafana
- [ ] Import/create dashboards (core adapter, onboarding, API overview)
- [ ] Configure alert rules in Grafana
- [ ] Configure contact points (Telegram bot, email SMTP, Slack/Teams webhook)
- [ ] Configure notification policies (severity → channel routing)
- [ ] Test alerts: trigger a fake circuit breaker open, verify Telegram/email/Slack receive it

---

## 8. Docker Compose Addition

```yaml
# Add to existing docker-compose.yml
services:
  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]
    volumes: [loki_data:/loki]
    restart: always

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./infra/promtail/config.yml:/etc/promtail/config.yml
    depends_on: [loki]
    restart: always

volumes:
  loki_data:
```

Grafana already exists in the stack. Just add Loki as a data source.
