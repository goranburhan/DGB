# Corebridge — AI Agent Workflow

---

## 1. Contract-First Development

```
┌──────────────────────────────────────────────────┐
│              SINGLE SOURCE OF TRUTH               │
│            (contracts/ directory)                  │
│                                                   │
│   ├── openapi.yaml          ← Full API spec      │
│   ├── schemas/              ← JSON Schema models  │
│   ├── validations/          ← Rules with i18n     │
│   ├── errors/               ← Error codes + i18n  │
│   └── events/               ← Domain events       │
└──────────────┬───────────────────────────────────┘
               │ Code Generation
    ┌──────────┼──────────┐
    ▼          ▼          ▼
 Backend    Mobile    Backoffice
  types     types      types
  + DTOs    + API      + API
  + valid.  client     client
```

Backend agent → updates contracts → implements endpoint → Swagger stays in sync.
Frontend agent → reads contracts → uses generated API client → applies same validations.

---

## 2. Figma → AI Agent → Mobile App

### Design Tokens + Screen Specs

```
design/
├── tokens/
│   ├── colors.json            ← Exported from Figma
│   ├── typography.json
│   ├── spacing.json
│   └── shadows.json
├── screens/
│   ├── onboarding/
│   │   ├── phone-verification.spec.md
│   │   ├── phone-verification.png
│   │   └── ...
│   ├── auth/
│   │   ├── login.spec.md
│   │   ├── login.png
│   │   └── ...
│   └── accounts/
│       └── ...
└── components/
    ├── button.spec.md
    ├── input.spec.md
    └── ...
```

### How It Works

1. Export design tokens from Figma (Figma Tokens plugin → JSON)
2. Create screen specs (`.spec.md` + screenshot per screen)
3. AI agent reads spec + screenshot + tokens → builds the screen

Screen spec example:

```markdown
# Login Screen — login.spec.md
## Layout (top to bottom)
1. Logo (centered, 80x80, top margin 60px)
2. Heading: i18n key `auth.login.title` — heading1, centered
3. Phone input: with +964 prefix, numeric keyboard
4. Password input: secure text, show/hide toggle
5. "Forgot password?" link: right-aligned, primary color
6. Login button: full width, primary, 48px height

## API Calls
- POST /api/v1/auth/login (phone, password)

## States
- Default, Loading, Error, OTP sent

## RTL Behavior
- All text right-aligned in Arabic/Kurdish
```

---

## 3. AI_CONTEXT.md

Root-level file that gives any AI agent immediate context:

```markdown
# AI_CONTEXT.md
## What This Repo Is
Single Nx monorepo for Corebridge digital banking platform.
Built for one Iraqi bank on ICS BANKS (ICSFS) core.

## Structure
- apps/api — NestJS backend
- apps/mobile — React Native
- apps/backoffice — React web
- libs/* — shared modules
- contracts/ — OpenAPI, schemas, validations, error codes

## Key Rules
- All core calls through CoreAdapter in libs/core-adapter/
- All endpoints need Swagger decorators
- All mutations must write to audit log
- Never store banking data in PostgreSQL
- Validation rules from contracts/, not hardcoded
- All text must use i18n keys
- RTL layout is default for mobile
- Screen specs in design/screens/
- Design tokens in design/tokens/
```

---

## 4. Change Log Protocol

When any agent makes a change, append to `CHANGELOG_AGENT.md`:

```markdown
## 2026-03-15 — Backend Agent
### Added
- POST /api/v1/onboarding/submit-documents
### Changed
- GET /api/v1/accounts/:id/transactions — now cursor paginated
### Impact
- Frontend must update transaction list to use cursor pagination
```

---

## 5. Validation Sync

Defined once in `contracts/validations/`, generated into backend and frontend.

```json
{
  "nationalId": {
    "type": "string",
    "minLength": 12, "maxLength": 12,
    "pattern": "^[0-9]+$",
    "label": { "ar": "رقم الهوية الوطنية", "ku": "ژمارەی ناسنامە", "en": "National ID Number" }
  },
  "phoneNumber": {
    "type": "string",
    "pattern": "^\\+964[0-9]{10}$",
    "label": { "ar": "رقم الهاتف", "ku": "ژمارەی مۆبایل", "en": "Phone Number" }
  }
}
```

Backend → NestJS validation pipes. Frontend → form validation + localized errors. One source, zero mismatch.

---

## 6. Error Code Registry

Defined in `contracts/errors/error-codes.json`:

```json
{
  "AUTH_001": {
    "message": { "ar": "انتهت صلاحية الجلسة", "ku": "کاتی دانیشتن تەواو بوو", "en": "Session expired" },
    "httpStatus": 401,
    "action": "REDIRECT_LOGIN"
  }
}
```

Backend returns code → frontend looks up localized message + action.

---

## 7. Agent Handoff Checklist

Before finishing a session:

- [ ] New/changed endpoints reflected in `contracts/openapi.yaml`
- [ ] New validation rules in `contracts/validations/`
- [ ] New error codes in `contracts/errors/error-codes.json`
- [ ] `CHANGELOG_AGENT.md` updated with changes + impact
- [ ] Code generation (`npm run generate`) runs clean
- [ ] Tests pass for affected modules
- [ ] No banking data stored in PostgreSQL
- [ ] All text uses i18n keys
- [ ] Screen specs updated if UI changed
