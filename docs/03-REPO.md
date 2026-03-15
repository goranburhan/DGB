# Corebridge — Repository Structure

Single Nx monorepo. No package publishing, no per-bank repos.

---

## Monorepo Layout

```
corebridge/
├── apps/
│   ├── api/                        ← NestJS backend (runnable app)
│   │   ├── src/
│   │   │   ├── app.module.ts       ← Wires all lib modules
│   │   │   └── main.ts
│   │   └── package.json
│   ├── mobile/                     ← React Native app
│   │   ├── src/
│   │   │   ├── api/
│   │   │   │   └── generated/      ← Auto-generated typed API client
│   │   │   ├── validations/
│   │   │   │   └── generated/      ← Auto-generated from contracts
│   │   │   ├── screens/
│   │   │   │   ├── onboarding/
│   │   │   │   ├── auth/
│   │   │   │   ├── accounts/
│   │   │   │   ├── cards/
│   │   │   │   ├── notifications/
│   │   │   │   └── settings/
│   │   │   ├── components/
│   │   │   ├── navigation/
│   │   │   ├── i18n/
│   │   │   ├── theme/              ← RTL-first theming
│   │   │   ├── store/              ← Redux Toolkit
│   │   │   └── utils/
│   │   └── package.json
│   └── backoffice/                 ← React web app
│       ├── src/
│       │   ├── api/
│       │   │   └── generated/
│       │   ├── pages/
│       │   │   ├── dashboard/
│       │   │   ├── onboarding-review/
│       │   │   ├── customer-360/
│       │   │   ├── kyc-review/
│       │   │   ├── card-orders/
│       │   │   ├── audit-log/
│       │   │   └── settings/
│       │   ├── components/
│       │   ├── i18n/
│       │   ├── theme/
│       │   └── store/
│       └── package.json
├── libs/
│   ├── common/
│   ├── core-adapter/
│   │   ├── src/
│   │   │   ├── interfaces/
│   │   │   ├── mappers/
│   │   │   ├── soap-client/
│   │   │   │   ├── wsdl/
│   │   │   │   └── generated/
│   │   │   └── core-adapter.module.ts
│   │   └── index.ts
│   ├── auth/
│   ├── onboarding/
│   ├── kyc/
│   ├── sanctions/
│   ├── accounts/
│   ├── cards/
│   ├── notifications/
│   ├── backoffice/
│   ├── audit/
│   ├── storage/
│   └── i18n/
├── contracts/
│   ├── openapi.yaml                ← Full API spec
│   ├── schemas/                    ← JSON Schema models
│   ├── validations/                ← Rules with i18n
│   ├── errors/                     ← Error codes + i18n messages
│   └── events/                     ← Domain events
├── tools/
│   ├── codegen/                    ← Generates DTOs, API clients from contracts
│   └── wsdl-gen/                   ← Generates TS clients from ICSFS WSDLs
├── design/
│   ├── tokens/                     ← Figma-exported design tokens
│   ├── screens/                    ← Screen specs + screenshots
│   └── components/                 ← Component specs
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── infra/
│   ├── grafana/
│   └── prometheus/
├── AI_CONTEXT.md
├── CHANGELOG_AGENT.md
├── nx.json
├── tsconfig.base.json
└── package.json
```

---

## Why Nx

- First-class NestJS plugin (`@nx/nest`)
- Built-in code generators for modules, services, libraries
- Visual dependency graph (`nx graph`)
- Local caching (faster builds)
- Predictable structure — AI agents work better with this

---

## Why Single Repo (Not Multi-Repo)

| Before (multi-bank) | Now (single bank + exit) |
|---------------------|--------------------------|
| 5+ repos, package publishing | 1 repo, everything local |
| Per-bank customization repos | Direct implementation |
| GitHub Packages CI/CD | Simple docker build |
| Override/extend pattern via DI | Build exactly what's needed |

Single repo = faster development, simpler CI, easier handover at acquisition.
