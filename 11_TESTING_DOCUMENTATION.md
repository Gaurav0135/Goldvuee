# 11 — GoldVuee Test Plan, Acceptance Cases and Evidence Register

**GV-DOC-11 | v1.0 | Test cases drafted; none executed or independently observed.** Until a test is actually run, `Actual result = Not run` and `Status = NOT RUN`. Do not label a test passed from architecture text, mockup, code existence or a screenshot alone.

## 1. Quality gates and test levels

**Unit:** Apex `@isTest` classes using `HttpCalloutMock` for Razorpay, separate data fixtures (SeeAllData=false), positive/negative/authorization/large input; LWC Jest tests when supported.  
**Configuration:** Object fields, relationships, category slug uniqueness, permission sets, FLS, OWD, guest rules, native registration and profile membership.  
**Integration:** Actual test gateway checkout and server-side signature+captured status, payment error cases, repeated callback, stale inventory, CSP constraints.  
**System/E2E:** guest incognito → signed-in order → payment → inventory exactly once → own history.  
**Security:** guest no private access, A/B customer isolation, unauthorized Order Id, no payment/secret leakage, staff least privilege.  
**Usability:** responsive 375px/1440px, keyboard navigation, alt text, form/error feedback.

**Release criteria:** all P0 test IDs TC-01–TC-35 pass, plus no Critical/High bugs; Project Apex tests meet platform deployment requirements and target >=85% coverage; 100% of declared high-risk gateway/idempotency/ownership branches tested. Any optional Flow tests TC-36–38 become required **if the Flow is activated**. The test org and Experience site must be explicit; screenshots without run context are insufficient.

## 2. Test execution protocol

For each run capture `Test ID`, test date/time & timezone, Salesforce org alias / non-sensitive org ID, Experience site URL, deployment commit hash, user/role (Guest/Customer A/Customer B/etc.), test input fixture/SKU, precise steps, **actual observed result**, `PASS/FAIL/BLOCKED/NOT RUN`, Salesforce test output/record Ids (redacted as needed), screenshots/network captures without credentials/PII, bug reference and retest date. Use a reproducible synthetic SKU `GV-W-RNG-001` and sample unit price ₹2500 in the test fixture; do not assume org already contains it.

**Baseline:** all cases below were designed from requirements, **not executed**. The `Actual / status` column is intentionally `Not run / NOT RUN` for each.

## 3. Detailed test cases

| Test ID | Module | Preconditions | Steps | Expected result | Actual / status |
|---|---|---|---|---|---|
| TC-01 | Public site | Published LWR site | Open `/` in fresh incognito with no Salesforce auth | Home renders and no login required | Not run / NOT RUN |
| TC-02 | Guest security | Guest profile configured | Attempt checkout, order/payment controller and private record access unauthenticated | Denied; no private data or DML | Not run / NOT RUN |
| TC-03 | Collection navigation | 15 category records | Open Men/Women/Kids in incognito | Three collection routes work; five categories each | Not run / NOT RUN |
| TC-04 | Dynamic categories | Active/hidden categories | Change Category name/active in Salesforce and reload | Public menu reflects data; hidden not shown | Not run / NOT RUN |
| TC-05 | Catalogue and detail | Published product with price/CMS image | Open listing then product | Correct name/image/INR price/specs and availability flag | Not run / NOT RUN |
| TC-06 | Catalogue privacy | Draft product / inactive category | Query via UI/API by product Id and category | Unpublished/hidden product not returned | Not run / NOT RUN |
| TC-07 | Catalogue price/images | Active Retail price; valid image key | Compare UI to PricebookEntry and CMS image | Same price, public image only, broken image fallback | Not run / NOT RUN |
| TC-08 | Search/filter/sort | At least 8 sample products | Search, select category, sort, try invalid sort injection | Correct scoped results, safe allowlist, 24 max page size | Not run / NOT RUN |
| TC-09 | Registration/auth | External license, self-registration enabled | Register user, login, logout, reset password | Native auth works; unique Contact and linked User | Not run / NOT RUN |
| TC-10 | Customer isolation basics | Two customer logins A/B | View own profile, attempt peer Contact/account read | Only own data, no leakage through shared Account | Not run / NOT RUN |
| TC-11 | Cart mutations | Browse product | Add, update quantity, remove, reload | Cart `gv_cart_v1` persists IDs/qty only; count/subtotal estimate updates | Not run / NOT RUN |
| TC-12 | Cart limits | SKU stock fixture | Enter 0, -1, >10, noninteger; corrupt browser cart JSON | Safely rejected/cleared; no forged price accepted | Not run / NOT RUN |
| TC-13 | Auth/checkout route | Guest cart filled | Open checkout then login | Login required, cart retained, totals revalidated | Not run / NOT RUN |
| TC-14 | Checkout authorization | Two users; API inspectable | Add foreign ContactId or OrderId to checkout payload | Server ignores/denies foreign IDs; User.ContactId authoritative | Not run / NOT RUN |
| TC-15 | Stock validation | `GV_On_Hand__c=1` | Order qty 2 | `INSUFFICIENT_STOCK`; no pending order or stock mutation | Not run / NOT RUN |
| TC-16 | Input validation | Any valid SKU | Submit malformed address, bad postal code, duplicate lines, missing price | Bounded validation; safe error, no orphan items | Not run / NOT RUN |
| TC-17 | Pending order | Customer logged in, 2 units @₹2500 | Submit valid 2-line or 1 SKU qty2 checkout | One Pending Payment order; correct Contact, item snapshots, ₹5000 total | Not run / NOT RUN |
| TC-18 | Price integrity | Client cart stale price; Retail updated | Start checkout with old price | Server uses new PricebookEntry price; requires reconfirmation | Not run / NOT RUN |
| TC-19 | Razorpay order create | Named Credential test mode; pending order | Start payment, simulate gateway timeout and retry | Valid gateway order/attempt; retry safe and creates fresh attempt | Not run / NOT RUN |
| TC-20 | Secret hygiene | Source/metadata/site available | Scan Git, JS, network, logs for `key_secret` | Test secret absent; browser sees public key ID only | Not run / NOT RUN |
| TC-21 | Test payment success/failure | Approved test card/test UPI or Razorpay test method | Complete success then separate failed payment | Success only after server verification; failed stays unconfirmed, no stock deduction | Not run / NOT RUN |
| TC-22 | Forged signature/frontend | Pending order and mock captured state | Submit bad/absent HMAC or manually set JS success flag | Verification denied, stock unchanged | Not run / NOT RUN |
| TC-23 | Authorized-only/abandoned | Mock `authorized`, browser closes | Verify or refresh own order | Not Confirmed; no deduction; pending state shown | Not run / NOT RUN |
| TC-24 | Provider mismatch | Payment mock wrong order/amount/currency | Submit verifyPayment | Denied/Review with no confirmation or inventory change | Not run / NOT RUN |
| TC-25 | Exactly-once purchase | SKU stock=5; qty=2; valid captured payment | Verify; inspect order/payments/inventory | Confirmed, Captured, stock=3, one adjustment | Not run / NOT RUN |
| TC-26 | Replay protection | TC-25 completed | Call `verifyPayment` twice more with same ids/signature | No duplicate payment ID or stock decrement; stock still 3 | Not run / NOT RUN |
| TC-27 | Captured/stock race | Pending qty2, stock reduced to1 before capture | Verify valid captured payment | Order Payment Review; attempt Captured; stock stays1, never negative | Not run / NOT RUN |
| TC-28 | Concurrent transactions | Two pending orders competing for stock1 | Trigger concurrent valid provider-verification transactions | At most one sale deducted; second Review; no negative inventory | Not run / NOT RUN |
| TC-29 | Own order history | Customer A confirmed order | Login as A; open My Orders/Order Result | Own order and line snapshots match Salesforce | Not run / NOT RUN |
| TC-30 | Guest & horizontal security | A order known, B logged in | Guest and B open A's order URL / Apex method | Safe denial; no PII or status leakage | Not run / NOT RUN |
| TC-31 | Staff permissions | Product, inventory, order managers seeded | Each attempts allowed/disallowed create/update | Only assigned CRUD/FLS actions work | Not run / NOT RUN |
| TC-32 | Order lifecycle | Confirmed order | Order Mgr transition Processing→Shipped→Delivered; try illegal status | Allowed changes succeed; illegal transition denied | Not run / NOT RUN |
| TC-33 | Responsive UI | Site deployed | Test at 375px and 1440px, product/detail/cart/checkout | No horizontal overflow, readable labels/buttons, images fit | Not run / NOT RUN |
| TC-34 | Accessibility/CSP | Site deployed, browser devtools | Keyboard nav, alt/error labels, inspect CSP errors/localStorage | Usable focus/accessibility; no stored personal/secret data | Not run / NOT RUN |
| TC-35 | Full shopping journey | All P0 gates green; test mode | Incognito browse → register/login → qty2 cart → checkout → Razorpay capture → history | Single verified end-to-end receipt; inventory exactly once; evidence saved | Not run / NOT RUN |
| TC-36 | Optional Flow P1 | Flow active, Contact has email | Transition Pending→Confirmed once | One email request; Flow does not mutate stock/payment | Not run / NOT RUN |
| TC-37 | Optional Flow P1 | Confirmed order | Edit non-status field; then Confirmed→Processing | Zero repeated confirmation messages | Not run / NOT RUN |
| TC-38 | Optional Flow P1 | No recipient/email send fault | Trigger eligible transition | Order remains confirmed, safe failure observation | Not run / NOT RUN |

## 4. Sample Apex test assertion contracts

| Assertion group | Examples |
|---|---|
| Pricing | Server totals equal `Σ(PricebookEntry.UnitPrice × qty) + tax + shipping`; browser price ignored |
| Identifiers | Product SKU unique, category slug unique, Inventory Product Key unique, Razorpay Order/Payment Id unique |
| Owner | `User.ContactId` matches stored `GV_Order__c.GV_Contact__c` before viewing/mutating |
| Payment | Valid callback HMAC; fetched `captured`; `payment.order_id = attempt.gatewayOrderId`, amount/currency exact |
| Inventory | `newOnHand = oldOnHand - qty`, never below 0, one capture once; `Payment Review` when paid-no-stock |
| Bulk/input bounds | Validated 20 cart lines, 10 quantity max, up to 24 catalogue entries/page; no SOQL/DML in loops |
| Privacy | Other Contact's Order Id returns generic error, guest denied, no secrets in logs/DTOs |

Salesforce automation is not a substitute for test assertions. Use `HttpCalloutMock` to simulate gateway endpoints, `System.runAs` for access/ownership scenarios (plus real-site external-user manual verification), and `Test.startTest/stopTest` where governor-limit isolation matters. Because `System.runAs` does not automatically prove every CRUD/FLS rule, use the actual Experience site and effective permission audits to substantiate security.

## 5. Evidence and defects templates

### Test run entry

```text
Run ID: GV-TEST-YYYYMMDD-01
Date/time/timezone:
Org alias / Site URL:
Git commit hash / package version:
Tester role:
Test ID:
Synthetic fixture SKU/order (redacted):
Steps performed:
Expected:
Actual observed (mandatory):
Status: PASS | FAIL | BLOCKED
Evidence filename/link:
Bug ID (if any):
Retest result/date:
```

### Bug entry

```text
Bug ID: GV-BUG-001
Severity: Critical | High | Medium | Low
Module/Test ID:
Environment/commit:
Reproduction steps:
Expected vs actual:
Security/payment exposure? Yes/No
Root cause / fix commit:
Retest evidence / status:
```

## 6. Traceability summary

| Requirement | Tests | Release blocker? |
|---|---|---|
| FR-01–04 Public browse/catalogue | TC-01–08 | Yes |
| FR-05 Identity | TC-09–10 | Yes |
| FR-06 Cart | TC-11–12 | Yes |
| FR-07 Checkout auth + validation | TC-13–16 | Yes |
| FR-08 Order snapshots | TC-17–18 | Yes |
| FR-09 Gateway and secrets | TC-19–20 | Yes |
| FR-10 Verified payment | TC-21–24 | Yes |
| FR-11 Stock exactly once | TC-25–28 | Yes |
| FR-12 Order privacy | TC-29–30 | Yes |
| FR-13 Staff admin | TC-31–32 | Yes |
| FR-14 UX and logging | TC-33–34 | Yes |
| Complete shopping journey | TC-35 | Yes |
| P1 optional notification Flow | TC-36–38 if active | Conditional |

### Formal sign-off (intentionally blank)

- Test execution date: **Not yet scheduled/executed**
- P0 test pass count: **0 verified / 35 required**, not "0 passed" as a failure claim
- Critical/High open defects: **Unknown — no run executed**
- Apex coverage: **Not measured**
- Customer A/B privacy: **Not verified**
- Razorpay payment HMAC/captured response: **Not verified**
- Signed release commit/tag: **Not assigned**
- Release status: **NOT READY FOR CLAIM OF MVP COMPLETION**

---
[Requirements](01_PROJECT_OVERVIEW.md) · [Apex](06_APEX_DOCUMENTATION.md) · [Integration](08_INTEGRATION_DOCUMENTATION.md) · [Release](12_DEPLOYMENT_GUIDE.md)