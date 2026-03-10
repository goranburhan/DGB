# Corebridge — Business & Operations Guide

How the technical architecture translates to real business operations.

---

## 1. How We Sell to a New Bank

### What the Bank Gets

1. **License** — the right to use Corebridge software (`@corebridge/*` packages)
2. **Setup** — we deploy the software on their infrastructure, connect it to their ICS BANKS core
3. **Customization** — 3 months of tailoring: branding, onboarding flow, KYC provider, SMS gateway, ICSFS field mappings, any bank-specific features

### What We Deliver

- Their own codebase (not shared with other banks)
- Deployed and running on their on-prem servers
- CI/CD pipeline configured
- Monitoring (Grafana) dashboards set up
- Training for their IT team on basic operations (restart, monitor, backup)
- Documentation specific to their deployment

---

## 2. How a Bank Engagement Works (3 Months)

### Month 1: Integration & Core Setup

```
Week 1-2:
├── Create bank repos (bank-x-backend, bank-x-mobile, bank-x-backoffice)
├── Set up bank's dev/staging environment
├── Get ICSFS WSDL files from bank's core team
├── Map their ICSFS fields to Corebridge models (CoreAdapter mappers)
└── Basic connectivity test: can we talk to their core?

Week 3-4:
├── Integrate their SMS gateway (1-2 APIs)
├── Configure their KYC provider (Sumsub/Regula/GBG)
├── Configure their sanctions provider (or use their existing)
├── Test: onboarding flow end-to-end against their test core
└── Test: login, balance, transactions against their test core
```

### Month 2: Customization & Branding

```
Week 5-6:
├── Customize onboarding steps per bank policy
├── Add bank-specific document requirements
├── Apply bank branding (logo, colors, typography) to mobile app
├── Apply bank branding to backoffice
├── Customize notification templates (SMS text, push messages)
└── Bank-specific feature requests (if any)

Week 7-8:
├── Bank's compliance team reviews the system
├── Backoffice training for bank employees
├── UAT (User Acceptance Testing) with bank's team
├── Fix issues from UAT
└── Performance testing against their core
```

### Month 3: Production & Handover

```
Week 9-10:
├── Set up production environment on bank's servers
├── Production deployment
├── Configure CI/CD for bank's production
├── Set up monitoring and alerting
├── Set up backup scripts and cron jobs
└── Smoke testing on production

Week 11-12:
├── Soft launch (limited users or internal employees)
├── Monitor and fix production issues
├── Full launch
├── Handover documentation
└── Support transition (define ongoing support terms)
```

---

## 3. How Banks Customize the Software

### What They Can Change (Without Touching Base Code)

| Area | How | Example |
|------|-----|---------|
| ICSFS field mappings | Override mappers in `/overrides/core-adapter/` | Their core uses `CUST_NUM` instead of `CIF_NO` |
| Onboarding steps | Config-driven flow in `bank.config.ts` | Add "employer letter" step, remove "proof of address" |
| KYC provider | Config switch | Swap Sumsub for Regula |
| Sanctions provider | Config switch | Use bank's existing provider |
| SMS gateway | Override in `/overrides/notifications/` | Integrate bank's SMS API |
| Branding | Theme tokens in mobile + backoffice | Logo, colors, fonts |
| Notification text | Override templates | Different wording for SMS/push |
| Feature flags | Toggle in database | Enable/disable card ordering |
| New features | Add in `/extensions/` | VIP customer dashboard, custom reports |

### What They Cannot Change

| Area | Why |
|------|-----|
| Auth mechanism (JWT) | Security architecture, must be consistent |
| Audit logging | Compliance requirement, must remain untouched |
| CoreAdapter interface | Other modules depend on it — changing the interface breaks everything |
| API response format | Mobile app depends on consistent structure |

### How Customization Works Technically

Bank's repo imports our base packages as npm dependencies:

```json
// bank-x-backend/package.json
{
  "dependencies": {
    "@corebridge/auth": "^1.2.0",
    "@corebridge/onboarding": "^1.2.0",
    "@corebridge/accounts": "^1.2.0",
    "@corebridge/core-adapter": "^1.2.0"
  }
}
```

Their `app.module.ts` imports base modules and swaps what they need:

```typescript
// Bank X uses a custom SMS gateway
@Module({
  imports: [
    AuthModule,              // base, no changes
    OnboardingModule,        // base, no changes
    AccountsModule,          // base, no changes
    CoreAdapterModule.forRoot({
      mappers: BankXMappers  // their custom field mappings
    }),
    NotificationsModule.forRoot({
      smsProvider: BankXSmsGateway  // their SMS integration
    })
  ]
})
export class AppModule {}
```

---

## 4. How We Ship Updates to Banks

### Scenario: Bug Fix in Base

```
1. We fix the bug in @corebridge/onboarding (for example)
2. Publish new version: @corebridge/onboarding@1.2.1
3. Notify bank: "Version 1.2.1 of onboarding fixes [bug description]"
4. Bank updates their package.json: "@corebridge/onboarding": "^1.2.1"
5. Bank runs: npm install
6. Bank runs their tests
7. Bank deploys to staging → production
```

**Key point:** the bank's customizations are in a separate layer. Updating a base package does NOT overwrite their customizations. It's like updating a library — your code that uses the library stays the same.

### Scenario: New Feature in Base

```
1. We build "virtual card ordering" in @corebridge/cards@2.0.0
2. Update contracts: new endpoints in openapi.yaml
3. Publish new package version
4. Notify banks: "Cards 2.0.0 adds virtual card support. Here's the migration guide."
5. Bank decides: do they want this feature?
   - Yes → update package, configure for their PSP, test, deploy
   - No → stay on current version, nothing breaks
```

### Scenario: Breaking Change in Base

We use semantic versioning:
- `1.2.1` → bug fix (safe to update)
- `1.3.0` → new feature (safe to update, backward compatible)
- `2.0.0` → breaking change (requires migration)

For breaking changes:
1. We publish a **migration guide** explaining what changed and how to update
2. We keep the old version available (banks don't have to upgrade immediately)
3. During the bank's next engagement or support window, we help them migrate

### Version Lifecycle

| Version State | What It Means |
|---------------|--------------|
| Active | Current recommended version, receives bug fixes |
| Maintained | Still receives critical security fixes, no new features |
| End of Life | No more updates, bank should upgrade |

We support the last 2 major versions. Example: when v3.0 launches, v1.x becomes EOL.

---

## 5. How Banks Get New Features After Launch

### Option A: Self-Service (Unlikely for Iraqi Banks)

Bank's dev team pulls new package version, integrates, tests, deploys themselves.
Realistic only if the bank has a capable IT team.

### Option B: Paid Update Engagement (Most Common)

Bank pays Corebridge for a mini-engagement:
- 1-4 weeks depending on feature complexity
- We update their packages, test, deploy
- Similar to customization but shorter

### Option C: Included in Support Contract

If the bank has an ongoing support/maintenance contract:
- Bug fixes and security patches are included
- Minor features may be included
- Major features are a separate engagement

---

## 6. Ongoing Support Model

### Tier 1: Basic Support (Included with License)

- Bug fix releases (bank applies them)
- Security patches
- Email support (48h response)

### Tier 2: Managed Support (Paid Monthly)

- Everything in Tier 1
- We apply updates for them
- 24h response time
- Monitoring review (monthly)
- Priority bug fixes

### Tier 3: Full Managed (Paid Monthly, Premium)

- Everything in Tier 2
- We manage their CI/CD and deployments
- 4h response time for critical issues
- Quarterly health checks
- Included minor feature updates

---

## 7. What Happens When We Change Base Code

This is the most common concern. Here's every scenario:

### We Fix a Bug

- Publish patch version (e.g., 1.2.0 → 1.2.1)
- Bank's customizations: **unaffected**
- Bank updates when ready, no rush unless it's a security fix

### We Add a Feature

- Publish minor version (e.g., 1.2.0 → 1.3.0)
- Bank's customizations: **unaffected**
- New feature is opt-in (feature flag or new module they import)

### We Change an API Endpoint

- Publish major version (e.g., 1.x → 2.0.0)
- Bank's customizations: **may need updating**
- We provide migration guide
- Old version still works until bank is ready to migrate

### We Change the CoreAdapter Interface

- This is the riskiest change — affects all banks
- We avoid this unless absolutely necessary
- If needed: major version bump + detailed migration guide + we help each bank migrate

### We Change the Database Schema

- Migration scripts included with the update
- Bank runs migration on staging first, then production
- We provide rollback scripts in case of issues

### We Change Design Tokens

- Only affects banks that use our default theme
- Banks with custom branding: **unaffected** (they override tokens)

---

## 8. How AI Agents Help With Bank Customization

During the 3-month bank engagement, AI agents accelerate the work:

### ICSFS Mapping (Week 1-2)

1. We give the agent the bank's WSDL files
2. Agent reads the WSDL + our CoreAdapter interfaces
3. Agent generates mapper files (CIF_NO → customerId, etc.)
4. We review and adjust

### SMS Gateway Integration (Week 1-2)

1. We give the agent the bank's SMS API documentation
2. Agent reads our notification module interface
3. Agent generates the SMS provider implementation
4. We test with the bank's sandbox

### Mobile App Branding (Month 2)

1. We update design tokens with bank's colors/fonts
2. Agent regenerates theme
3. Agent updates screens with new branding
4. We review screenshots

### Bank-Specific Features (Month 2)

1. We describe the feature in a spec
2. Agent reads the spec + existing codebase + contracts
3. Agent implements in the bank's `/extensions/` folder
4. We review and test

---

## 9. Revenue Timeline per Bank

```
Month 0:    License fee (one-time)
Month 1-3:  Setup + customization fee (one-time)
Month 4+:   Support contract (monthly recurring)
Ad-hoc:     Feature update engagements (per project)
```

Each bank becomes recurring revenue through support contracts and periodic update engagements.
