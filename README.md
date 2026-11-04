# GoldVuee — Salesforce Jewellery E-Commerce Documentation

GoldVuee is a Salesforce Experience Cloud LWR jewellery storefront and inventory-management project with a secure **7-day MVP**: browse → customer login → cart → pending order → Razorpay **test** payment → verified capture → exactly-once stock deduction → own order history.

> **Status:** Architecture and implementation blueprint only. The Salesforce org, code deployment, payment connection and tests have **not** been independently verified. Do not describe features as implemented or passed until evidence is recorded in [11_TESTING_DOCUMENTATION.md](11_TESTING_DOCUMENTATION.md).

## Documentation index

| File | Purpose |
|---|---|
| [01_PROJECT_OVERVIEW.md](01_PROJECT_OVERVIEW.md) | Goals, requirements, MVP vs future, acceptance criteria |
| [02_SYSTEM_ARCHITECTURE.md](02_SYSTEM_ARCHITECTURE.md) | **Canonical architecture and API naming registry** |
| [03_DATA_MODEL.md](03_DATA_MODEL.md) | Objects, fields, relationships and ER diagram |
| [04_SECURITY_ARCHITECTURE.md](04_SECURITY_ARCHITECTURE.md) | Profiles, permission sets, sharing, data isolation |
| [05_FLOW_DOCUMENTATION.md](05_FLOW_DOCUMENTATION.md) | Flow inventory, execution, optional notification |
| [06_APEX_DOCUMENTATION.md](06_APEX_DOCUMENTATION.md) | Classes, methods, transactions, tests |
| [07_LWC_DOCUMENTATION.md](07_LWC_DOCUMENTATION.md) | Components, page ownership and behavior |
| [08_INTEGRATION_DOCUMENTATION.md](08_INTEGRATION_DOCUMENTATION.md) | Razorpay test-only payment and security rules |
| [09_UI_UX_DESIGN.md](09_UI_UX_DESIGN.md) | Pages, navigation, tokens and accessibility |
| [10_IMPLEMENTATION_ROADMAP.md](10_IMPLEMENTATION_ROADMAP.md) | Seven-day sprint and separate post-MVP backlog |
| [11_TESTING_DOCUMENTATION.md](11_TESTING_DOCUMENTATION.md) | 38 planned test cases and execution evidence |
| [12_DEPLOYMENT_GUIDE.md](12_DEPLOYMENT_GUIDE.md) | GitHub workflow, deployment, rollback and release |
| [13_DEVELOPMENT_LOG.md](13_DEVELOPMENT_LOG.md) | Daily progress and unverified baseline |

**First read:** [02_SYSTEM_ARCHITECTURE.md](02_SYSTEM_ARCHITECTURE.md) for immutable design names; then [10_IMPLEMENTATION_ROADMAP.md](10_IMPLEMENTATION_ROADMAP.md) for the 7-day delivery sequence.

**No Azure, no live payment credentials, and no unnecessary 20-Flow demonstration.** Transactional payment/stock logic is owned by Apex; advanced B2B/wishlist/refunds/warehouses are post-MVP.
