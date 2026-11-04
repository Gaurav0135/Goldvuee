# 10 — GoldVuee Implementation Roadmap: 7-Day MVP vs Post-MVP

**GV-DOC-10 | v1.0 | Plan, not elapsed work.** Day numbers are a **relative seven-day sprint**, not assertions that work was already performed. Do not mark a phase complete until its acceptance tests pass, evidence is saved, and changes are committed.

## 1. Delivery method and release gates

**Track A** is exactly seven development days culminating in a secure, demonstrable test-only purchase. **Track B** begins only after the P0 sign-off, with later enterprise capabilities. Any item whose inclusion threatens the end-to-end journey is deferred, **unless deferral would break P0 security or payment correctness**.

MVP goal: existing public site → dynamic catalogue → native login → local cart → pending order → Razorpay **test** payment → signature/provider verification → confirmed order and exactly-once stock update → own order history → negative tests. All functionality maps to FR-01–14 and TC-01–35 in Docs [01](01_PROJECT_OVERVIEW.md) and [11](11_TESTING_DOCUMENTATION.md).

**Prerequisites and hard gates:** Salesforce external-user license and registration must work; correct guest security must be possible; CMS or approved public image strategy must be confirmed; Razorpay test credentials and an approved HMAC secret strategy must exist by Day 6; the configured site must support CSP-compliant checkout. If any fail, record the limitation, **do not mark payment success or MVP complete**, and show partial functionality with clear blockers.

## 2. Track A — seven-day plan with test and Git milestone

### Day 1 — Org audit, architecture freeze, identity/data foundation

**Objective:** establish deployable minimum schema without rebuilding the org.  
**Prerequisites:** CLI auth already reported configured; access to Setup, existing site and external-user-license allocation.

**Tasks:**
1. Capture `sf org display`, list org metadata, site details, existing Profiles/Accounts/Sharing, available External User license, CMS workspace and permission constraints. Classify each reported baseline item as verified/not verified.
2. Freeze Docs 02 and 03 names. Evaluate existing Standard Order vs custom `GV_Order__c` and record ADR-03 acceptance.
3. Create `GV_Category__c`, `GV_Inventory__c`, `GV_Order__c`, `GV_Order_Item__c`, `GV_Payment__c` with **P0 fields only** as defined in Doc 03. Extend `Product2` with GoldVuee attributes. Configure `GoldVuee Retail` Pricebook2 (including Standard Pricebook prerequisites).
4. Add minimum validation rules, labels, app tabs and proper relationships; check unique External IDs on category slug/SKU/inventory key/gateway IDs.
5. Preserve, inspect and test baseline Contact/User self-registration setup; never replace site/profile blindly.

**Gate:** metadata deploy succeeds; sample schema can persist/relate a category/Product2/PricebookEntry/Inventory; no duplicate standard/custom product/price model. **Tests:** TC-09 registration smoke (or formally blocked), TC-16 validation schema, TC-31 staff access preliminary. **Output:** org audit and field dictionary frozen. **Commit:** `docs(architecture): freeze GoldVuee P0 contracts`, `feat(schema): add MVP objects and validation`.

### Day 2 — Catalogue, CMS media, retail prices and stock

**Objective:** staff can create and maintain credible sample products.  
**Depends on:** Day 1 objects, required profile/permissions, Pricebook entry prerequisites.

**Tasks:**
1. Seed 15 `GV_Category__c` records (five each for Men/Women/Kids) with unique slugs.
2. Add 8–15 sample jewellery Product2 records with unique `GV_SKU__c`, matching audience/category, description/specs and published status.
3. Populate Retail PricebookEntry positive INR prices; ensure active Standard Pricebook setup where required.
4. Configure Salesforce CMS public workspace/channel and test LWR public image retrieval; if not available, adopt approved public site asset fallback **with explicit tracking** and accurate Product2 image keys.
5. Create exactly one `GV_Inventory__c` per SKU, `GV_Product_Key__c` equals Product2 18-character Id; verify no duplicates/negative values.
6. Create minimum Lightning admin layouts/list views for Product2/Category/Inventory and staff permissions.

**Gate:** products and photos are visible in secure staging preview, price correct, inactive product absent from planned public DTO, inventory has no duplicate rows. **Tests:** TC-03–07, TC-31. **Commit:** `feat(catalogue): seed categories prices images stock`.

### Day 3 — Public LWR pages and read-only catalogue

**Objective:** incognito customer can browse whole basic catalogue.  
**Depends on:** Day 2 data, guest read permission/sharing proof.

**Tasks:**
1. Implement `GV_CatalogueController`: `getCategories`, `getProducts`, `getProduct` with DTO allowlist, active/published filters, name/price sort, safe search and pagination.
2. Build and expose `goldVueeHeader`, `goldVueeHome`, `goldVueeCatalogue`, `goldVueeProductCard`, `goldVueeProductDetail` in existing Experience Builder LWR site.
3. Configure public page routes and working image resolution; test Men/Women/Kids and all categories from Salesforce, not hardcoded in JS.
4. Audit guest profile class access, Category guest-sharing criteria and Product2/Pricebook access supported in org. Verify no private objects accessible.
5. Publish site and test in a clean incognito window, mobile width and console/network tab.

**Gate:** FR-01–04 pass anonymously; any unpublished/private record denied. **Tests:** TC-01–08, TC-30, TC-33 partial. **Commit:** `feat(site): launch secure public catalogue`.

### Day 4 — Identity, account isolation, client cart

**Objective:** a distinct Contact can sign in and hold a safe cart.  
**Depends on:** guest site functional, approved external-user license/profile.

**Tasks:**
1. Verify registration/login/logout/forgot password through existing Experience Cloud standard pages; confirm new User maps to unique Contact under correct Account strategy.
2. Make two isolated test customers; test A cannot see B's Contact or Account data (especially if sharing `GoldVuee Customers` Account).
3. Build `goldVueeCart` using only `gv_cart_v1` `{productId,quantity}`; add/update/remove from catalogue/detail using component events; persist across same-browser refresh.
4. Guard checkout route; preserve cart through login. Validate local data with size limits, corrupt data rejection and no stored address/price/payment secret.
5. Configure `GV_Storefront_Customer` permission set and Apex class access as supported.

**Gate:** A/B identity isolation confirmed; cart updates, quantity and estimate work without granting guest DML. **Tests:** TC-09–14, TC-29/30 preliminary, TC-34 partial. **Commit:** `feat(customer): verify native auth and local cart`.

### Day 5 — Server checkout, order records and history

**Objective:** authenticated buyer can create a safe **Pending Payment** order.  
**Depends on:** Day 4 identity and cart; Day 1 price/stock schema.

**Tasks:**
1. Build `GV_CheckoutController`, `GV_CheckoutService.createPendingOrder` with server-derived ContactId, published product + Retail Pricebook checks, stock check, 1–10 quantity limit, 20 unique SKU max, address validation.
2. Calculate INR totals on server; use demo zero tax and shipping only if explicitly accepted. Save pending order and order item snapshots; handle price changed vs client estimate.
3. Implement `GV_OrderController.getMyOrders/getMyOrder` filtered by authenticated Contact and restricted read architecture.
4. Build `goldVueeCheckout`, `goldVueeMyOrders`, `goldVueeOrderConfirmation` basic states. Order confirmation page says **Pending Payment** until payment integration complete.
5. Test insufficient stock, invalid order owner and cross-customer ID enumeration.

**Gate:** order/lines and owner isolation are correct, but nothing is called paid. **Tests:** TC-13–18, TC-29–30. **Commit:** `feat(checkout): create secure pending orders and history`.

### Day 6 — Razorpay test payment and exactly-once capture

**Objective:** complete the test-only purchase transaction.  
**Depends on:** valid test account, approved HMAC secret accessibility/access controls, Named Credential, LWR checkout script/CSP, Day 5 server order lifecycle.

**Tasks:**
1. Configure `GV_Razorpay_Test_Auth` External Credential + `GV_Razorpay_Test` Named Credential; load test credentials **outside Git**, grant only trusted principal access.
2. Implement `GV_RazorpayClient.createOrder/fetchPayment` with `HttpCalloutMock` tests; validate response and no logs of credentials.
3. Implement `GV_PaymentController` and `GV_PaymentService.startPayment/verifyPayment`. Call gateway before local payment-attempt DML, validate persisted gateway order ID, callback HMAC, fetched `captured` amount/order/currency.
4. Implement `GV_CheckoutService.confirmCapturedPayment` with lock/idempotency, unique payment IDs, confirmed status, stock decrement in same transaction, safe `Payment Review` state on captured-but-shortage.
5. Launch Razorpay Standard Checkout in CSP-compliant LWR page; do not set `Confirmed` from JavaScript.
6. Add optional `GV_Order_Confirmed_Notification` P1 **only if** core tests pass and deliverability is verified; no Flow needed for stock/payment.

**Hard gate:** signature secret must not be exposed, HMAC/server captured verification passes, duplicate callback decrements once, no-stock case enters Review, no guest/customer cross-access. If unresolved, document **BLOCKED** and do not falsely mark payment complete. **Tests:** TC-19–28, TC-36–38 if Flow enabled. **Commit:** `feat(payment): verify Razorpay test captures exactly once`.

### Day 7 — Full regression, security review, documentation and demo deployment

**Objective:** stable portfolio demonstration with reproducible evidence.  
**Depends on:** gates Days 1–6 all green.

**Tasks:**
1. Run Salesforce Apex suites and manual LWR/Chrome/phone test matrix. Record org id alias, commit hash, time, actual result and screenshot/test output for each TC.
2. Execute canonical end-to-end TC-35 in incognito and signed-in sessions, then replay payment callback, failed/authorized-only payment, concurrent stock case, A/B data isolation and mobile layout.
3. Run security scan of Git files/bundles/logs for credentials and private fields; verify guest permissions and sharing report.
4. Deploy metadata with Salesforce CLI to authorized target org; publish existing Experience site; recheck CSP, Named Credentials and guest setup per environment.
5. Update Docs 01–13, note defects/limitations, attach execution evidence, define follow-up post-MVP tasks and tag release only if P0 pass.

**Gate:** all P0 TC-01–35 pass; zero critical security or payment bugs; payment captures only test money; full Git repository reproducible; a single shopping journey runs with observable Order, Payment and Inventory records. **Commit:** `test(e2e): record GoldVuee MVP verification`, `docs(release): publish tested MVP`, tag `v0.1.0-mvp` **only after test success**.

## 3. Workboard / acceptance ledger

| Day | Key deliverable | Priority | Depends on | Current status | Test result | Documentation updated |
|---|---|---|---|---|---|---|
| 1 | Audit/schema/identity viability | P0 | Existing site/CLI | Not verified | Not run | Draft blueprint |
| 2 | Seed data/price/image/inventory | P0 | D1 | Not started | Not run | Draft blueprint |
| 3 | Anonymous secure catalogue | P0 | D2 | Not started | Not run | Draft blueprint |
| 4 | Identity isolation/cart | P0 | D3 | Not started | Not run | Draft blueprint |
| 5 | Pending order/own history | P0 | D4 | Not started | Not run | Draft blueprint |
| 6 | Signed test payment/stock exactly once | P0 | D5, secrets/CSP | Not started | Not run | Draft blueprint |
| 7 | Security regression/e2e/release | P0 | D1–6 | Not started | Not run | Draft blueprint |

**Status vocabulary:** `Not Started`, `In Progress`, `Blocked`, `Implemented — Untested`, `Test Failed`, `Verified`. Only `Verified` counts toward done. Evidence belongs in [Doc 11](11_TESTING_DOCUMENTATION.md); daily notes belong in [Doc 13](13_DEVELOPMENT_LOG.md).

## 4. Track B — post-MVP backlog (do not mix into Day 1–7)

| Phase | Features | Value / complexity | Dependencies | Priority |
|---|---|---|---|---|
| B1: Salesforce admin mastery | Page Layouts, Lightning Record Pages/App Builder, List Views, page actions, Formula Fields, Record Types only if justified | High learning value / low-medium | Verified base metadata | P1/P2 |
| B2: Customer experience | Wishlist `GV_Wishlist_Item__c`, cross-device persisted cart, saved address, richer account area | Higher engagement / medium | Strong Contact sharing and identity | P2 |
| B3: B2B shopping | Business Accounts with `GV_B2B_Status__c`, GSTIN, approval, wholesale Pricebooks, MOQ/quote flows | Business channel / high | License, account separation, pricing rules | P2 |
| B4: Inventory control | Reservation/expiry, inventory movement ledger, low-stock alert Flow, returns, multi-warehouse | Prevent oversell / high | Current exactly-once capture; schema migration | P2/P3 |
| B5: Payments and compliance | Webhook HMAC, retry event dedupe, reconciliation, secure production secret management, refunds, tax/shipping engine | Reliability / high | P0 gateway integration and legal sign-off | P2 |
| B6: Analytics and scale | Reports/dashboards, advanced Flow notifications, batch jobs if limits justify, custom domain, marketing | Operational insight / medium-high | Clean order/payment history | P2/P3 |

### Explicit B2B vs B2C branch (future)

B2C: self-register → public Retail Pricebook → qty≥1 → immediate test-card checkout. B2B: company Account with GSTIN → manual approval → designated external business user → approved wholesale Pricebook / MOQ → quote/volume or invoice workflow. **Do not let a B2C customer choose a B2B pricebook in the browser.** Pricing selected by server-authorized customer/account classification. New tests required for account isolation, pricebook leakage and MOQ.

## 5. Decision and risk register

| Risk | Probability/impact | Owner | Mitigation / gate |
|---|---|---|---|
| Developer Edition site/customer license limitations | Unknown/high | Admin | Validate Day 1; keep evidence and workaround scope truthful |
| Existing shared Account leaks peer records | High impact | Security owner | Two-customer privacy test Day 4; fix association/sharing |
| Guest cannot query selected Product2/Pricebook | Medium/high | Security + developer | Inspect OWD/guest access and DTO design; do not widen private sharing |
| CMS public media/checkout.js blocked by LWR | Medium/high | UI developer | Prototype on Days 2/6; adjust only to supported secure integration |
| HMAC key storage cannot be protected adequately | High impact | Integration owner | Test-only restriction + production blocker; no fabricated completion |
| Captured payment but inventory already sold | Medium/high | Domain developer | Payment Review state; add reservation in B4 |
| Seven-day workload too large | High | Project owner | Protect P0 path, defer P1, seed small catalogue, no advanced Flows |

## 6. Final release acceptance

**Pass only if:** all P0 functionality works in the actual site, no unauthenticated private records are readable, two customers cannot share order history, gateway verifies amount/currency/capture/signature, duplicate callbacks are idempotent, stock never goes below zero, test data and evidence recorded, and deployment/recovery steps documented. A screenshot of a payment popup is **not proof** of the transaction.

---
[Project scope](01_PROJECT_OVERVIEW.md) · [Architecture](02_SYSTEM_ARCHITECTURE.md) · [Testing](11_TESTING_DOCUMENTATION.md) · [Log](13_DEVELOPMENT_LOG.md)