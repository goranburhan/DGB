# Corebridge — Overview & System Architecture

**Version:** 0.1 (Draft) | **Date:** March 10, 2026

---

## Overview

Corebridge is a digital banking platform that sits on top of legacy core banking systems (ICS BANKS by ICSFS). It provides retail customers with digital onboarding, account management, and self-service banking via a mobile app, while giving bank employees a backoffice for manual review and customer management.

**Business Model:** License fee + setup cost + customization cost (3 months per bank)

**Target Market:** Iraqi retail banks (Islamic and conventional)

**Languages:** Arabic (default, RTL), Sorani Kurdish (RTL), English (LTR)

---

## High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      MOBILE APP                             │
│                  (React Native)                             │
│         Onboarding · Login · Accounts · Cards               │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTPS (REST/JSON)
                       │ + FCM Push Notifications
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    API GATEWAY                              │
│               (NestJS Application)                          │
│                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │  Auth   │ │Onboard- │ │Accounts │ │  Cards  │          │
│  │ Module  │ │  ing    │ │ Module  │ │ Module  │          │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │
│       │           │           │           │                │
│  ┌────┴───────────┴───────────┴───────────┴────┐           │
│  │           DOMAIN SERVICES LAYER              │           │
│  │     (Clean business logic, no SOAP here)     │           │
│  └────────────────────┬────────────────────────┘           │
│                       │                                     │
│  ┌────────────────────┴────────────────────────┐           │
│  │              CORE ADAPTER                    │           │
│  │    (Single module, internally separated)     │           │
│  │                                              │           │
│  │  ┌────────────────────────────────────────┐  │           │
│  │  │  /interfaces                           │  │           │
│  │  │  Clean TypeScript interfaces that      │  │           │
│  │  │  domain services depend on             │  │           │
│  │  └────────────────────────────────────────┘  │           │
│  │                                              │           │
│  │  ┌────────────────────────────────────────┐  │           │
│  │  │  /mappers (ACL logic)                  │  │           │
│  │  │  - Field mapping & transformation      │  │           │
│  │  │  - Corebridge models ←→ ICSFS models   │  │           │
│  │  │  - Error code normalization            │  │           │
│  │  │  - Response validation                 │  │           │
│  │  └────────────────────────────────────────┘  │           │
│  │                                              │           │
│  │  ┌────────────────────────────────────────┐  │           │
│  │  │  /soap-client                          │  │           │
│  │  │  - WSDL-generated TypeScript clients   │  │           │
│  │  │  - Connection pooling                  │  │           │
│  │  │  - Retry logic & circuit breaker       │  │           │
│  │  │  - Request/response logging            │  │           │
│  │  └────────────────────────────────────────┘  │           │
│  └────────────────────┬────────────────────────┘           │
│                       │                                     │
│  ┌─────────┐ ┌───────┴──┐ ┌──────────┐ ┌──────────┐      │
│  │Postgres │ │ Object   │ │   KYC    │ │Sanctions │      │
│  │  (local)│ │ Storage  │ │ Provider │ │ Provider │      │
│  └─────────┘ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────┬───────────────────────────────────┘
                          │ SOAP/HTTPS
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  ICS BANKS CORE                              │
│               (Bank's on-prem server)                       │
│                                                             │
│  Customer Master · Accounts · Transactions · Cards          │
│  KYC Storage · Sanctions · All compliance data              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   BACKOFFICE APP                             │
│                    (React Web)                               │
│     Manual Review · 360 View · KYC Approval · Audit Log     │
│     Connects to same NestJS backend via REST API            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    MONITORING (Grafana)                      │
│     API metrics · Error rates · Core response times         │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Architecture Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Backend framework | NestJS | Module system, DI, built-in Swagger, guards/interceptors |
| Mobile framework | React Native | Shared JS context with backend, cross-platform |
| Backoffice | React (web) | Separate app for bank employees |
| Database | PostgreSQL | Audit logs, sessions, config. NOT source of truth for banking data |
| Source of truth | ICS BANKS Core | All customer, account, transaction, compliance data |
| Document storage | Core + local object storage | Dual storage for safety |
| Auth | JWT (access + refresh tokens) | Stateless, pluggable |
| Core integration | Direct SOAP via WSDL-generated clients | CoreAdapter keeps business logic clean |
| Notifications | FCM (push) + in-app center + SMS (bank's gateway) | |
| Monitoring | Grafana + Prometheus | On-prem friendly |
| Deployment | On-prem per bank, Docker | Bank controls their infra |
| Distribution | `@corebridge/*` npm packages | Per-bank repos extend them |

---

## Data Flow: What Lives Where

| Data | Location | Why |
|------|----------|-----|
| Customer records | Core | CBI compliance, single source of truth |
| Account balances & transactions | Core | Real-time from core APIs |
| KYC documents | Core + local object storage | Dual storage for safety |
| Audit trail | PostgreSQL | App-level audit, queryable for backoffice |
| User sessions & tokens | PostgreSQL | Session tracking |
| Notification history | PostgreSQL | In-app notification center |
| App configuration | PostgreSQL | Per-bank feature flags, settings |
| Sanctions screening results | Core | Compliance data stays in core |

---

See also: [02-PACKAGES.md](./02-PACKAGES.md) · [03-REPOS.md](./03-REPOS.md) · [04-API-DESIGN.md](./04-API-DESIGN.md) · [05-DATABASE.md](./05-DATABASE.md) · [06-DEPLOYMENT.md](./06-DEPLOYMENT.md) · [07-SECURITY.md](./07-SECURITY.md) · [08-AI-WORKFLOW.md](./08-AI-WORKFLOW.md)
