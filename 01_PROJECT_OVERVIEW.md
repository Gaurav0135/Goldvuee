# 01 — GoldVuee Project Overview and Business Requirements

**Document control:** GV-DOC-01 | Version 1.0 | Architecture baseline: 2026-11-04 | Status: **DESIGN — NOT IMPLEMENTATION EVIDENCE**  
**Authority:** Architecture decisions, names and cross-document conventions are controlled by [02_SYSTEM_ARCHITECTURE.md](02_SYSTEM_ARCHITECTURE.md); [03_DATA_MODEL.md](03_DATA_MODEL.md) is the detailed field dictionary.  
**Source:** The two project briefs supplied by the project owner; current-org facts are **owner-reported** and not independently inspected.

## 1. Executive statement — what exactly are we building?

GoldVuee is a Salesforce-native, mobile-responsive jewellery storefront with a public Men / Women / Kids catalogue and a signed-in B2C purchase journey. Customers can browse products without login, register or sign in using Experience Cloud, build a cart, provide an Indian shipping address, pay through **Razorpay test mode**, and see a verified order in their history. Employees manage products and quantities in Salesforce Lightning. The MVP proves a complete, secure journey rather than a wide catalogue of unrelated Salesforce automations.

**Seven-day definition of done:** an anonymous visitor browses a product, signs in, creates a cart, places a pending order, completes a Razorpay **test** payment, receives server-verified confirmation, sees inventory reduced **exactly once**, and can access only their own order. Negative tests must pass for data leakage, failed payments, duplicate callbacks and low stock. This is a demonstration/portfolio MVP, **not an authorization to conduct live financial transactions**.

## 2. Provenance and baseline

| Item | Status | Evidence to collect / action |
|---|---|---|
| Developer Edition org, My Domain, CLI connection | Reported configured | `sf org display`, org settings screenshots |
| Experience Cloud enabled, GoldVuee LWR site created and published | Reported configured | Site configuration, published revision |
| Public site opens in private/incognito window | Reported observed | New dated test evidence, no cached Salesforce login |
| GoldVuee Customers Account, cloned GoldVuee Customer profile, GoldVuee guest profile | Reported configured | Verify Account, Profile, Site Membership and external license |
| Self-registration enabled with default registration page | Reported configured, **not tested** | Test Contact + User creation, login, forgot password, duplicate email |
| Custom objects, product records, Apex, Flows, LWCs, Razorpay, data access | **Not verified/implemented** | Build; collect deployment and test results |

Existing site (owner-provided): https://goldvuee-dev-ed.develop.my.site.com/goldvuee. **Do not recreate the org or site.** No repository, org metadata, key, build log or Salesforce execution result was available for independent inspection while authoring this design.

## 3. Audiences, roles and journeys

| Audience | Capabilities in MVP | Post-MVP |
|---|---|---|
| Guest | Browse active public catalogue; browse collections, category, product, search; hold local non-authoritative cart | Public content, personalization |
| B2C customer | Standard Experience Cloud registration/login; cart; checkout; payment; own orders | Persistent server cart, wishlist, profile/address book, cancellation |
| Jewellery Manager | Manage categories, product details, publication, merchandising imagery | Campaigns and promotions |
| Inventory Manager | Manage a single stock record per SKU; read orders | Warehouse locations, stock movements, alerts |
| Order Manager | View and progress orders; investigate payments | Returns/refunds, fulfillment notifications |
| System Administrator | Security, integration credentials, metadata, monitoring | Production operations |
| B2B customer | No separate commerce behavior within seven days; business intent is documented | Business onboarding/approval, GSTIN, MOQ, tiered prices, wholesale orders |

**Account policy:** the existing shared `GoldVuee Customers` Account is a legacy setup baseline, not proof of safe contact sharing. B2C self-registration must link each user to a unique Contact. Private Contact sharing and server-side Contact-ID checks are mandatory; if shared-Account membership leaks peers' records, change onboarding/account association before release rather than loosening sharing.

## 4. Priority language and scope boundary

- **P0 / Day 1–7:** website and categories; Product2 catalogue and single-location inventory; customer identity and isolation; client cart; custom pending order/order lines; price/stock revalidation; Razorpay test transaction with server-authoritative verification; exactly-once inventory deduction; own order history; basic Lightning admin; tests/deployment.
- **P1 / only after P0 passes:** product search/sorting refinements, category imagery, standard profile settings, optional order-confirmed email Flow, desktop/mobile polish, basic sales and low-stock reports. Product images and basic search are P0; enhancements are P1.
- **P2 / after MVP:** wishlist (`GV_Wishlist_Item__c`), durable cross-device cart, business registration and approval, B2B price books/MOQ, advanced inventory ledger/reservation, cancellation/refunds/returns, configurable shipping/taxes, webhooks and reconciliation, more Flow automations, detailed dashboards.
- **P3 / future:** loyalty/reviews/coupons, multi-warehouse, advanced analytics, promotions, recommendations, custom domain, forecasting, sophisticated order orchestration.

**Explicit MVP exclusions:** no live Razorpay payments; no B2B differential checkout; no multi-currency; no multi-warehouse; no public guest record creation; no post-payment automatic refund; no advanced sales-tax engine; no over-engineered trigger framework. These exclusions must be visible in the demo.

## 5. Functional requirements and test traceability

| ID | Functionality and acceptance criteria | Owner Salesforce component | Evidence |
|---|---|---|---|
| FR-01 | Guest can open site in new incognito session, no authentication | Experience Cloud LWR + page configuration | TC-01, TC-02 |
| FR-02 | Men/Women/Kids and 5 categories each come from Salesforce data | `GV_Category__c`; `GV_CatalogueController`; `goldVueeHeader`, `goldVueeCatalogue` | TC-03, TC-04 |
| FR-03 | Only active/published products show to public; image, description and correct price displayed | `Product2`, `PricebookEntry`, CMS; catalogue Apex + LWCs | TC-05, TC-06, TC-07 |
| FR-04 | Search and category filter return only authorized matches | `GV_CatalogueController.getProducts` | TC-08 |
| FR-05 | Self-registration/login/logout works; each user maps to own Contact | Standard Experience Cloud Login & Registration | TC-09, TC-10 |
| FR-06 | Quantity add/update/remove, subtotal; no trusted client price | `goldVueeCart` + `GV_CheckoutService` | TC-11, TC-12 |
| FR-07 | Checkout rejects anonymous and malformed/insufficient-stock orders | `goldVueeCheckout`, `GV_CheckoutController` | TC-13–TC-16 |
| FR-08 | Pending order/lines hold immutable unit-price/address snapshots | `GV_Order__c`, `GV_Order_Item__c`, `GV_CheckoutService` | TC-17, TC-18 |
| FR-09 | Gateway attempt starts in Razorpay test mode; no key secret exposed | `GV_PaymentController`, `GV_RazorpayClient`, Named Credential | TC-19, TC-20 |
| FR-10 | Frontend response cannot itself confirm payment; server verifies gateway status/order/amount and signature where configured | `GV_PaymentService` | TC-21–TC-24 |
| FR-11 | Captured confirmed payment deducts stock exactly once; concurrent low stock handled safely | `GV_CheckoutService` row locks + `GV_Payment__c` unique IDs | TC-25–TC-28 |
| FR-12 | Customer sees their orders but cannot read another customer's orders | `GV_OrderController`, security / sharing model | TC-29, TC-30 |
| FR-13 | Assigned staff manage catalogue, inventory and order status; non-staff cannot | Standard Lightning pages, permission sets | TC-31, TC-32 |
| FR-14 | Site renders usable layout on mobile/desktop; no customer PII in logs | LWC + CSS + security conventions | TC-33, TC-34 |

All tests are **planned / not run** until supported by recorded execution evidence in [11_TESTING_DOCUMENTATION.md](11_TESTING_DOCUMENTATION.md).

## 6. Key business decisions (explicit assumptions)

1. **Single-currency test market: INR / India.** Display amount in rupees; gateway accepts integer paise. INR and Indian delivery format are assumptions to approve before Day 5.
2. **One price per SKU for MVP.** Standard `Pricebook2` + `PricebookEntry` hold authoritative catalogue prices. A B2B price book is future scope.
3. **One inventory row per Product2.** No reservation at checkout; customer is not promised stock until verified payment is reconciled. See stock-shortage handling in [02](02_SYSTEM_ARCHITECTURE.md) and [08](08_INTEGRATION_DOCUMENTATION.md).
4. **No invented tax law.** Demo defaults to `GV_Tax__c = 0` and `GV_Shipping__c = 0`, clearly shown at checkout; use fixed, signed-off test charges only. Real tax/shipping calculations require commercial/legal approval and are outside MVP.
5. **Existing website/auth remains.** Use native registration/login where the Developer Edition supports the chosen external-user license; account/contact and password setup must be tested.
6. **No confirmed payment without captured gateway state.** An authorized-only or failed payment is not an order confirmation.
7. **Successful payments with late stock shortage go to `Payment Review`, not silently fulfilled.** Manual resolution in test mode, with future reservation/refund handling planned.

## 7. Non-functional requirements

| Dimension | MVP acceptance target / limit |
|---|---|
| Security | Default-deny, guest read only public fields, no PII/secret leakage, order belongs to current Contact |
| Reliability | Two identical payment verifications create one captured state and one deduction; safe retry on gateway timeout |
| Performance | Catalogue capped/paginated (maximum 24 products per request); measured on sample data, no unverified SLA claims |
| Responsiveness | Manual checks at ~375px and ~1440px plus keyboard navigation |
| Accessibility | Semantic headings, labelled inputs, image alt text, visible focus, contrast review, usable errors |
| Maintainability | Same API names across documents; all pricing/payment invariants centralized in Apex |
| Testing | Apex tests plus LWC/manual browser, incognito, customer A/B isolation and concurrency; all release gates green |
| Capacity | Developer Edition limits, available Experience Cloud licenses, CMS access, email quotas and API callout availability must be verified |

## 8. Seven-day success criteria

The project owner demonstrates one continuous trace: incognito `Home → Women → Rings → product detail → login/registration → cart(quantity 2) → shipping → Razorpay test capture → order Confirmed → inventory decreased by 2 → My Orders`. Then demonstrate a failed payment, a replayed callback that does not deduct twice, a stock-conflict case and a second customer who cannot view the first user's order. Record the run in TC-35 and attach screenshots, record IDs (redacted), deployment commit, and result. See [10_IMPLEMENTATION_ROADMAP.md](10_IMPLEMENTATION_ROADMAP.md).

## 9. Post-MVP business design

B2B uses a separate business Account with B2B Contacts/Users, a vetted business-approval state and its own `Pricebook2` (or pricing strategy when justified). MOQ validation belongs to checkout services, not browser code. VAT/GSTIN, purchase terms, bulk quotes, fulfillment and credits are future design, never implied as completed. Future customer wishlists have one `GV_Wishlist_Item__c` per Contact/Product with a unique composite key; persistent carts will require separately versioned object/field designs and migration tests. No need to build 20 Flows.

---
**Navigation:** [Architecture](02_SYSTEM_ARCHITECTURE.md) · [Data model](03_DATA_MODEL.md) · [Roadmap](10_IMPLEMENTATION_ROADMAP.md) · [Tests](11_TESTING_DOCUMENTATION.md)