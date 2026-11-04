# 12 — GoldVuee GitHub, Salesforce DX, Deployment and Release Guide

**GV-DOC-12 | v1.0 | Proposed procedure; no GitHub remote, Salesforce deployment or release executed as part of this document.** Use the existing Developer Edition org and LWR Experience Cloud site. The documentation package can be committed to the repository once GitHub access is connected/authorized; **a local file bundle is not proof of a GitHub push**.

## 1. Repository layout (one folder, no giant primary document)

```text
GoldVuee/                              # GitHub repository root (recommended)
├── 01_PROJECT_OVERVIEW.md
├── 02_SYSTEM_ARCHITECTURE.md          # CANONICAL SOURCE OF TRUTH
├── 03_DATA_MODEL.md
├── 04_SECURITY_ARCHITECTURE.md
├── 05_FLOW_DOCUMENTATION.md
├── 06_APEX_DOCUMENTATION.md
├── 07_LWC_DOCUMENTATION.md
├── 08_INTEGRATION_DOCUMENTATION.md
├── 09_UI_UX_DESIGN.md
├── 10_IMPLEMENTATION_ROADMAP.md
├── 11_TESTING_DOCUMENTATION.md
├── 12_DEPLOYMENT_GUIDE.md
├── 13_DEVELOPMENT_LOG.md
├── README.md                            # Add during project bootstrap; links above
├── .gitignore
├── sfdx-project.json
├── sf-config/                            # optional nonsecret sample settings only
├── config/project-scratch-def.json       # optional later; no need to recreate DE org
├── force-app/main/default/
│   ├── classes/                          # seven P0 classes + five test classes
│   ├── lwc/                              # P0 components from Doc 07
│   ├── objects/                          # 5 P0 custom objects, Product2 fields
│   ├── permissionsets/                   # GV_Catalogue_Manager, etc.
│   ├── profiles/                         # only metadata changes carefully reviewed
│   ├── layouts/                          # Lightning admin record forms
│   ├── flexipages/                       # Lightning Record Pages
│   ├── tabs/                             # Category, Inventory, Order, Payment
│   ├── applications/                     # GoldVuee Admin if implemented
│   ├── flows/                            # optional GV_Order_Confirmed_Notification P1
│   ├── experiences/ and/or digitalExperiences/ # site metadata as supported by org
│   ├── networks/                         # site membership/network config as supported
│   ├── sites/                            # relevant site metadata only
│   ├── staticresources/                  # fallback placeholder images only
│   ├── email/                             # optional P1 notification template
│   └── customMetadata/                  # only nonsecret configuration, if used
└── scripts/                              # vetted metadata/data seeds and tests
```

**Note:** `force-app` directories are created as metadata is retrieved or implemented; this documentation does not manufacture Salesforce metadata or claim all directories exist. Experience Cloud metadata shape varies by site and CLI/API version; retrieve actual site-specific metadata before selecting package XML. **Never deploy unrelated org-wide profile settings or overwrite a live site inadvertently.**

## 2. GitHub workflow and branch policy

Branch model: `main` protected, small task branches `feature/schema`, `feature/catalogue`, `feature/checkout`, `fix/payment-replay`, `docs/architecture`. Pull requests include summary, affected components, TC IDs, screenshots/redacted evidence and any updated docs. Direct changes to canonical architecture require corresponding updates to Docs 03–13 in the same PR.

Conventional commit suggestions:

```text
feat(schema): create inventory and order objects
feat(site): publish guest-safe catalog components
feat(checkout): persist repriced pending orders
feat(payment): confirm Razorpay capture exactly once
test(security): cover cross-customer order denial
docs(architecture): align payment and inventory contracts
fix(catalogue): filter unpublished products
```

Recommended tags `v0.1.0-mvp` **after** all P0 test gates pass, `v0.2.0` for post-MVP enhancements. Add a small `README.md` once GitHub repo exists linking all 13 docs and stating demo/test mode status. Do not commit generated deployment results or org auth caches.

## 3. Minimum `.gitignore` and secret hygiene

```gitignore
# Salesforce auth / local settings
.sf/
.sfdx/
.env
.env.*
!.env.example
*.key
*.pem
*.p12
*.pfx
*.jks
# local output and caches
node_modules/
coverage/
.tmp/
.DS_Store
*.log
# never check in payment credentials or test records with PII
secrets/
private/
```

**Secret scan before every push:** scan tracked/untracked files for Razorpay test/live secrets, Salesforce session/auth JSON, connected app client secrets, webhook secrets, phone/email/order PII, raw HTTP transcripts. Use an automated scanner such as `gitleaks` if approved and available; otherwise documented human review is the minimum. Only `razorpayKeyId` (public test key) may be sent to browser. **Do not store any actual `key_secret` in Custom Metadata checked into Git.**

## 4. Salesforce CLI setup — existing org only

Use modern Salesforce `sf` CLI (confirm version and actual commands on the current installed CLI):

```bash
sf --version
sf org list
sf org display --target-org goldvuee-dev
# Only if existing alias is absent, authenticate the already existing org:
sf org login web --alias goldvuee-dev --set-default
# Confirm source/project status:
sf project retrieve start --target-org goldvuee-dev --metadata CustomObject:GV_Category__c
sf project deploy validate --target-org goldvuee-dev --source-dir force-app --test-level RunLocalTests
sf project deploy start --target-org goldvuee-dev --source-dir force-app --test-level RunLocalTests
sf apex run test --target-org goldvuee-dev --test-level RunLocalTests --result-format human --wait 30
```

**Important:** sample `retrieve` should run **only after the object exists**; it will fail for unbuilt metadata. The alias `goldvuee-dev` is illustrative; actual alias needs verification, and `sf project` commands require a valid SFDX project with `sfdx-project.json`. `deploy validate` and deploy options may depend on API target and supported test level. For a new non-production scratch org, only create one if explicitly approved; do **not** replace the existing developer org.

## 5. Recommended release steps

### Before deployment

1. Confirm checked-out commit/branch and last passing CI; verify Apex tests and required source metadata consistent with Doc 02 names.
2. Export existing org metadata (especially site, profiles, permission sets and sharing settings) for documented rollback; capture org metadata revision and site publication state.
3. Verify configured external-user license, guest profile and sharing rules; plan smallest scope metadata deploy. Confirm `GoldVuee Retail` Standard/retail Pricebook data seed and existing Contacts are not accidentally overwritten.
4. Ensure every nonsecret environment-specific setting can be re-created manually; escrow secrets only in approved Salesforce secret configuration, **not** source files or screenshots.
5. Pre-stage test-mode Razorpay key in Named Credential/External Credential per Doc 08; confirm not live mode.

### Deploy + configure

1. `sf project deploy validate ...` with project-owned Apex test suites (or selected test classes appropriate to environment), review metadata conflicts, destructive changes and coverage.
2. `sf project deploy start ...` to authorized org. Monitor and archive deployment ID and test results.
3. Configure/verify postdeploy settings not guaranteed by source deploy: site membership, Experience Builder routes, guest profile access, guest sharing reports, CMS workspace/channel/public media, external credentials/principal access, test HMAC secret access controls, custom domain not required.
4. Publish **existing** GoldVuee LWR site. Check public URL as incognito with cache cleared and actual site revision recorded.
5. Seed synthetic categories/Products/PricebookEntries/Inventory with a repeatable ordered data script; never overwrite existing customer/payment real data or delete historical orders.
6. Execute smoke TC-01, 03, 05, 09, 11, 17, 21, 25, 26, 29, 30, 33; then full required TC-01–35 as release gate.
7. Record deployment commit, org alias, date/time, test data IDs (redacted), before/after stock, gateway test transaction ID (masked), and screenshots in Doc 13 or a referenced private evidence location.

### Release go/no-go checklist

- [ ] Site still public and route navigation works
- [ ] Guest cannot read private Salesforce data
- [ ] Two customers cannot read each other's orders/contacts
- [ ] Catalog prices equal Retail Pricebook entries
- [ ] Native registration/login/logout and reset work
- [ ] Razorpay **test** callback HMAC and `captured` server check succeed
- [ ] Browser-only success fails closed
- [ ] Captured payment has exactly one stock deduction
- [ ] Duplicate and concurrent payment tests pass
- [ ] No unapproved P2/B2B features or live credentials
- [ ] All P0 test cases marked PASS with evidence
- [ ] Architecture and field dictionary match deployed metadata
- [ ] Git commit/tag and rollback plan recorded

Checklist is intentionally unchecked. **Do not tag a release until signed off.**

## 6. Deployment rollback/recovery

| Failure | Response | Data caution |
|---|---|---|
| Metadata deploy failed | Stop, inspect validation errors, correct branch and redeploy; Salesforce metadata deploy may roll back depending deployment transaction behavior | Do not run manual data patch blindly |
| Site broken after publishing | Revert to prior captured Experience Builder revision / redeploy prior known-good metadata, republish and smoke test | Avoid deleting site or its auth config |
| Permission leak | Immediately remove exposed Apex/guest permission or temporarily disable public site while remediating; audit access logs | Treat exposure as security incident, not UI bug |
| Payment verified but stock/Order inconsistent | Stop checkout; compare gateway capture and GV_Payment/Order/Inventory records; flag Payment Review, reconcile manually | Never replay decrement without state/lock/unique check |
| Product price/catalogue incorrect | Revert PricebookEntry/Product publication data via controlled restore and test | Preserve order price snapshots |
| Wrong credentials/CSP | Stop gateway checkout; restore secure Named Credential/principal/CSP and re-test only in test mode | Never expose secret by logging |

**Metadata rollback is not data rollback.** Reverting code does not undo payments or order stock movements; reconcile from gateway records and preserve immutable historical order details. Production migration would need formal backups, incident response and compliance approval.

## 7. Environment matrix

| Setting | Existing Developer Edition (MVP) | Future production (NOT approved) |
|---|---|---|
| Site | Existing `.../goldvuee` LWR site | Domain/CSP/legal reviewed separately |
| Payment | Razorpay TEST ONLY | Razorpay live after security/compliance hardening |
| API auth | Named `GV_Razorpay_Test` / `GV_Razorpay_Test_Auth` | Separate live credential, never reuse test |
| Email | Test deliverability / optional P1 Flow | Validated sender, monitoring, unsubscribe requirements as applicable |
| Users | Synthetic external test customers | Licensed identity architecture, privacy compliance |
| Images | Public CMS images or verified site placeholder | Licensed media / CDN/cache policy |
| Webhooks | Not deployed | Authenticated raw-body HMAC + event dedupe + reconciliation required |

## 8. Source/document consistency review before PR merge

Automated text review should confirm exactly **13 numbered docs**, links resolve, and canonical objects/classes/Flow names match Doc 02. Review that `GV_Order__c` (not a second standard `Order`) is the business order; `PricebookEntry.UnitPrice` is authoritative; `GV_Razorpay_Test` is test-only; no `GV_Cart__c` or wholesale object is accidentally part of P0; all functionality FR IDs have TC IDs. Run the verification script provided alongside the documentation package if available, or manually review.

---
[Architecture](02_SYSTEM_ARCHITECTURE.md) · [Roadmap](10_IMPLEMENTATION_ROADMAP.md) · [Tests](11_TESTING_DOCUMENTATION.md) · [Development log](13_DEVELOPMENT_LOG.md)