# Mobile App — AI Agent Workflow

How AI agents build and maintain the mobile app.

---

## 1. What the Agent Has Access To

```
Before writing any mobile code, the agent reads:

1. contracts/openapi.yaml           ← What endpoints exist, request/response shapes
2. contracts/schemas/               ← Data models
3. contracts/validations/           ← Form validation rules + i18n error messages
4. contracts/errors/                ← Error codes, actions, localized messages
5. design/tokens/                   ← Colors, typography, spacing, shadows (from Figma)
6. design/screens/{feature}/*.spec.md  ← Screen layout, states, RTL, API calls
7. design/screens/{feature}/*.png      ← Figma screenshot (visual reference)
8. design/components/*.spec.md     ← Shared component specs
9. src/api/generated/              ← Auto-generated typed API client
10. src/validations/generated/     ← Auto-generated validation schemas
11. CHANGELOG_AGENT.md             ← What backend changed recently
12. AI_CONTEXT.md                  ← Repo rules and conventions
```

The agent never guesses API shapes, validation rules, or design — everything is derived from these sources.

---

## 2. Building a New Screen

### Step-by-step

```
1. Read the screen spec
   └── design/screens/accounts/balance.spec.md
   └── Knows: layout, components, API calls, validations, states, RTL behavior

2. Read the screenshot
   └── design/screens/accounts/balance.png
   └── Visual reference for spacing, alignment, hierarchy

3. Check API contract
   └── contracts/openapi.yaml → GET /api/v1/accounts/:id
   └── Knows exact request/response types

4. Check if RTK Query endpoint exists
   └── src/api/endpoints/accounts.api.ts
   └── If not → create it

5. Check if validation schema exists
   └── src/validations/generated/ (for form screens)
   └── If not → run codegen first

6. Build the screen
   └── app/(main)/(accounts)/[id]/index.tsx
   └── Compose from: Screen, existing components, hooks
   └── Max ~150 lines, thin screen pattern

7. Create feature-specific components if needed
   └── app/(main)/(accounts)/components/AccountCard.tsx
   └── Keep in feature folder, not in shared components (unless reused)

8. Update CHANGELOG_AGENT.md
```

### What the agent must NOT do

- Hardcode validation rules (use generated schemas)
- Hardcode text strings (use i18n keys)
- Make API calls outside RTK Query
- Create components that duplicate existing shared ones
- Store sensitive data in Redux
- Ignore RTL behavior

---

## 3. Modifying an Existing Screen

```
1. Read CHANGELOG_AGENT.md → what backend changed
   └── e.g., "GET /accounts/:id/transactions now returns `runningBalance` field"

2. Read updated contract
   └── contracts/openapi.yaml → check new response shape

3. Re-run codegen if types changed
   └── npm run generate

4. Update screen/component to use new data
   └── e.g., add runningBalance column to transaction list

5. Update screen spec if layout changed
   └── design/screens/accounts/transactions.spec.md

6. Update CHANGELOG_AGENT.md
```

---

## 4. Figma Design Change → Mobile Update

```
Designer updates Figma
    │
    ├── 1. Export new design tokens (Figma Tokens plugin → JSON)
    │      └── design/tokens/colors.json, typography.json, etc.
    │
    ├── 2. Export updated screenshots
    │      └── design/screens/{feature}/{screen}.png
    │
    ├── 3. Update screen spec if layout changed
    │      └── design/screens/{feature}/{screen}.spec.md
    │
    └── 4. Commit to repo

Agent picks up changes:
    ├── Reads updated tokens → regenerates theme (src/theme/)
    ├── Reads screen spec → knows what changed
    ├── Views screenshot → visual reference
    └── Updates affected React Native screens/components
```

---

## 5. Adding a New Feature (End-to-End)

Example: adding a "Transfer Money" feature.

```
Agent receives:
├── Screen specs: design/screens/transfers/*.spec.md + *.png
├── API contract: contracts/openapi.yaml (new transfer endpoints)
├── Validation rules: contracts/validations/transfer.rules.json
├── Error codes: contracts/errors/error-codes.json (new TRANSFER_* codes)

Agent creates:
├── app/(main)/(transfers)/
│   ├── _layout.tsx
│   ├── index.tsx                    ← Transfer list / initiate
│   ├── new.tsx                      ← New transfer form
│   ├── confirm.tsx                  ← Review + selfie if required
│   └── components/
│       ├── TransferForm.tsx
│       └── TransferCard.tsx
├── src/api/endpoints/transfers.api.ts  ← RTK Query endpoints
├── src/store/slices/transfers.slice.ts ← If local state needed
├── src/i18n/ → adds transfer.* keys to ar.json, ku.json, en.json

Agent updates:
├── app/(main)/_layout.tsx           ← Add transfers tab (if in tab bar)
├── src/utils/navigation.ts          ← Add deep link routes
├── CHANGELOG_AGENT.md
```

---

## 6. Form Screen Pattern

Most onboarding and settings screens are forms. Agent follows this pattern:

```
1. Read validation schema from contracts/validations/
2. Use useForm hook with the schema
3. Build form using FormField components
4. Pre-fill from Redux slice if available (e.g., OCR data)
5. On submit → dispatch to slice + navigate to next step (or call API)
6. Error display via FormField (inline) + ErrorBanner (API errors)
```

```typescript
// Agent generates this pattern for every form screen:
export default function WorkDataScreen() {
  const { formData } = useAppSelector(s => s.onboarding);
  const dispatch = useAppDispatch();
  const router = useRouter();

  const { values, errors, handleChange, handleSubmit, isValid } = useForm({
    schema: workDataSchema,
    initialValues: formData.workData,
    onSubmit: (data) => {
      dispatch(setWorkData(data));
      router.push('/(onboarding)/data-entry/income-data');
    },
  });

  return (
    <Screen header={<Header title="onboarding.workData.title" />}>
      <StepIndicator current={4} total={8} />
      <FormField label="onboarding.workData.status" ... />
      <FormField label="onboarding.workData.employer" ... />
      <Button label="common.next" onPress={handleSubmit} disabled={!isValid} />
    </Screen>
  );
}
```

---

## 7. RTL Checklist (Per Screen)

Agent verifies for every screen:

- [ ] Text alignment flips in Arabic/Kurdish
- [ ] Flex direction flips (row → row-reverse)
- [ ] Directional icons flip (chevrons, arrows, back button)
- [ ] Phone input prefix on correct side
- [ ] Padding/margin uses `Start`/`End` not `Left`/`Right`
- [ ] Scrollable content respects direction
- [ ] Navigation gestures match direction

---

## 8. Agent Context File

```markdown
# AI_CONTEXT.md (mobile root)

## What This App Is
React Native (Expo) mobile app for Corebridge digital banking.
Single bank, Iraqi market, Arabic/Kurdish/English, RTL-first.

## Tech
- Expo managed workflow, Expo Router (file-based)
- Redux Toolkit + RTK Query
- expo-secure-store for tokens/PIN
- expo-local-authentication for biometrics

## Rules
- API calls ONLY through RTK Query endpoints in src/api/endpoints/
- Validations ONLY from generated schemas in src/validations/generated/
- All text MUST use i18n keys — no hardcoded strings
- RTL is default layout direction
- Screen files max ~150 lines — compose from hooks + components
- Tokens/PIN NEVER in Redux — only in expo-secure-store
- Screen specs in design/screens/{feature}/
- Design tokens in design/tokens/
- Never store banking data on device
- Always check CHANGELOG_AGENT.md for backend changes before modifying API-dependent code

## Codegen
- npm run generate:api    → regenerates src/api/generated/
- npm run generate:valid  → regenerates src/validations/generated/
- Run these after any contracts/ change
```

---

## 9. Agent Handoff Checklist (Mobile)

Before finishing a session:

- [ ] New screens follow the thin screen pattern (max ~150 lines)
- [ ] All API calls use RTK Query (no raw fetch)
- [ ] All form validations use generated schemas
- [ ] All text uses i18n keys (ar, ku, en)
- [ ] RTL verified for new/changed screens
- [ ] No sensitive data in Redux or AsyncStorage
- [ ] Screen specs updated if UI changed
- [ ] `CHANGELOG_AGENT.md` updated with changes + impact
- [ ] Codegen runs clean (`npm run generate`)
- [ ] TypeScript compiles with no errors
