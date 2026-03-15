# Corebridge — Business & Exit Strategy

---

## 1. The Model

Build a complete digital banking platform for one Iraqi bank on top of their ICS BANKS core. Sell the company (Corebridge) + all software IP to that bank.

**Not** a multi-bank SaaS. **Not** a license model. One bank, one exit.

---

## 2. What the Bank Gets

- Full ownership of the company and all source code
- Working digital banking platform (mobile app + backoffice + backend)
- Connected to their ICS BANKS core via SOAP
- KYC, sanctions screening, notifications, audit trail — all integrated
- On-prem deployed on their infrastructure
- CI/CD pipeline, monitoring, backup scripts
- Full documentation and AI agent workflow for future development

---

## 3. Build Phase

### Month 1: Core Integration

```
Week 1-2:
├── Get ICSFS WSDL files from bank's core team
├── Map their ICSFS fields to Corebridge models (CoreAdapter mappers)
├── Basic connectivity: can we talk to their core?
├── Integrate their SMS gateway
└── Configure KYC provider (Sumsub/Regula/GBG)

Week 3-4:
├── Configure sanctions provider
├── Test: onboarding flow end-to-end against their test core
├── Test: login, balance, transactions against their test core
└── Fix integration issues
```

### Month 2: Customization & Polish

```
Week 5-6:
├── Customize onboarding steps per bank policy
├── Bank-specific document requirements
├── Apply bank branding (logo, colors, typography) to mobile + backoffice
├── Customize notification templates
└── Bank-specific feature requests

Week 7-8:
├── Bank's compliance team reviews the system
├── Backoffice training for bank employees
├── UAT with bank's team
├── Fix issues from UAT
└── Performance testing against their core
```

### Month 3: Production & Handover

```
Week 9-10:
├── Set up production environment on bank's servers
├── Production deployment
├── Configure CI/CD, monitoring, alerting
├── Set up backup scripts
└── Smoke testing on production

Week 11-12:
├── Soft launch (limited users or internal employees)
├── Monitor and fix production issues
├── Full launch
├── Handover documentation
└── Transition support
```

---

## 4. What Makes This Sellable

| Asset | Value to Bank |
|-------|---------------|
| Working product | Immediate digital banking capability |
| Source code ownership | No vendor lock-in, full control |
| Clean architecture | Their team (or contractors) can maintain and extend |
| AI workflow | Accelerated future development with AI agents |
| Documentation | Complete technical docs for handover |
| On-prem deployment | Meets CBI regulatory requirements |
| ICSFS integration | Already connected to their core — hardest part done |

---

## 5. Exit Structure

The bank acquires Corebridge (the company), which includes:

- All source code and IP
- Deployment and infrastructure setup
- Documentation and AI workflow
- Optional: transition period where Corebridge team supports the bank (3-6 months post-acquisition)

---

## 6. Why a Bank Would Buy This

- Building from scratch takes 1-2 years and costs significantly more
- Finding developers who understand both NestJS and ICSFS SOAP integration in Iraq is hard
- They get a working product, not a proposal
- CBI is pushing banks toward digital — time pressure
- They own everything, no recurring license fees
- Clean codebase with AI-assisted development means lower future maintenance cost

---

## 7. Pre-Acquisition Checklist

Before the sale:

- [ ] Product is live and serving real users
- [ ] All source code in a clean, documented Git repo
- [ ] No third-party dependencies with restrictive licenses
- [ ] All secrets rotatable, no hardcoded credentials
- [ ] Bank's team has been trained on basic operations
- [ ] Compliance review passed (CBI requirements)
- [ ] Performance benchmarks documented
- [ ] Backup and disaster recovery tested
