# 06 — Apex Architecture, Classes, Methods and Trigger Policy

**GV-DOC-06 | v1.0 | Planned code contracts; no Apex classes exist as verified artifacts.** Names and behavior must match [02_SYSTEM_ARCHITECTURE.md](02_SYSTEM_ARCHITECTURE.md). Do not add a generic framework unless code duplication justifies it.

## 1. Design standards

- Controllers: `public with sharing` (unless platform constraints require explicit equivalent) and `@AuraEnabled`; no public mutable sObject responses; allowlisted DTOs only.
- Services: single business rule owner. Narrow, audited `SYSTEM_MODE` writes only after authenticated principal, current Contact ownership and validation. Explicit user mode for normal public catalogue reads. No customer-wide CRUD grant as a shortcut.
- Keep SOQL/DML out of loops; map products, stock and lines before insert/update; cap user input and page sizes; `FOR UPDATE` on Order and Inventory during confirmation.
- No exception containing PII, API keys or raw gateway payload returned to LWC or `System.debug`.
- Class/API versions pinned in metadata and tested; Salesforce API 67+ behavior differs in user/system default modes, so document explicit `WITH USER_MODE`, `WITH SYSTEM_MODE` or Database access levels and get security review.
- Domain validates quantities, prices, order status, gateway identifiers, amount/currency, duplicate attempts, stock and ownership. UI validations are only for fast feedback.
- Callouts before DML in their method transaction (`startPayment`); payment verification fetches provider state first, then calls transaction-only confirmation method.

## 2. Exact class inventory, methods, tests

### `GV_CatalogueController` — P0

| Method | Parameters | Returns | Responsibilities |
|---|---|---|---|
| `@AuraEnabled(cacheable=true) getCategories` | `String audience` | `List<CategoryDTO>` | Valid `Men/ Women/ Kids`, active categories, fields whitelist |
| `@AuraEnabled(cacheable=true) getProducts` | `String audience, Id categoryId, String searchText, String sortBy, Integer pageNumber` | `List<ProductCardDTO>` | Published active products with active category and Retail price; search, allowlisted sorting, page size <=24 |
| `@AuraEnabled(cacheable=true) getProduct` | `Id productId` | `ProductDetailDTO` | One published active product, price, public image key, derived in-stock boolean |

**Objects:** `GV_Category__c`, `Product2`, `PricebookEntry`, `Pricebook2`; private stock boolean derived through trusted narrow service query without returning inventory fields (guest data exposure reviewed). **LWCs:** `goldVueeHeader`, `goldVueeHome`, `goldVueeCatalogue`, `goldVueeProductDetail`. **Security:** read with explicit user mode where access supported; no exposing unpublished, inactive, missing-price items. `sortBy` whitelist `NameAsc`, `PriceAsc`, `PriceDesc`, `Newest`; canonical date uses `CreatedDate`. **Errors:** invalid input returns safe validation; unknown product returns generic not found. **Tests:** `GV_CatalogueControllerTest`, TC-03–08.

### `GV_CheckoutController` — P0

| Method | Parameters | Returns | Responsibilities |
|---|---|---|---|
| `@AuraEnabled createPendingOrder` | `String checkoutJson` | `PendingOrderDTO` | Resolve `User.ContactId`, parse bounded cart/address input, invoke domain, return Order Id/number/canonical total |

**Objects:** indirectly `Contact`, `Product2`, Retail pricebook, Inventory, Order, Order Item. **LWC:** `goldVueeCheckout`. **Constraints:** login required; `checkoutJson` schema allowlist: `items:[{productId,quantity}]`, `shipping:{recipientName,phone,addressLine1,city,state,postalCode,country}`; no `contactId`, `unitPrice`, `tax`, `discount`, `paymentStatus` accepted. Each line quantity integer 1–10 for demo; 1–20 unique lines; reject duplicate Product Id or consolidate server side deterministically (choose reject for this MVP). **Tests:** `GV_CheckoutServiceTest`, TC-11–18.

### `GV_PaymentController` — P0

| Method | Parameters | Returns | Responsibilities |
|---|---|---|---|
| `@AuraEnabled startPayment` | `Id orderId` | `PaymentStartDTO` | Authenticated owner, order Pending Payment, create fresh gateway order + local `GV_Payment__c` attempt, return public test key ID and gateway order ID |
| `@AuraEnabled verifyPayment` | `Id orderId, Id paymentAttemptId, String razorpayPaymentId, String razorpaySignature` | `PaymentResultDTO` | Ownership validation, provider fetch, HMAC/secret gate, captured result, invoke exactly-once domain confirmation |

**Objects:** `GV_Order__c`, `GV_Payment__c`, Inventory indirectly. **LWC:** `goldVueeCheckout`, `goldVueeOrderConfirmation`. **Security:** deny guest, verify `GV_Order__c.GV_Contact__c = User.ContactId` even if row is visible; no arbitrary amount or order id in server callout; never expose secret. **Tests:** `GV_PaymentServiceTest`, TC-19–28.

### `GV_OrderController` — P0

| Method | Parameters | Returns | Responsibilities |
|---|---|---|---|
| `@AuraEnabled(cacheable=true) getMyOrders` | none | `List<OrderSummaryDTO>` | Only orders for logged-in Contact, newest first; cap initial result to 50 |
| `@AuraEnabled(cacheable=true) getMyOrder` | `Id orderId` | `OrderDetailDTO` | Own order + lines; status, totals, sanitized payment result (not payment gateway secrets) |

**Objects:** `GV_Order__c`, `GV_Order_Item__c`. **LWC:** `goldVueeMyOrders`, `goldVueeOrderConfirmation`. **Security:** no customer ID parameter, reject guest and cross-customer Ids. Use guarded privileged read if user mode lacks direct access to private Order; do not remove Contact filter. **Tests:** `GV_OrderControllerTest`, TC-29–30.

### `GV_CheckoutService` — P0 domain and transactional truth

| Method | Parameters | Returns | Core operations |
|---|---|---|---|
| `createPendingOrder` | `Id contactId, List<CheckoutLineDTO> lines, AddressDTO address` | `PendingOrderDTO` | Validate identity, active product and Retail prices, stock check, calculate totals, insert Order/Order Items in one transaction |
| `confirmCapturedPayment` | `Id orderId, Id paymentAttemptId, GatewayPaymentDTO payment` | `PaymentResultDTO` | Lock order and inventory; validate amount/currency, provider IDs/status, replay and stock; update Payment, stock, Order atomically |

**Order creation validation sequence:** verify Contact belongs to logged-in User; reject duplicate/invalid line entries; query all SKU/pricebook entries once; query all inventory records once; check published/active + Category; compute subtotal in Decimal; validate Indian address and six-digit postal code; insert Order first, then all Items (bulk DML). No initial stock deduction and no payment acceptance at this step. **Return** canonical total from server; client must display changed-price warning if different from local estimate.

**Capture transaction:** lock `GV_Order__c FOR UPDATE`, lock all related `GV_Inventory__c` rows in deterministic Product Id order, inspect `GV_Payment__c` attempt and captured provider proof, skip changes if already Confirmed with same captured payment, reject if previously confirmed by another payment, validate quantities/stock again; set Payment Captured, Order Confirmed, `GV_Confirmed_At__c`, decrement `GV_On_Hand__c`. If stock insufficient after valid capture, set `GV_Payment__c.GV_Status__c=Captured` and `GV_Order__c.GV_Status__c=Payment Review` without decrement (preserve the fact that funds were captured); manual test-mode staff reconciliation. Never use `Savepoint`/rollback in a way that loses already-verified provider info; document reconciliation instructions.

**Critical idempotency:** unique `GV_Razorpay_Payment_Id__c`; guard Order state under lock; idempotent replay should return `alreadyConfirmed=true` and not decrement. **Tests:** `GV_CheckoutServiceTest` and `GV_PaymentServiceTest`, TC-17, TC-25–28.

### `GV_PaymentService` — P0 orchestration

| Method | Parameters | Returns | Responsibilities |
|---|---|---|---|
| `startPayment` | `Id contactId, Id orderId` | `PaymentStartDTO` | Fetch own pending order and canonical amount, Razorpay API call, persist unique payment attempt with `Created` state |
| `verifyPayment` | `Id contactId, Id orderId, Id paymentAttemptId, String razorpayPaymentId, String signature` | `PaymentResultDTO` | Fetch own attempt, signed callback verification, fetch gateway payment, validate captured status/order/INR/paise, delegate transactional confirmation |

**Avoid DML-before-callout:** `startPayment` only reads (and validates) prior to callout, then inserts attempt. If insert fails after a successful gateway callout, log an admin-safe reconciliation reference and fail closed (never confirm based only on callback). **Signature design:** HMACSHA256 over persisted `GV_Razorpay_Order_Id__c + '|' + razorpayPaymentId` with a server-held secret, constant-time comparison where feasible, followed by Razorpay fetch. See [08_INTEGRATION_DOCUMENTATION.md](08_INTEGRATION_DOCUMENTATION.md) for the **important Named Credential secret accessibility limitation** and test-only boundary. A captured fetched payment without signature must not be treated as satisfying the specified signed-callback implementation; if the HMAC secret gate is not resolved, mark Day 6 integration incomplete, do not ship payment success.

**Tests:** `GV_PaymentServiceTest`, `HttpCalloutMock`, TC-19–28.

### `GV_RazorpayClient` — P0 integration adapter

| Method | Parameters | Returns | Endpoint |
|---|---|---|---|
| `createOrder` | `Long amountPaise, String receipt` | `GatewayOrderDTO` | `POST callout:GV_Razorpay_Test/v1/orders` |
| `fetchPayment` | `String razorpayPaymentId` | `GatewayPaymentDTO` | `GET callout:GV_Razorpay_Test/v1/payments/{id}` |

**Validation:** positive integer amount; INR; URL-encode/validate provider ID format, finite timeouts, approved HTTP status codes, response schema, no raw response logs. Basic authentication done by `GV_Razorpay_Test_Auth` External Credential and Named Credential (no secrets in Apex). **Tests:** `GV_RazorpayClientTest` with `HttpCalloutMock`, TC-19–24.

## 3. DTO contracts (public vs internal)

| DTO | Fields allowed (representative fixed schema; no arbitrary sObjects) |
|---|---|
| `CategoryDTO` | `id`, `name`, `slug`, `audience`, `sortOrder` |
| `ProductCardDTO` | `id`, `sku`, `name`, `audience`, `categoryId`, `unitPrice`, `imageKey`, `inStock` |
| `ProductDetailDTO` | ProductCard fields + `description`, `material`, `weightGrams` |
| `CheckoutLineDTO` | **Input:** `productId`, `quantity` only |
| `AddressDTO` | **Input:** `recipientName`, `phone`, `addressLine1`, `city`, `state`, `postalCode`, `country` |
| `PendingOrderDTO` | `orderId`, `orderNumber`, `subtotal`, `tax`, `shipping`, `total`, `currency`, `status` |
| `PaymentStartDTO` | `paymentAttemptId`, `razorpayOrderId`, `razorpayKeyId` (public test key), `amountPaise`, `currency` |
| `GatewayOrderDTO` | `id`, `amountPaise`, `currency`, `status` (trusted only after authenticated callout) |
| `GatewayPaymentDTO` | `id`, `razorpayOrderId`, `amountPaise`, `currency`, `status` |
| `PaymentResultDTO` | `orderId`, `orderStatus`, `alreadyConfirmed`, `message` |
| `OrderSummaryDTO` | `id`, `orderNumber`, `status`, `total`, `createdDate` |
| `OrderDetailDTO` | OrderSummary + shipping summary + `lines:[{sku,name,quantity,unitPrice,lineTotal}]`; **no** raw Payment record |

Use public DTO-only return values; never serialize custom fields from `User`, `Contact` or `GV_Payment__c` wholesale. DTOs can be nested classes within their owner controller/service to avoid a forest of classes. The method *names and parameters* in Section 2 are fixed; implementation-language differences between DTO class definitions and JSON serialization can be settled within these contracts.

## 4. Error codes and response handling

| Code | When | Expected UI action |
|---|---|---|
| `AUTH_REQUIRED` | Guest calls private operation | Redirect native login |
| `FORBIDDEN` | Other Contact's Order/attempt | Generic unavailable, no record details |
| `PRODUCT_UNAVAILABLE` | Draft/inactive/no price | Remove item or return catalogue |
| `INSUFFICIENT_STOCK` | Requested qty greater than live stock | Ask customer to change quantity |
| `PRICE_CHANGED` | Server Retail price differs from shown estimate | Refresh total; require reconfirmation |
| `PAYMENT_NOT_CAPTURED` | Failed/authorized-only/fake callback | Keep pending; show retry guidance |
| `PAYMENT_VERIFICATION_FAILED` | Invalid HMAC/amount/order/currency | Fail closed; log redacted reference |
| `PAYMENT_REVIEW` | Captured but stock unavailable | Display safe manual-review state; no fulfillment |
| `GATEWAY_UNAVAILABLE` | Callout timeout/non-2xx | Retry with fresh attempt where appropriate; do not confirm |

No customer-facing stack traces or internal SOQL error details. Do not assert an error has been handled unless a negative test verifies it.

## 5. Triggers and governor-limit discipline

**MVP trigger inventory: NONE.** Trigger handlers may be introduced P2 for non-UI stock adjustments or enterprise integrations. Do not put gateway verification or payment capture in record-triggered automation. Use `SELECT ... FOR UPDATE` in the confirm service, bounded lock wait/retry policy, deterministic ordering and no DML/SOQL in item loops. Confirm with two simultaneous test transactions (TC-28) and annotate governor-limit behavior under load; `FOR UPDATE` makes races manageable but does not replace uniqueness and state guards.

**Code layout** (metadata is created during implementation, not provided here):

```text
force-app/main/default/classes/
  GV_CatalogueController.cls
  GV_CheckoutController.cls
  GV_PaymentController.cls
  GV_OrderController.cls
  GV_CheckoutService.cls
  GV_PaymentService.cls
  GV_RazorpayClient.cls
  GV_CatalogueControllerTest.cls
  GV_CheckoutServiceTest.cls
  GV_PaymentServiceTest.cls
  GV_OrderControllerTest.cls
  GV_RazorpayClientTest.cls
force-app/main/default/triggers/   # empty in 7-day MVP
```

## 6. Apex test catalog

| Test class | Required cases |
|---|---|
| `GV_CatalogueControllerTest` | Guest/public filter, category audience, active price, search injection, pagination, unpublished exclusion |
| `GV_CheckoutServiceTest` | Valid pending order totals/snapshots, 0/negative/large qty, duplicate lines, missing stock/price, price change, status rules, lock conflict |
| `GV_PaymentServiceTest` | Success + HMAC, forged signature, wrong order ID, wrong amount/currency, authorized-only, replay captured callback, concurrent inventory, captured insufficient stock |
| `GV_RazorpayClientTest` | Authenticated mocked order/create/fetch, non-2xx, timeout, malformed payload, no logging of secrets |
| `GV_OrderControllerTest` | Own history, Customer A/B isolation, guest denial, missing Id |

**Coverage target:** >= 85% overall Apex line coverage for project-owned classes as a quality target, 100% of named security/payment/stock business branches exercised; Salesforce deployment minimum coverage rules still apply. Passing coverage alone is insufficient. Never claim any test passed until run in the target org and results recorded.

---
[Architecture](02_SYSTEM_ARCHITECTURE.md) · [Flows](05_FLOW_DOCUMENTATION.md) · [Payment](08_INTEGRATION_DOCUMENTATION.md) · [Tests](11_TESTING_DOCUMENTATION.md)