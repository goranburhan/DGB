# Corebridge — AI Agent Workflow & Communication Protocol

---

## 1. The Problem

When multiple AI agents work on a codebase (backend, frontend, customization), they need:

1. **Shared contract** — frontend agent must know exactly what APIs exist
2. **Change propagation** — backend changes must reach frontend agent immediately
3. **Validation sync** — same rules on both ends
4. **Design awareness** — frontend agent must know what to build visually
5. **Domain knowledge** — agents need banking context without guessing

---

## 2. Contract-First Development

```
┌──────────────────────────────────────────────────┐
│              SINGLE SOURCE OF TRUTH               │
│         (corebridge-contracts repo)               │
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

**Backend agent** → updates contracts → implements endpoint → Swagger stays in sync.
**Frontend agent** → reads contracts → uses generated API client → applies same validations.
**Customization agent** → reads contracts + bank's overrides → extends only what's needed.

---

## 3. Figma → AI Agent → Mobile App

### The Problem

AI agents can't see Figma designs by default. They need structured design information to build screens that match the design accurately.

### Solution: Design Tokens + Screen Specs

```
corebridge-mobile/
├── design/
│   ├── tokens/
│   │   ├── colors.json            ← Exported from Figma
│   │   ├── typography.json        ← Font sizes, weights, families
│   │   ├── spacing.json           ← Padding, margins, gaps
│   │   └── shadows.json           ← Elevation, shadows
│   ├── screens/
│   │   ├── onboarding/
│   │   │   ├── phone-verification.spec.md
│   │   │   ├── phone-verification.png     ← Screenshot from Figma
│   │   │   ├── personal-info.spec.md
│   │   │   ├── personal-info.png
│   │   │   └── ...
│   │   ├── auth/
│   │   │   ├── login.spec.md
│   │   │   ├── login.png
│   │   │   └── ...
│   │   └── accounts/
│   │       ├── balance.spec.md
│   │       ├── balance.png
│   │       └── ...
│   └── components/
│       ├── button.spec.md
│       ├── input.spec.md
│       ├── card.spec.md
│       └── ...
```

### How It Works

**Step 1: Export design tokens from Figma**

Use Figma Tokens plugin (or Figma Variables) to export tokens as JSON. These become the theme source.

```json
// design/tokens/colors.json
{
  "primary": "#1B4D3E",
  "primaryLight": "#2A7A5F",
  "background": "#FFFFFF",
  "surface": "#F5F5F5",
  "text": "#1A1A1A",
  "textSecondary": "#6B7280",
  "error": "#DC2626",
  "success": "#16A34A",
  "border": "#E5E7EB"
}
```

```json
// design/tokens/typography.json
{
  "heading1": { "fontSize": 28, "fontWeight": "700", "lineHeight": 36 },
  "heading2": { "fontSize": 22, "fontWeight": "600", "lineHeight": 28 },
  "body": { "fontSize": 16, "fontWeight": "400", "lineHeight": 24 },
  "caption": { "fontSize": 12, "fontWeight": "400", "lineHeight": 16 }
}
```

**Step 2: Create screen specs**

For each screen, create a `.spec.md` file + a screenshot. The spec tells the AI agent exactly what to build.

```markdown
# Login Screen — login.spec.md

## Screenshot
See: login.png

## Layout (top to bottom)
1. Logo (centered, 80x80, top margin 60px)
2. Heading: i18n key `auth.login.title` — heading1, centered
3. Subheading: i18n key `auth.login.subtitle` — body, textSecondary, centered
4. Phone input: i18n key `auth.login.phone` — with +964 prefix, numeric keyboard
5. Password input: i18n key `auth.login.password` — secure text, show/hide toggle
6. "Forgot password?" link: i18n key `auth.login.forgot` — right-aligned, primary color
7. Login button: i18n key `auth.login.submit` — full width, primary, 48px height
8. Spacer
9. "Don't have an account? Register" — body, centered, "Register" is primary color link

## API Calls
- POST /api/v1/auth/login (phone, password)
- On success: POST /api/v1/auth/verify-otp (OTP screen)

## Validations (from contracts)
- phone: onboarding.rules.json → phoneNumber
- password: auth.rules.json → password

## States
- Default: all fields empty
- Loading: button shows spinner, inputs disabled
- Error: error banner below subheading, shake animation
- OTP sent: navigate to OTP verification screen

## RTL Behavior
- All text right-aligned in Arabic/Kurdish
- "Forgot password?" link left-aligned in RTL
- Phone input prefix on right side in RTL
```

**Step 3: AI agent reads spec + screenshot + tokens → builds the screen**

The agent has everything: visual reference, exact layout, API calls, validations, i18n keys, states, and RTL behavior. No guessing.

### Figma Export Workflow

```
Designer updates Figma
    │
    ├── 1. Export design tokens (Figma Tokens plugin → JSON)
    ├── 2. Export screen screenshots (Figma → PNG, one per screen)
    ├── 3. Update screen spec .md files (layout, states, RTL notes)
    └── 4. Commit to corebridge-mobile/design/

AI agent picks up changes:
    ├── Reads updated tokens → regenerates theme
    ├── Reads screen spec → knows exactly what to build
    ├── Views screenshot → visual reference
    └── Builds/updates the React Native screen
```

### Component Specs

Shared components also get specs:

```markdown
# Button Component — button.spec.md

## Variants
- primary: background=primary, text=white, height=48
- secondary: background=transparent, border=primary, text=primary, height=48
- danger: background=error, text=white, height=48
- text: background=transparent, text=primary, no border

## Props
- label: string (i18n key)
- variant: 'primary' | 'secondary' | 'danger' | 'text'
- onPress: function
- loading: boolean (shows spinner, disables press)
- disabled: boolean (opacity 0.5, no press)
- fullWidth: boolean (default true)

## RTL
- No special handling needed (text auto-flips)

## Accessibility
- accessibilityRole="button"
- accessibilityLabel from label prop
```

---

## 4. Agent Context Files

Each repo includes `AI_CONTEXT.md` at root. See examples:

### Backend

```markdown
# AI_CONTEXT.md
## What This Repo Does
NestJS backend for Corebridge digital banking.
## Key Rules
- Modules in /src/modules/{feature}/ (controller, service, dto, entity)
- All core calls through CoreAdapter in /src/core-adapter/
- All endpoints need Swagger decorators
- All mutations must write to audit log
- Never store banking data in PostgreSQL
- Validation rules from contracts, not hardcoded
```

### Mobile

```markdown
# AI_CONTEXT.md
## What This Repo Does
React Native mobile app for Corebridge.
## Key Rules
- API calls ONLY through generated client in /src/api/generated/
- Validations ONLY from generated rules in /src/validations/generated/
- All text must use i18n keys
- RTL layout is default
- Screen specs in /design/screens/{feature}/
- Design tokens in /design/tokens/
- Never store sensitive data on device
```

### Bank Customization

```markdown
# AI_CONTEXT.md
## What This Repo Does
Customization for [Bank Name] on top of @corebridge/* packages.
## Rules
- Base packages: @corebridge/* (DO NOT modify directly)
- Overrides in /src/overrides/{module}/
- Config in /config/bank.config.ts
- Can customize: onboarding steps, KYC docs, theme, SMS, sanctions, backoffice screens
- Cannot change: auth mechanism, CoreAdapter interface, audit logging
```

---

## 5. Change Log Protocol

When any agent makes a change, it appends to `CHANGELOG_AGENT.md`:

```markdown
## 2026-03-15 — Backend Agent
### Added
- POST /api/v1/onboarding/submit-documents
  - Accepts: nationalId (front + back), selfie, proof of address
  - Returns: applicationId, status
### Changed
- GET /api/v1/accounts/:id/transactions — now cursor paginated
### Impact
- Frontend must update transaction list to use cursor pagination
- Backoffice unaffected
```

---

## 6. Validation Sync

Defined once in `corebridge-contracts/validations/`, generated into backend and frontend.

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

## 7. Error Code Registry

Defined in `corebridge-contracts/errors/error-codes.json`:

```json
{
  "AUTH_001": {
    "message": { "ar": "انتهت صلاحية الجلسة", "ku": "کاتی دانیشتن تەواو بوو", "en": "Session expired" },
    "httpStatus": 401,
    "action": "REDIRECT_LOGIN"
  }
}
```

Backend returns code → frontend looks up localized message + action. AI agents know every error and when it's thrown.

---

## 8. Agent Handoff Checklist

Before finishing a session:

- [ ] New/changed endpoints reflected in `openapi.yaml`
- [ ] New validation rules in `validations/`
- [ ] New error codes in `error-codes.json`
- [ ] `CHANGELOG_AGENT.md` updated with changes + impact
- [ ] Code generation (`npm run generate`) runs clean
- [ ] Integration tests pass for affected modules
- [ ] No banking data stored in PostgreSQL
- [ ] All text uses i18n keys
- [ ] Screen specs updated if UI changed
