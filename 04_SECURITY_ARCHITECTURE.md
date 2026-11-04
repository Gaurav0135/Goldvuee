# 04 — GoldVuee Security Architecture

**GV-DOC-04 | v1.0 | Planned controls, not a security audit.** The existing published public Experience Cloud site is owner-reported. Guest permissions, data sharing, registration and external customer security are **unverified**; opening a page in incognito does not prove data isolation.

## 1. Design principles and threat model

- **Default deny.** Guest is for public catalogue read, no Contact/User/Account/Inventory/Order/Payment access. Do not enable `API Enabled`, `View All Data`, `Modify All Data`, or object-wide `View All` on guest.
- **Least privilege.** Use minimal profiles and assign permissions with permission sets. Native authentication, not a custom password store. Apply object CRUD, field-level security (FLS), record sharing, Apex class access and business ownership checks separately.
- **Server authority.** No client-provided `ContactId`, prices, stock counts, payment status, order status, amount, or discount can authorize a transaction.
- **No sensitive leakage.** Avoid email, phone, delivery address, credential, payment signature or raw payment payload in client URLs, browser storage, CMS, debug logs, error messages or GitHub.
- **Separate public read from privileged writes.** Public catalogue controller returns allowlisted DTO fields. Checkout and payment writes use narrowly scoped privileged operations with immutable server identity checks, and are not callable by guest.
- **Release blocker:** two-user security audit must prove a user cannot read another customer's Contact, Order, Order Item or Payment, including by guessing an Id or altering Apex arguments.

Key threats: a guest extracting unpublished inventory or private contacts through exposed Apex; two external users attached to the same Account seeing each other's contacts/orders; horizontal privilege escalation by supplying someone else's Order Id; client payment spoofing, replay and checkout price changes; CSP/XSS when loading Razorpay; secrets in metadata/repo; concurrent requests overselling.

## 2. Identity and access model

| User | Auth/license | Assignment | Core restrictions |
|---|---|---|---|
| System Administrator | Salesforce internal | Existing System Administrator profile | Setup, credentials, deploy; do not test customer security while logged in as admin |
| Jewellery Manager | Internal user/license as available | Minimal internal profile + `GV_Catalogue_Manager` | Maintain category, product, Retail Pricebook; no payment credentials |
| Inventory Manager | Internal | Minimal internal profile + `GV_Inventory_Manager` | Update Inventory, see SKU info; cannot confirm payment |
| Order Manager | Internal | Minimal internal profile + `GV_Order_Manager` | Read/manage orders, order items and sanitized payment status; no secret |
| B2C Customer | Experience Cloud external user; license must be verified | Existing GoldVuee Customer profile if suitable + `GV_Storefront_Customer` | Self profile and own order DTO; no direct inventory/payment object rights |
| B2B Customer | **P2** separate external role/profile/license strategy | `GV_B2B_Customer` P2 | Restricted to own approved Account and wholesale orders |
| Guest Visitor | GoldVuee site-specific guest profile | Guest profile; no blanket permission set | Public catalogue fields only; no write |

The owner reports existing `GoldVuee Customer` and guest `GoldVuee Profile`; **inspect actual API names and licenses before making changes**. Avoid cloning more profiles for each function. Assign staff permission sets; a Permission Set Group is optional P2 when staff role count grows.

## 3. Object CRUD / visibility matrix

Legend `R`=read; `C`=create; `U`=update; `D`=delete; `-`=none; `DTO`=only server-filtered fields, not direct object CRUD. Matrix is a target; actual metadata must be audited.

| Object | Guest | Customer | Jewellery Mgr | Inventory Mgr | Order Mgr | Admin |
|---|---|---|---|---|---|---|
| `Account` | - | Own minimal identity only if license requires | - | - | R if required | CRUD |
| `Contact` | - | Own profile only via standard settings | - | - | R as needed | CRUD |
| `User` | - | Self standard identity operations | - | - | - | CRUD |
| `Product2` | R approved public fields | R | RCU | R | R | CRUD |
| `Pricebook2` | Conditional R for catalogue pricing | Conditional R | RCU | R | R | CRUD |
| `PricebookEntry` | Conditional R active Retail price only | Conditional R | RCU | R | R | CRUD |
| `GV_Category__c` | R public active records | R | RCU | R | R | CRUD |
| `GV_Inventory__c` | - | - | R | RCU | R limited | CRUD |
| `GV_Order__c` | - | **DTO** own only | - | R if fulfillment needs | RU restricted status fields | CRUD |
| `GV_Order_Item__c` | - | **DTO** own parent only | - | R if required | R | CRUD |
| `GV_Payment__c` | - | - | - | - | R sanitized operational records | CRUD |
| `GV_Wishlist_Item__c` (P2) | - | Own CRUD (via secure service) | - | - | - | CRUD |
| `GV_Inventory_Movement__c` (P2) | - | - | - | RCU audited | R | CRUD |

**Important implementation nuance:** a customer-facing Apex service may require explicit **system-mode DML for protected Order/Payment/Inventory** while the customer has no direct object access. It MUST first derive `ContactId` from logged-in User, validate ownership and inputs, operate on allowlisted fields, and be narrowly reviewed/tested. An ordinary user-mode query of restricted objects would correctly fail; do not respond by giving customers broad CRUD on private Payment/Inventory data.

## 4. Organization-wide defaults, sharing and ownership

| Record type | Internal OWD target | External OWD target | Record ownership/strategy |
|---|---|---|---|
| Account/Contact | Private (where available/compatible) | Private | Contact customer access only through native self-user relation and explicitly evaluated sharing |
| `GV_Category__c` | Public Read Only if approved public fields only, otherwise Private | Private/guest criteria-read rules | Internal owner; publicly exposed active categories only |
| `Product2` + pricebook | Follow supported product/pricebook sharing model; restrict field/object access | Verify actual external object behavior | Show only `IsActive + GV_Published__c` and active price via allowlisted DTO |
| `GV_Inventory__c` | Private | Private | Inventory manager/staff controlled |
| `GV_Order__c` | Private | Private | Internal owner, explicit customer-only DTO guarded by ContactId; staff sharing |
| `GV_Order_Item__c` | Controlled by parent | Controlled by parent | Master-Detail inheritance |
| `GV_Payment__c` | Private | Private | Internal-only; no customer direct rights |

**Guest sharing:** Salesforce guest user rules are criteria-based, **read-only**, and make matching records public to anyone. Do not create a blanket guest sharing rule on Contact, Order, Payment or Inventory. For `GV_Category__c`, share only `GV_Active__c = true` if required for guest user-mode read. For products, confirm how `Product2` and pricebook objects expose guest records in this specific org and use supported guest criteria sharing if available; never infer that a Salesforce object supports an arbitrary guest rule. Require `GV_Published__c`, `IsActive`, category active and active price in Apex even if underlying sharing grants read. Use the Salesforce **Guest User Sharing Rule Access Report** during audit.

**External Customer access:** Avoid broad sharing set mappings by `AccountId` while all customers use the existing `GoldVuee Customers` Account; that may leak peers. Either use ContactId-scoped sharing for own Contact where supported, or let explicit server ownership checks expose order DTOs from restricted objects. For any deliberate elevated read of private orders, query by *both* `Id` and current `User.ContactId`, never by Id alone. The shared Account is a security **risk to verify**, not permission to share customers. Do not make internal staff sharing more permissive solely to fix storefront UI.

## 5. Field-level security and API protection

**Public Product2 fields allowlist:** `Id`, `Name`, `Description`, `GV_SKU__c`, `GV_Audience__c`, `GV_Category__c`, `GV_Image_Key__c`, `GV_Material__c`, `GV_Weight_Grams__c`; price DTO is active Retail `UnitPrice`; in-stock DTO is boolean only. `GV_Published__c`/`IsActive` may be used internally to filter. Never expose private inventory row IDs/quantity, order/payment fields, Contact email/phone. Guest Category allowlist: `Id`, `Name`, `GV_Slug__c`, `GV_Audience__c`, `GV_Sort_Order__c`.

**Staff FLS:** Jewellery Manager editable only catalogue fields and Retail pricing; Inventory Manager editable only `GV_On_Hand__c`/`GV_Reorder_Level__c`, not `GV_Product_Key__c` after creation; Order Manager can update `GV_Status__c` through approved Lightning action/guard, not order totals, customer or payment IDs; Payment status fields writeable only to trusted payment domain service. Customer Order fields are not direct editable API fields.

**Apex guidelines (explicit across API versions):** use `with sharing` on controllers. Use `WITH USER_MODE` for public catalogue reads or appropriate explicit CRUD/FLS checks; note that API v67+ defaults differ from earlier versions, so pin API version and declare modes. Internal domain operations that must change privileged records should use intentionally narrow system-mode access after validating caller permission, authenticated Contact ownership, status transition, posted amount and expected data. Avoid `without sharing` on public-facing controllers; use specific privileged domain helper only if truly necessary. User-supplied SOQL identifiers/order-by values must be enum allowlisted; never concatenate arbitrary input. Never expose generic `getRecordById` to storefront.

**Experience site class access:** Guest: only `GV_CatalogueController` (when its access model is verified); Customer: `GV_CatalogueController`, `GV_CheckoutController`, `GV_PaymentController`, `GV_OrderController`; no controllers/REST handlers exposing internal credentials. Managers use standard Lightning record pages with permission sets, not privileged guest-callable LWCs.

## 6. Security implementation checklist

1. Export current Profile/Permission Set/OWD metadata before editing. Confirm actual guest profile vs customer profile and required license.
2. Restrict guest Apex classes and object permissions to read-only public data; check site pages and API exposure.
3. Configure category sharing only for published active records. Inspect supported Product2/Pricebook sharing and test guest read in user mode; do not loosen private object OWD.
4. Configure staff permission sets: `GV_Catalogue_Manager`, `GV_Inventory_Manager`, `GV_Order_Manager`; customer permission set `GV_Storefront_Customer` as supported by license.
5. Apply FLS on newly added fields. Review `Object Manager → Fields & Relationships → Set Field-Level Security` and deployed profile/permission set metadata.
6. Ensure password reset, registration, user activation, login/logout and failed login tested using two non-admin external accounts.
7. Ensure each created User links to a unique Contact and inspect whether the shared Account leaks anything through standard Experience Cloud pages, reports, related lists or APIs.
8. Enforce `GV_CheckoutService` source-of-truth on prices, quantities, addresses and stock; no direct customer DML to `GV_Order__c`, `GV_Payment__c` or `GV_Inventory__c`.
9. Configure `GV_Razorpay_Test` Named Credential, `GV_Razorpay_Test_Auth` External Credential and principal access only to trusted service execution context; test key not visible in repo/metadata/LWC/logs.
10. Whitelist required Razorpay script/frame endpoints through Experience Builder CSP as strictly as feasible, after checking LWR and Razorpay current integration requirements. Never disable CSP wholesale.
11. Test ID tampering, URL deep-linking, guest page source, Lightning network responses, browser localStorage, user-mode queries and permission-dependent staff access.
12. Review security after every new custom LWC/Apex class or site page; record evidence in [Doc 11](11_TESTING_DOCUMENTATION.md).

## 7. Formal security acceptance tests

| Test | Attack / check | Expected outcome |
|---|---|---|
| TC-02 | Guest invoke checkout/order Apex or access private objects | Unauthorized, no data/DML |
| TC-06 | Guest query draft/unpublished product | No product DTO returned |
| TC-10 | New user login and self-registration with separate Contacts | Success, no unintended peer visibility |
| TC-14 | Submit somebody else's ContactId/OrderId | Ignored or denied; server derives owner |
| TC-20 | Scan page, JS, Git metadata for gateway secret | No secret anywhere |
| TC-22 | Forged callback without captured provider state/signature | Rejected, no deduction |
| TC-24 | Stolen/altered gateway attempt or wrong amount | Rejected, no deduction |
| TC-29 | Customer B reads Customer A Order Id | Denied with safe generic error |
| TC-30 | Guest directly access order/payment URLs, APIs | Denied, no PII |
| TC-31 | Inventory staff edit a payment secret/order total | Denied |
| TC-34 | CSP/XSS/local storage inspection | No confidential data or unsafe inline script |

All are planned, none are declared passed until executed with a date, actor, evidence and result.

## 8. External references

Salesforce guest policies: https://help.salesforce.com/s/articleView?id=platform.networks_guest_policies_timelines.htm&type=5  
Guest sharing rules: https://help.salesforce.com/s/articleView?id=platform.security_sharing_rules_guest.htm&type=5  
Guest Security 2026 best practices: https://help.salesforce.com/s/articleView?id=000390407&type=1  
Apex modes and FLS: https://developer.salesforce.com/docs/platform/apex-dev/references/apex-dev/apex-classes-perms-enforcing.html  
Sensitive data: https://developer.salesforce.com/docs/platform/secure-coding/guide/secure-coding-storing-sensitive-data.html

---
[Architecture](02_SYSTEM_ARCHITECTURE.md) · [Data](03_DATA_MODEL.md) · [Tests](11_TESTING_DOCUMENTATION.md)