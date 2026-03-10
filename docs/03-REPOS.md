# Corebridge — Repository Structure

---

## Repos Overview

```
GitHub: corebridge-io (organization)
│
├── corebridge-contracts          ← OpenAPI, schemas, validations, error codes (separate repo)
├── corebridge-backend            ← NestJS Nx monorepo containing all @corebridge/* backend packages
├── corebridge-mobile             ← React Native mobile app
├── corebridge-backoffice         ← React web backoffice app
├── corebridge-infra              ← Docker, CI/CD, Grafana dashboards, deployment scripts
│
└── bank-deployments/
    ├── bank-a-backend            ← Bank A's backend customization (imports @corebridge/*)
    ├── bank-a-mobile             ← Bank A's mobile app (themed + customized)
    ├── bank-a-backoffice         ← Bank A's backoffice
    └── ...
```

---

## Backend Monorepo (NestJS + Nx)

```
corebridge-backend/
├── packages/
│   ├── common/                    ← @corebridge/common
│   │   ├── src/
│   │   │   ├── decorators/
│   │   │   ├── guards/
│   │   │   ├── interceptors/
│   │   │   ├── types/
│   │   │   └── utils/
│   │   └── package.json
│   ├── core-adapter/              ← @corebridge/core-adapter
│   │   ├── src/
│   │   │   ├── interfaces/
│   │   │   ├── mappers/
│   │   │   ├── soap-client/
│   │   │   │   ├── wsdl/
│   │   │   │   └── generated/
│   │   │   └── core-adapter.module.ts
│   │   └── package.json
│   ├── auth/                      ← @corebridge/auth
│   │   ├── src/
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── dto/
│   │   │   ├── entities/
│   │   │   └── auth.module.ts
│   │   └── package.json
│   ├── onboarding/
│   ├── kyc/
│   ├── sanctions/
│   ├── accounts/
│   ├── cards/
│   ├── notifications/
│   ├── backoffice/
│   ├── audit/
│   ├── storage/
│   ├── i18n/
│   └── config/
├── apps/
│   └── api/                       ← The runnable NestJS app
│       ├── src/
│       │   ├── app.module.ts      ← Wires all @corebridge/* modules
│       │   └── main.ts
│       └── package.json
├── tools/
│   ├── codegen/                   ← Generates DTOs, API clients from contracts
│   └── wsdl-gen/                  ← Generates TS clients from ICSFS WSDLs
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── AI_CONTEXT.md
├── CHANGELOG_AGENT.md
├── nx.json
├── tsconfig.base.json
└── package.json
```

---

## Mobile App

```
corebridge-mobile/
├── src/
│   ├── api/
│   │   ├── generated/             ← Auto-generated typed API client
│   │   └── client.ts              ← Axios/fetch wrapper with auth interceptor
│   ├── validations/
│   │   └── generated/             ← Auto-generated from contracts
│   ├── screens/
│   │   ├── onboarding/
│   │   ├── auth/
│   │   ├── accounts/
│   │   ├── cards/
│   │   ├── notifications/
│   │   └── settings/
│   ├── components/
│   ├── navigation/
│   ├── i18n/
│   ├── theme/                     ← RTL-first theming
│   ├── store/                     ← Redux Toolkit
│   └── utils/
├── AI_CONTEXT.md
├── CHANGELOG_AGENT.md
└── package.json
```

---

## Backoffice

```
corebridge-backoffice/
├── src/
│   ├── api/
│   │   └── generated/
│   ├── pages/
│   │   ├── dashboard/
│   │   ├── onboarding-review/
│   │   ├── customer-360/
│   │   ├── kyc-review/
│   │   ├── card-orders/
│   │   ├── audit-log/
│   │   └── settings/
│   ├── components/
│   ├── i18n/
│   ├── theme/
│   └── store/
├── AI_CONTEXT.md
├── CHANGELOG_AGENT.md
└── package.json
```

---

## Bank Customization Repo

When Bank A engages Corebridge, their backend repo:

```
bank-a-backend/
├── src/
│   ├── overrides/
│   │   ├── core-adapter/
│   │   │   └── mappers/           ← Bank A's ICSFS field mappings
│   │   ├── onboarding/
│   │   │   └── bank-a-steps.ts    ← Custom onboarding steps
│   │   └── notifications/
│   │       └── sms-gateway.ts     ← Bank A's SMS provider
│   ├── extensions/
│   │   └── custom-reports/        ← New features bank wants
│   └── app.module.ts              ← Imports @corebridge/* + overrides
├── config/
│   ├── bank.config.ts
│   └── core-mappings.ts
├── AI_CONTEXT.md
├── docker/
└── package.json                   ← Depends on @corebridge/* packages
```

**Override** = replace existing behavior (e.g., different ICSFS field mappings).
**Extend** = add new behavior (e.g., VIP customer report that doesn't exist in base).

NestJS dependency injection makes this clean — Bank A's `app.module.ts` imports base modules and swaps/adds what they need.

---

## Why Nx

- First-class NestJS plugin (`@nx/nest`)
- Built-in code generators for modules, services, libraries
- Visual dependency graph (`nx graph`)
- Local + remote caching (faster CI)
- Predictable structure — AI agents work better with this
