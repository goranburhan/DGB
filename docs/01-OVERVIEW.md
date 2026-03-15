# Corebridge — Overview & System Architecture

**Version:** 0.2 | **Date:** March 15, 2026

---

## Overview

Corebridge is a digital banking platform built on top of ICS BANKS (ICSFS) legacy core. It provides retail customers with digital onboarding, account management, and self-service banking via a mobile app, and gives bank employees a backoffice for manual review and customer management.

**Business Model:** Build for one bank → sell the company + software to that bank (acquisition/exit).

**Target Market:** Iraqi retail bank (Islamic or conventional) — single bank, TBD.

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
│                    NESTJS BACKEND                            │
│               (Single Nx Monorepo)                          │
│                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │  Auth   │ │Onboard- │ │Accounts │ │  Cards  │          │
│  │ Module  │ │  ing    │ │ Module  │ │ Module  │          │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │
│       │           │           │           │                │
│  ┌────┴───────────┴───────────┴───────────┴────┐           │
│  │           DOMAIN SERVICES LAYER              │           │
│  └────────────────────┬────────────────────────┘           │
│                       │                                     │
│  ┌────────────────────┴────────────────────────┐           │
│  │              CORE ADAPTER                    │           │
│  │  ┌──────────────┐ ┌──────────┐ ┌──────────┐ │           │
│  │  │  interfaces/  │ │ mappers/ │ │soap-client││           │
│  │  └──────────────┘ └──────────┘ └──────────┘ │           │
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
| Mobile framework | React Native | Cross-platform, shared JS ecosystem |
| Backoffice | React (web) | Separate app for bank employees |
| Database | PostgreSQL | Audit logs, sessions, config. NOT source of truth for banking data |
| Source of truth | ICS BANKS Core | All customer, account, transaction, compliance data |
| Document storage | Core + local object storage | Dual storage for safety |
| Auth | JWT (access + refresh tokens) | Stateless |
| Core integration | Direct SOAP via WSDL-generated clients | CoreAdapter keeps business logic clean |
| Notifications | FCM (push) + in-app center + SMS (bank's gateway) | |
| Monitoring | Grafana + Prometheus | On-prem friendly |
| Deployment | On-prem, Docker | Bank controls their infra |
| Repo structure | Single Nx monorepo | No package distribution needed — one bank, one codebase |

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
| App configuration | PostgreSQL | Feature flags, settings |
| Sanctions screening results | Core | Compliance data stays in core |

---

See also: [02-STRUCTURE.md](./02-STRUCTURE.md) · [03-REPO.md](./03-REPO.md) · [04-API-DESIGN.md](./04-API-DESIGN.md) · [05-DATABASE.md](./05-DATABASE.md) · [06-DEPLOYMENT.md](./06-DEPLOYMENT.md) · [07-SECURITY.md](./07-SECURITY.md) · [08-AI-WORKFLOW.md](./08-AI-WORKFLOW.md) · [09-BUSINESS-OPERATIONS.md](./09-BUSINESS-OPERATIONS.md)
