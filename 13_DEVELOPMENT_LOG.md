# 13 — GoldVuee Development Log, MVP Tracker and Post-MVP Backlog

**GV-DOC-13 | v1.0 | Initial documentation baseline dated 2026-11-04.** This is an evidence ledger, not a retroactive claim that implementation, test execution, GitHub push, or Salesforce deployment occurred. Update daily after actual work.

## 1. Status definitions and source evidence

| Label | Meaning | May count as done? |
|---|---|---|
| `Reported Configured` | Owner says setting was configured; not revalidated in this documentation exercise | No |
| `Not Started` | No verified implementation evidence | No |
| `In Progress` | Work underway; acceptance not met | No |
| `Implemented — Untested` | Metadata/code appears present, tests missing | No |
| `Blocked` | Dependency prevents progress, logged with action | No |
| `Test Failed` | Reproducible failure logged | No |
| `Verified` | Requirements + acceptance tests passed with evidence, date and commit | Yes |

No test is `PASS` unless [11_TESTING_DOCUMENTATION.md](11_TESTING_DOCUMENTATION.md) contains a real observed result and evidence location.

## 2. Baseline inventory — as of documentation creation

| Area | Owner-reported position | Verification state | Next action |
|---|---|---|---|
| Developer Edition org / My Domain / Salesforce CLI | Configured | Unverified independently | Day 1 org audit / metadata inventory |
| Existing LWR public site created/published | Configured | Owner reports incognito opened | Day 1 & 3 independent incognito and data security audit |
| `GoldVuee Customers` Account | Created | Unverified independently | Verify existence, Contact ownership and shared Account privacy |
| `GoldVuee Customer` profile and site membership | Configured | Unverified independently | Check license, permissions and login test |
| Site guest profile `GoldVuee Profile` | Exists | Unverified independently | Inspect class/object/FLS/sharing access |
| Self-registration set to Default Page | Configured | **Function not tested** | Actual separate User/Contact registration/login/reset |
| Product/category/inventory/order/payment schema | Not reported implemented | Not started/unknown without org metadata | Day 1 build P0 model |
| Apex/Flow/LWC/payment/commerce functionality | Not reported implemented | Not started/unknown without org metadata | Days 3–6 implementation |
| End-to-end and security test evidence | None supplied | Not run | Day 7 test evidence |
| GitHub remote/repository | No connected repo or remote provided | Not verified | Connect/push after explicit access/remote confirmation |

**Baseline site URL:** https://goldvuee-dev-ed.develop.my.site.com/goldvuee (owner supplied, no live configuration introspection performed while drafting).

## 3. Documentation decision log — 2026-11-04

| Decision | Outcome | Why | Approval/evidence |
|---|---|---|---|
| SSOT | `02_SYSTEM_ARCHITECTURE.md` owns naming, states, component roles | Prevent contradictions across 13 files | Proposed; project owner review required |
| Reuse standard objects | Account, Contact, User, Product2, Pricebook2/Entry | Native product/customer model | Proposed |
| Custom order | `GV_Order__c`, `GV_Order_Item__c` P0 | Pending-payment lifecycle simpler than standard Order activation | Proposed ADR-03 |
| Custom inventory/payment | `GV_Inventory__c`, `GV_Payment__c` P0 | Controlled stock and gateway idempotency | Proposed |
| MVP cart | Browser `gv_cart_v1`, server reprices | Minimize custom objects, no guest write | Proposed |
| Customer auth | Native Experience Cloud existing site | Avoid custom password UI | Reported configured; not yet tested |
| Transaction owner | Apex, zero required Flows | Payment/stock atomicity | Proposed |
| Optional notification | `GV_Order_Confirmed_Notification` P1 | Small meaningful Flow only | Planned, not built |
| B2B/wishlist/refunds | Track B P2 | Protect seven-day single-journey milestone | Planned |
| Secret strategy | External Credential for API auth; additional test-only restricted HMAC key needed | Credential API doesn't automatically expose key to Apex Crypto | **Unresolved implementation/security gate** |
| Site image system | Salesforce CMS public media if supported | No Azure/additional storage | CMS/License unverified |

**Architecture documents created as a local deliverable do not constitute Salesforce implementation and do not count as a Day 1 sprint completion.**

## 4. Seven-day MVP working log (complete after real activity)

| Day | Planned focus | Priority | Dependencies | Work status | Test result | Docs updated | Git commit |
|---|---|---|---|---|---|---|---|
| D1 | Org audit, schema, identity license | P0 | Existing org access | Not Started | Not run | Blueprint drafted only | — |
| D2 | 15 categories, sample SKUs, Retail prices, CMS, stock | P0 | D1 | Not Started | Not run | Blueprint drafted only | — |
| D3 | Public site LWCs, secure catalogue | P0 | D2 + guest sharing | Not Started | Not run | Blueprint drafted only | — |
| D4 | Registration/login, A/B isolation, local cart | P0 | D3 + external license | Not Started | Not run | Blueprint drafted only | — |
| D5 | Pending order and own history | P0 | D4 + Retail prices | Not Started | Not run | Blueprint drafted only | — |
| D6 | Razorpay test capture, HMAC, idempotent stock | P0 | D5 + secret/CSP approval | Not Started | Not run | Blueprint drafted only | — |
| D7 | Security regression, E2E, source deploy, Git release | P0 | D1–6 verified | Not Started | Not run | Blueprint drafted only | — |

**When completing each day**, use the following dated entry structure. Never convert attempted tasks into completed tasks without acceptance criteria and proof.

### Daily entry template

```text
Date / local timezone:
Day: D1 ... D7
Objective and owner:
Org alias / experience URL / Git branch and commit:
Completed and VERIFIED (with TC IDs + evidence):
Implemented but NOT tested:
Pending work:
Blockers and proposed decision:
New bugs (GV-BUG-###, severity):
Tests run (PASS / FAIL / BLOCKED + actual result):
Security/secret checks:
Metadata deployed/site published? Evidence:
Documentation files changed:
Next day's objective:
Sign-off (owner + date):
```

## 5. Post-MVP backlog (not counted toward MVP)

| Feature | Business value | Complexity | Dependencies | Priority | Planned phase | Status |
|---|---|---|---|---|---|---|
| `GV_Wishlist_Item__c` + `goldVueeWishlist` | Saved favorites | Medium | Customer identity and isolation | P2 | B2 | Not Started |
| Cross-device cart/addresses | Better return shopping | Medium | Cart persistence data design | P2 | B2 | Not Started |
| B2B Account approval/GSTIN | Enterprise customer onboarding | High | External licensing and private sharing | P2 | B3 | Not Started |
| Business pricebooks and MOQ | Wholesale pricing | High | B2B approval and checkout service rules | P2 | B3 | Not Started |
| Inventory movement ledger | Auditability | Medium | Stock transaction and returns design | P2 | B4 | Not Started |
| Inventory reservations/multiwarehouse | Less paid/no-stock risk | High | Correct lock/idempotency core | P2/P3 | B4 | Not Started |
| Low stock scheduled alert Flow | Operations | Low/Medium | Inventory threshold policy | P2 | B4 | Not Started |
| Razorpay webhooks/reconciliation | Resilience | High | Safe public endpoint, raw-body HMAC, event dedupe | P2 | B5 | Not Started |
| Refund/return handling | Full post-sale lifecycle | High | Gateway refund policy and stock movement | P2 | B5 | Not Started |
| Tax/shipping engine | Commercial correctness | High | Business/legal tax and shipping policy | P2 | B5 | Not Started |
| Salesforce Reports/Dashboards | Visibility | Medium | Clean production-like data and report types | P2 | B6 | Not Started |
| Record Types, advanced pages/list views | Admin learning & UX | Low/Medium | Verified P0 permissions/layouts | P1/P2 | B1 | Not Started |
| Reviews/loyalty/promotions | Retention | High | More commerce rules | P3 | Later | Not Started |
| Custom domain | Brand production readiness | Medium | Security/production release | P3 | Later | Not Started |

## 6. Change history (documentation)

| Date | Version | Changes | Scope |
|---|---|---|---|
| 2026-11-04 | 1.0 | Proposed architecture, data dictionary, security, flow/Apex/LWC contracts, payment plan, 7-day sprint, test cases, release plan and initial log | **Documentation only**, no Salesforce changes claimed |

## 7. Final release sign-off record (blank until real verification)

| Item | Recorded value |
|---|---|
| Release version/tag | Not assigned |
| Deployment ID | Not assigned |
| GitHub URL / commit | Not connected / not assigned |
| Org/Site tested | Not independently verified |
| P0 test pass evidence | Not run |
| Critical open bugs | Not assessed |
| Named Credential/test HMAC secret approved | Not verified |
| Customer A/B sharing isolation | Not verified |
| Payment success, replay, low-stock proof | Not run |
| Project owner sign-off | Pending |
| Release status | **NOT RELEASED** |

---
[Roadmap](10_IMPLEMENTATION_ROADMAP.md) · [Tests](11_TESTING_DOCUMENTATION.md) · [Deployment](12_DEPLOYMENT_GUIDE.md)