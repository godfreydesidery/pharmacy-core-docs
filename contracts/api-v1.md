# Pharmacy Core API — v1 contract

The HTTP contract between `pharmacy-core-api` and its clients (`pharmacy-core-web`, POS terminals, partner systems). The running API also serves the generated OpenAPI 3.1 document at `/v3/api-docs` and Swagger UI at `/swagger-ui.html`. If this page and the generated document disagree, the generated document wins.

- Base path: `/api/v1`
- Local base URL: `http://localhost:8080/api/v1`
- JSON only (`application/json`), UTF-8, field names in camelCase.
- Identifiers are UUIDv7 strings (`uid`). Business references such as `INV-HQ-000123` are for people, not for lookups.
- Times are ISO 8601 / RFC 3339 in UTC (`2026-11-02T09:15:00Z`). Dates are `YYYY-MM-DD`.
- Money is a JSON number in the tenant's base currency (`GET /settings` → `currency`, an ISO 4217 code), unless a `currency` field beside it says otherwise. Amounts carry their currency's ISO 4217 minor units (TZS and USD 2, JPY 0, KWD 3). See §16.
- Quantities are integers in the product's base unit (tablet, capsule, bottle…).

## 1. Headers

| Header | When | Meaning |
|---|---|---|
| `Authorization: Bearer <token>` | every route except sign-in and health | Session token from `POST /auth/sign-in` |
| `X-Tenant-Id: <tenantUid>` | every tenant route | The pharmacy business the request acts in. A member must hold a membership in it; the Root administrator may name any tenant. Ignored on `/platform/*` and account (`/auth/*`) routes. |
| `X-Branch-Id: <branchUid>` | every branch route (marked **[branch]**) | The branch the request acts in. It must belong to the tenant and be one the membership may use. Optional on reports and the dashboard, where it is omitted to mean "all my branches". |
| `X-Correlation-Id` | optional | Echoed back. Generated when absent. |
| `Idempotency-Key` | optional on `POST /sales` and the money-moving POSTs below | On `POST /sales`: same as `idempotencyKey` in the body; a repeated key returns the first sale. Elsewhere (IETF draft *The Idempotency-Key HTTP Header Field*, ≤ 255 visible ASCII characters), on: `/receivables/payments`, `/receivables/write-offs`, `/finance/supplier-payments`, `/expenses`, `/sales/{uid}/returns`, `/vouchers`, `/vouchers/{uid}/void`, `/stock/adjustments`, `/stock/opening`, `/transfers`, `/transfers/{uid}/receive`, `/insurance/remittances`, `/goods-receipts`, `/destructions`, `/supplier-returns`, `/pos/payment-requests`. The first request with a key is recorded; a repeat with the same body, route and user gets the stored 2xx answer with `Idempotent-Replayed: true` and nothing runs twice; another body → `422 idempotency_key_reused`; while the first still runs → `409 idempotency_in_progress`; a failed first request frees the key. Keys last 24 hours. A bad key → `400 invalid_idempotency_key`. |

**Paths are read one way.** A path with `;`, an encoded `/`, `\` or `%`, `//`, a `.`/`..` segment or a control character answers `400 malformed_request`, before any route.

**Account messages** (`access.*` templates: invitation, reset, codes) are the platform's: a pharmacy's override answers `422 template_fixed`.

## 2. Envelopes

**Success** (every 2xx):

```json
{ "success": true, "code": "OK", "message": "Products retrieved", "timestamp": "2026-11-02T09:15:00.123Z", "data": { } }
```

`code` is `OK` (200) or `CREATED` (201).

**Lists**: `data` is a page.

```json
{ "items": [ ], "nextCursor": "0193f1c2-…", "total": null }
```

Query parameters `cursor` (the `nextCursor` of the previous page) and `limit` (1–100, default 25). `nextCursor` is `null` on the last page. Lists are newest first unless stated otherwise.

**Errors**: RFC 9457 Problem Details, `Content-Type: application/problem+json`.

```json
{
  "type": "https://pharmacy-core/problems/insufficient_stock",
  "title": "Unprocessable request",
  "status": 422,
  "detail": "Only 12 of Paracetamol 500mg are in stock at this branch.",
  "reason": "insufficient_stock",
  "field": "lines[0].quantity",
  "correlationId": "5b0d…"
}
```

| Status | Typical `reason` |
|---|---|
| 400 | `malformed_request` |
| 401 | `unauthenticated`, `session_expired`, `invalid_credentials` |
| 403 | `forbidden`, `tenant_required`, `branch_required`, `branch_not_allowed`, `tenant_suspended` |
| 404 | `<thing>_not_found` |
| 409 | `conflict`, `duplicate_code`, `duplicate_email`, `stale_version` |
| 422 | `invalid_<field>`, `insufficient_stock`, `batch_expired`, `prescription_required`, `payment_short`, … |

## 3. Permissions

Every route needs one permission (listed per route). A Root administrator holds all of them in every tenant. `GET /permissions` lists the catalogue.

| Code | Grants |
|---|---|
| `platform.manage` | Root administrator only: tenants and root administrators |
| `dashboard.view` | Dashboard |
| `sales.sell` | POS: create sales |
| `sales.view` | Sales history |
| `sales.refund` | Sales returns |
| `sales.discount` | Apply a discount at POS |
| `sales.credit` | Sell on account: credit sales and balances put on account |
| `sales.price_plan` | Sell under another pricing plan than the customer's own |
| `sales.voucher` | Issue vouchers (`POST /vouchers`): prepaid value sold for money. Not in the Cashier or Pharmacist presets |
| `inventory.view` | Stock, batches, expiry, movements |
| `inventory.adjust` | Stock adjustments, opening stock |
| `inventory.transfer` | Branch transfers |
| `purchasing.view` | Purchase orders, goods receipts |
| `purchasing.manage` | Create and cancel purchase orders |
| `purchasing.receive` | Record goods receipts |
| `catalogue.view` / `catalogue.manage` | Products, categories, manufacturers |
| `partners.view` / `partners.manage` | Customers, suppliers |
| `finance.view` / `finance.manage` | Expenses, accounts, journal |
| `reports.view` | Reports |
| `users.manage` | Users, memberships, roles |
| `branches.manage` | Branches |
| `settings.manage` | Tenant settings |
| `integrations.manage` | Webhooks |

Default roles seeded in every new tenant: `ADMIN` (all tenant permissions), `BRANCH_MANAGER`, `PHARMACIST`, `CASHIER`, `STORE_KEEPER`, `ACCOUNTANT`.

## 4. Authentication — `/auth`

### `POST /auth/sign-in` (public)
Request `{ "email": "admin@afya.co.tz", "password": "…" }`
Response `201` → `Session`:

```json
{
  "token": "pcs_…",
  "expiresAt": "2026-11-02T21:15:00Z",
  "user": { "uid": "…", "name": "Amina Hassan", "email": "admin@afya.co.tz", "rootAdmin": false },
  "memberships": [
    {
      "uid": "…", "tenantUid": "…", "tenantCode": "AFYA", "tenantName": "Afya Pharmacy",
      "tenantStatus": "ACTIVE",
      "roleUid": "…", "roleCode": "ADMIN", "roleName": "Administrator",
      "permissions": ["sales.sell", "…"],
      "allBranches": true,
      "branches": [ { "uid": "…", "code": "HQ", "name": "Head Office" } ]
    }
  ]
}
```

A Root administrator has `rootAdmin: true` and usually no memberships. It picks a tenant from `GET /platform/tenants` and that tenant's branches from `GET /branches` (sent with `X-Tenant-Id`).

When the account has two-step verification on, sign-in answers **`202`** with a challenge instead of a session (see §17 Security):

```json
{ "challengeToken": "pch_…", "methods": ["TOTP", "RECOVERY_CODE", "PASSKEY"], "expiresAt": "…" }
```

`user.twoFactorSetupRequired` is `true` for a root administrator without two-step verification when `pharmacy.access.require-2fa-for-root=true` (the default; `PHARMACY_REQUIRE_2FA_FOR_ROOT`). Until they set it up, every route except their own account routes answers `403 two_factor_setup_required`.

Errors:
- `401 invalid_credentials`.
- `423 account_locked` with `retryAfterSeconds`, after 5 failures within 15 minutes; it lasts 15 minutes.
- `423 client_locked`, after 20 failures from one IP.
- `429 too_many_requests` with `Retry-After`, beyond 20 requests a minute per IP across the public `/auth` routes.

### `GET /auth/me` → `Session` without `token` and `expiresAt`.
### `POST /auth/sign-out` → `200`, revokes the token.
### `POST /auth/change-password` `{ "currentPassword", "newPassword" }` → `200`, and signs out every other session. A wrong current password counts toward the account lock like a failed sign-in.

### Passwords
8 to 128 characters (counted as characters), no composition rules (NIST SP 800-63B). `422 common_password` for a commonly used password, one character repeated, a run such as `12345678` or `abcdefgh`, or one made of the account's email name; `422 weak_password` for the length. Stored as Argon2id (19 MiB, 2 iterations). `429` when the server is busy hashing.

### Confirming it is you ("sudo mode") — `POST /auth/confirm-identity` `AUTHENTICATED`
`{ "password", "code" }` → `200`. `code` (TOTP or recovery code) only when two-step verification is on; without it `422 invalid_code`; a wrong password `422 invalid_password`; both count toward the account lock.

Routes that change how the account is entered (2FA setup, enable, disable and recovery codes; passkey options, register and delete; phone send and verify) need the session to have proved its password within **10 minutes** (`pharmacy.access.step-up-minutes`), at sign-in or here. Otherwise they answer `403 reauthentication_required`: ask for the password, call this route, and send the request again.

A session records what it proved. `secondFactor` (used by a pharmacy's "Require two-factor") is true only for a session signed in with a TOTP or recovery code or a user-verified passkey, or one that just turned two-step verification on; an account that merely has a passkey does not count.

### Reset links and codes
- Only the newest reset link works; a completed reset, a new password or a new email ends the others. At most 3 links an hour per account (more are dropped with the same `202`).
- A password reset (by email or phone) or a recovery removes every passkey and signs out every session.
- A phone resets a password only if it was verified in the last 180 days; never for a root administrator.
- Sent account messages (links, codes) keep no text: the delivery log shows that one was sent, `resend` answers `422 cannot_resend`, and a data export leaves the text out.

## 5. Platform (Root administrator) — `/platform` · `platform.manage`

| Route | Body → Response |
|---|---|
| `GET /platform/tenants?q=` | page of `Tenant` |
| `POST /platform/tenants` | `RegisterTenant` → `Tenant` (201) |
| `GET /platform/tenants/{uid}` | `Tenant` |
| `POST /platform/tenants/{uid}/suspend` | `{ "reason" }` → `Tenant` |
| `POST /platform/tenants/{uid}/reactivate` | → `Tenant` |
| `GET /platform/root-admins` | `User[]` |
| `POST /platform/root-admins` | `{ "name", "email", "password" }` → `User` (201) |
| `GET /platform/overview` | `{ "tenants", "activeTenants", "suspendedTenants", "branches", "users" }` |

`Tenant`:
```json
{ "uid": "…", "code": "AFYA", "name": "Afya Pharmacy", "legalName": "Afya Pharmacy Ltd", "country": "TZ",
  "currency": "TZS", "timezone": "Africa/Dar_es_Salaam", "status": "ACTIVE", "statusReason": null,
  "branchCount": 2, "createdAt": "…" }
```

`RegisterTenant`:
```json
{ "code": "AFYA", "name": "Afya Pharmacy", "legalName": "Afya Pharmacy Ltd", "country": "TZ",
  "currency": "TZS", "timezone": "Africa/Dar_es_Salaam",
  "headOffice": { "code": "HQ", "name": "Head Office", "address": "Samora Ave", "phone": "+255…" },
  "admin": { "name": "Amina Hassan", "email": "admin@afya.co.tz", "password": "…" } }
```
`code` is 2–12 uppercase letters or digits and unique.

## 6. Tenant administration

### Branches — `branches.manage` (reads need any tenant permission)
| Route | |
|---|---|
| `GET /branches` | `Branch[]` (all the tenant's branches, not paged) |
| `POST /branches` | `{ "code", "name", "address", "phone", "licenceNumber" }` → `Branch` |
| `PUT /branches/{uid}` | same body + `"active": true` → `Branch` |

`Branch`: `{ "uid", "code", "name", "address", "phone", "licenceNumber", "headOffice", "active", "createdAt" }`

### Settings — `settings.manage` (GET needs any tenant permission)
`GET /settings`, `PUT /settings` → `Settings`:
```json
{ "currency": "TZS", "timezone": "Africa/Dar_es_Salaam", "defaultTaxRatePercent": 0, "invoicePrefix": "INV",
  "receiptFooter": "Thank you. Get well soon.", "maxDiscountPercent": 10, "expiryAlertDays": 90,
  "lowStockAlerts": true, "acceptedCurrencies": ["TZS", "USD", "KES"] }
```
- `currency` and `timezone` are read-only after registration.
- `acceptedCurrencies` lists the base first, then the foreign currencies the pharmacy takes payments and buys in.
- `PUT` replaces the list. The base is always kept, and an unknown code is `422 invalid_currency`.

### Users and roles — `users.manage`
| Route | |
|---|---|
| `GET /users?q=` | page of `Member` |
| `POST /users` | `{ "name", "email", "password", "roleUid", "allBranches", "branchUids": [] }` → `Member`. An existing email gets a membership in this tenant and keeps its password. |
| `PUT /users/{membershipUid}` | `{ "roleUid", "allBranches", "branchUids", "active" }` → `Member` |
| `GET /roles` | `Role[]` |
| `POST /roles` | `{ "code", "name", "description", "permissions": [] }` → `Role` |
| `PUT /roles/{uid}` | `{ "name", "description", "permissions": [] }` → `Role` (system roles: permissions only) |
| `GET /permissions` | `[{ "code", "area", "description" }]` |

Who may grant what (least privilege):
- A role may include, and a member may be given a role with, only permissions the caller holds: `403 permission_not_held`.
- The built-in Administrator role always has every permission: `422 admin_role_fixed`.
- A change that would leave no active member able to manage people (deactivating, demoting, or taking `users.manage` from the role) is refused: `409 last_admin`.
- `GET /roles` lists this pharmacy's roles only.

`Member`: `{ "membershipUid", "userUid", "name", "email", "roleUid", "roleCode", "roleName", "allBranches", "branchUids", "active", "lastSignInAt", "createdAt" }`
`Role`: `{ "uid", "code", "name", "description", "system", "permissions": [] }`

## 7. Catalogue — `catalogue.view` / `catalogue.manage`

| Route | |
|---|---|
| `GET /categories` | `Category[]`, ordered by path. The first read in a tenant without categories seeds the default tree (Medicines, Medical Equipment, Medical Supplies, Personal Care & Cosmetics, Supplements & Nutrition, Mother & Baby, with their sub-categories). |
| `POST /categories`, `PUT /categories/{uid}` | `{ "name", "description", "parentUid" }`. `parentUid` is null at the top. The tree is at most 3 levels deep (`422 category_too_deep`) and has no cycles (`422 category_cycle`). Names are unique among siblings (`409 duplicate_name`). |
| `GET /product-types` | `[{ "code": "MEDICAL_EQUIPMENT", "defaultTrackingMode": "SERIAL" }, …]` |
| `GET /tracking-modes` | `["BATCH_EXPIRY","BATCH","SERIAL","QUANTITY"]` |
| `GET /units-of-measure` | `["PIECE","BOX","PACK","STRIP","BOTTLE","TUBE","SACHET","VIAL","PAIR","SET","KIT","ROLL"]` |
| `GET /manufacturers` | `Manufacturer[]` |
| `POST /manufacturers`, `PUT /manufacturers/{uid}` | `{ "name", "country" }` |
| `GET /dosage-forms` | `["TABLET","CAPSULE","SYRUP","SUSPENSION","INJECTION","INHALER","CREAM","OINTMENT","GEL","DROPS","SACHET","SUPPOSITORY","DEVICE","OTHER"]` |
| `GET /products?q=&categoryUid=&active=&productType=` | page of `Product`. `q` matches name, generic name, SKU, GTIN, brand and model. `categoryUid` includes its sub-categories. |
| `POST /products` | `ProductInput` → `Product` |
| `GET /products/{uid}` | `Product` |
| `PUT /products/{uid}` | `ProductInput` + `"active"` + `"version"` → `Product` |

`ProductInput`:
```json
{ "name": "Paracetamol 500mg", "genericName": "Paracetamol", "sku": "PCM500", "gtin": "06001234567890",
  "atcCode": "N02BE01", "categoryUid": "…", "manufacturerUid": "…", "dosageForm": "TABLET",
  "strength": "500 mg", "packSize": "Strip of 10", "schedule": "OTC",
  "sellingPrice": 1500.00, "costPrice": 800.00, "taxRatePercent": 0, "reorderLevel": 200,
  "storage": "ROOM_TEMPERATURE" }
```
`schedule`: `OTC` | `PRESCRIPTION` | `CONTROLLED`. `storage`: `ROOM_TEMPERATURE` | `COOL_DRY` | `REFRIGERATED`.
`Product` = input plus `uid`, `categoryName`, `manufacturerName`, `active`, `version`, `createdAt`.

**Product types and tracking.** A pharmacy also sells equipment, supplies, cosmetics, supplements and baby care. `ProductInput` gains these fields (all optional):

```json
{ "productType": "MEDICAL_EQUIPMENT", "trackingMode": "SERIAL", "unitOfMeasure": "PIECE",
  "brand": "Invacare", "model": "Action 1", "warrantyMonths": 12, "requiresSerialAtSale": true }
```

- **`productType`** is one of `MEDICINE` (the default), `MEDICAL_EQUIPMENT`, `MEDICAL_SUPPLY`, `COSMETIC`, `SUPPLEMENT`, `BABY_CARE` or `OTHER`.
  - A `MEDICINE` needs `genericName` (`422 invalid_generic_name`) and `dosageForm` (`422 invalid_dosage_form`).
  - Any other type has no `dosageForm` or `atcCode`, and its `schedule` is always `OTC`.
- **`trackingMode`** defaults by type:

  | Mode | Default for | Behaviour |
  |---|---|---|
  | `BATCH_EXPIRY` | `MEDICINE` | Batches with expiry, sold earliest expiry first (FEFO) |
  | `SERIAL` | `MEDICAL_EQUIPMENT` | Each unit has its serial number |
  | `BATCH` | every other type | Batches without expiry, sold first in, first out (FIFO) |
  | `QUANTITY` | — | No batches: one implicit batch `-` |

- **`unitOfMeasure`** defaults to `PIECE`.
- **`warrantyMonths`** is 0–120 (`422 invalid_warranty_months`).
- **`requiresSerialAtSale`** applies to `SERIAL` products only and defaults to true. When it is false, the POS may omit serials and the oldest units are sold.

`Product` returns all of these fields. `Category` gains `parentUid` and `path` (for example "Medical Equipment › Mobility Aids").

## 8. Partners — `partners.view` / `partners.manage`

| Route | |
|---|---|
| `GET /customers?q=&type=` | page of `Customer` |
| `POST /customers`, `PUT /customers/{uid}` | `{ "name", "phone", "email", "type", "insuranceProvider", "insuranceNumber", "creditLimit", "address", "active" }` |
| `GET /customers/{uid}` | `Customer` |
| `GET /suppliers?q=` | page of `Supplier` |
| `POST /suppliers`, `PUT /suppliers/{uid}` | `{ "name", "contactPerson", "phone", "email", "tin", "paymentTerms", "address", "active" }` |
| `GET /suppliers/{uid}` | `Supplier` |

Customer `type`: `RETAIL` | `WHOLESALE` | `INSURANCE`. `paymentTerms`: `COD` | `NET_15` | `NET_30` | `NET_45` | `NET_60`.
Both carry `uid`, `code` (`CUS-000001`, `SUP-000001`), `createdAt`.

## 9. Inventory **[branch]** — `inventory.view` / `inventory.adjust` / `inventory.transfer`

| Route | |
|---|---|
| `GET /stock?q=&status=&categoryUid=&productType=` | page of `StockItem`, ordered by product name. `status`: `IN_STOCK` \| `LOW_STOCK` \| `OUT_OF_STOCK` |
| `GET /stock/{productUid}/serials?status=` | `SerialUnit[]`: the branch's units of a SERIAL product, by serial number. `status`: `IN_STOCK` \| `SOLD` \| `RETURNED` \| `IN_TRANSIT` \| `WRITTEN_OFF` |
| `GET /stock/summary` | `{ "products", "inStock", "lowStock", "outOfStock", "expiringSoon", "expired", "costValue", "retailValue" }` |
| `GET /stock/{productUid}/batches` | `Batch[]` with quantity at this branch, earliest expiry first |
| `GET /stock/expiring?withinDays=90` | page of `ExpiringBatch`, earliest first. Expired batches have negative `daysLeft`. |
| `GET /stock/movements?productUid=&type=` | page of `Movement` |
| `POST /stock/opening` | `{ "productUid", "batchNumber", "expiryDate", "quantity", "unitCost" }` → `Movement` (opening balance) |
| `POST /stock/adjustments` | `{ "productUid", "batchUid", "quantityChange", "reasonCode", "note" }` → `Movement`. `quantityChange` is signed. `reasonCode`: `DAMAGE` \| `THEFT_LOSS` \| `COUNT_CORRECTION` \| `EXPIRED_WRITE_OFF` \| `SAMPLE` \| `OTHER` |
| `GET /transfers?direction=OUTGOING\|INCOMING` | page of `Transfer` |
| `POST /transfers` | `{ "toBranchUid", "note", "lines": [{ "productUid", "quantity" }] }` → `Transfer`, sent from the `X-Branch-Id` branch. Batches are picked earliest expiry first (FEFO) and the transfer is `IN_TRANSIT`. |
| `POST /transfers/{uid}/receive` | → `Transfer` (`RECEIVED`). `X-Branch-Id` must be the destination. |

```text
StockItem     { productUid, productName, genericName, sku, dosageForm, strength, categoryName,
                quantity, reorderLevel, status, nearestExpiry, batchCount, sellingPrice, costValue }
Batch         { uid, productUid, batchNumber, expiryDate, quantity, unitCost, daysLeft, expired }
ExpiringBatch { batchUid, productUid, productName, batchNumber, expiryDate, daysLeft, quantity, costValue }
Movement      { uid, productUid, productName, batchUid, batchNumber, type, quantityChange, balanceAfter,
                reference, reasonCode, note, createdAt, createdBy }
Transfer      { uid, number, fromBranchUid, fromBranchName, toBranchUid, toBranchName, status, note,
                lines: [{ productUid, productName, batchUid, batchNumber, expiryDate, quantity }],
                createdAt, receivedAt }
```
Movement `type`: `OPENING` | `RECEIPT` | `SALE` | `SALE_RETURN` | `ADJUSTMENT` | `TRANSFER_OUT` | `TRANSFER_IN`.

## 10. Purchasing **[branch]** — `purchasing.view` / `purchasing.manage` / `purchasing.receive`

| Route | |
|---|---|
| `GET /purchase-orders?status=&supplierUid=` | page of `PurchaseOrder` |
| `POST /purchase-orders` | `{ "supplierUid", "expectedDate", "note", "currency", "exchangeRate", "lines": [{ "productUid", "quantity", "unitCost" }] }` → `PurchaseOrder` (`ORDERED`). `currency` defaults to the base and must be accepted (`422 currency_not_accepted`). `exchangeRate` is base units per 1 order unit; it defaults to today's rate (`422 rate_missing` when there is none) and is locked on the order. |
| `GET /purchase-orders/{uid}` | `PurchaseOrder` |
| `POST /purchase-orders/{uid}/cancel` | `{ "reason" }` → `PurchaseOrder` |
| `GET /goods-receipts?supplierUid=` | page of `GoodsReceipt` |
| `POST /goods-receipts` | `{ "purchaseOrderUid" (optional), "supplierUid", "supplierInvoiceNumber", "note", "currency", "exchangeRate", "lines": [{ "productUid", "batchNumber", "expiryDate", "quantity", "unitCost" }] }` → `GoodsReceipt`. Stock goes up at this branch. The purchase order moves to `PARTIALLY_RECEIVED` or `RECEIVED`. |
| `GET /goods-receipts/{uid}` | `GoodsReceipt` |

```text
PurchaseOrder { uid, number, branchUid, supplierUid, supplierName, status, expectedDate, note, total,
                lines: [{ uid, productUid, productName, quantity, receivedQuantity, unitCost, lineTotal }],
                createdAt, createdBy, currency, exchangeRate, baseTotal }
GoodsReceipt  { uid, number, branchUid, purchaseOrderUid, purchaseOrderNumber, supplierUid, supplierName,
                supplierInvoiceNumber, note, total, currency, exchangeRate, baseTotal,
                lines: [{ productUid, productName, batchNumber, expiryDate, quantity, unitCost, lineTotal, baseUnitCost }],
                createdAt, createdBy }
```
Purchase order `status`: `ORDERED` | `PARTIALLY_RECEIVED` | `RECEIVED` | `CANCELLED`.

Currency rules for purchasing:
- `total`, `unitCost` and `lineTotal` are in the document's `currency`.
- `baseTotal` and each receipt line's `baseUnitCost` are in the base currency at the document's `exchangeRate`, which is locked when the document is made.
- Stock is always costed in the base currency (`baseUnitCost`).
- A receipt against an order is in the order's currency (another one is `422 currency_mismatch`). It uses the order's locked rate, even for a partial receipt days later, unless `exchangeRate` is given: the supplier invoiced at another rate.

## 11. Sales and POS **[branch]** — `sales.sell` / `sales.view` / `sales.refund`

| Route | |
|---|---|
| `GET /pos/products?q=&categoryUid=` | page of `SellableProduct`: active products with their price and quantity on hand here |
| `POST /sales` | `CheckoutRequest` → `Sale` (201) |
| `GET /sales?q=&status=&from=&to=` | page of `SaleSummary` (`q` matches number and customer) |
| `GET /sales/{uid}` | `Sale` |
| `POST /sales/{uid}/returns` | `{ "reason", "lines": [{ "saleLineUid", "quantity" }] }` → `SaleReturn`. Stock goes back to the batch sold. The refund goes back **the way the sale was paid**, split by share (see Returns below); `refundMethod` is optional and, if sent, must be a way the sale was paid, else `422 refund_method_not_allowed`. |
| `GET /sale-returns` | page of `SaleReturn` |

`CheckoutRequest`:
```json
{ "idempotencyKey": "0193f1c2-…", "customerUid": null, "discountPercent": 0,
  "lines": [ { "productUid": "…", "quantity": 2 } ],
  "payments": [ { "method": "CASH", "amount": 10.00, "currency": "USD", "exchangeRate": 2650, "reference": null },
                { "method": "MOBILE_MONEY", "amount": 13500.00, "reference": "MP-88" } ],
  "prescription": { "prescriberName": "Dr. Mushi", "prescriberRegistration": "MCT-1234",
                    "reference": "RX-77", "patientName": "John M." } }
```
- Payment `method`: `CASH` | `CARD` | `MOBILE_MONEY` | `INSURANCE` | `CREDIT`. `CREDIT` and `INSURANCE` need a customer.
- The total paid must cover the sale total, or the sale is refused with `422 payment_short`.
- **Multi-currency tender:**
  - A payment may be in any accepted currency (`currency`, default the base; `422 currency_not_accepted` otherwise).
  - Its `exchangeRate` (base units per 1 unit) defaults to today's rate and is locked on the sale. With no rate, the sale is refused with `422 rate_missing` (field `payments[i].currency`).
  - Coverage and change are computed in the base currency. Change is always given in base cash, and only from cash tendered (`422 overpaid_non_cash`). Prices and totals are always in the base.
  - `Sale.payments[]` is `{ method, amount, reference, currency, exchangeRate, baseAmount }`: `amount` is in `currency` and `baseAmount` in the base.
  - Refunds are in the base currency, even for a sale tendered in a foreign currency. No exchange difference arises, because the refund repays the base value the sale was booked at.
- A product with schedule `PRESCRIPTION` or `CONTROLLED` needs `prescription`, or the sale is refused with `422 prescription_required`.
- Batches are picked earliest expiry first. An expired batch is never sold.
- `discountPercent` needs `sales.discount` and must not exceed `maxDiscountPercent`.

```text
SellableProduct { productUid, name, genericName, sku, gtin, dosageForm, strength, categoryUid, categoryName,
                  schedule, sellingPrice, taxRatePercent, quantityOnHand }
Sale        { uid, number, branchUid, branchName, customerUid, customerName, status, discountPercent,
              subtotal, discountTotal, taxTotal, total, paidTotal, changeDue,
              lines: [{ uid, productUid, productName, batchUid, batchNumber, expiryDate, quantity,
                        returnedQuantity, unitPrice, discount, tax, lineTotal }],
              payments: [{ method, amount, reference }],
              prescription, cashierName, createdAt }
SaleSummary { uid, number, customerName, itemCount, total, paidTotal, paymentMethods: [], status, cashierName, createdAt }
SaleReturn  { uid, number, saleUid, saleNumber, reason, refundMethod, refundTotal,
              lines: [{ saleLineUid, productName, batchNumber, quantity, refund }], createdAt, createdBy }
```
Sale `status`: `COMPLETED` | `PARTIALLY_RETURNED` | `RETURNED`.

## 11a. Tracking modes and serial numbers in stock, receipts and sales

Every document takes the product's tracking mode (§7) into account.

**Stock in** (`POST /goods-receipts` lines, `POST /stock/opening`):

| Tracking | What the line carries |
|---|---|
| `BATCH_EXPIRY` | `batchNumber` and `expiryDate` |
| `BATCH` | `batchNumber`; `expiryDate` is ignored |
| `QUANTITY` | `quantity` only, kept on the implicit batch `-` |
| `SERIAL` | `"serialNumbers": ["WCH-0001", …]`, one per unit (`422 serials_mismatch`, `409 duplicate_serial`); `batchNumber` is optional |

A serial number on a product that is not tracked by serial is refused with `422 serials_not_tracked`.

**Picking:**
- Batches with expiry: earliest expiry first (FEFO); expired batches are never sold.
- Batches without expiry: first in, first out (FIFO).
- `SERIAL`: the named units, or the oldest in stock when none are named.

**Sales:**
- Each `CheckoutRequest.lines[]` entry may carry `"serialNumbers"`.
  - A product with `requiresSerialAtSale` refuses a line without them: `422 serial_required` on `lines[i].serialNumbers`.
  - Named units must be in stock at the branch: `422 serial_not_in_stock`.
- `Sale.lines[]` gains `serialNumbers`, `returnedSerialNumbers` and `warrantyEndsOn`. The warranty end is the sale date plus `warrantyMonths`, minus one day.
- Return lines (`POST /sales/{uid}/returns`) may name `"serialNumbers"`; otherwise the first units not yet returned come back. A unit not sold on the line, or already returned, is refused with `422 serial_not_on_sale`.

**Adjustments and transfers:**
- `POST /stock/adjustments` takes `"serialNumbers"`: the new units for an increase, or the units written off for a decrease. With none named on a decrease, the batch's oldest units are written off.
- `POST /transfers` lines may name `"serialNumbers"` (otherwise the oldest are sent); units are `IN_TRANSIT` until received.

**New fields on the views:**

| View | New fields |
|---|---|
| `StockItem` | `productType`, `unitOfMeasure`, `trackingMode` |
| `Batch` | `expiryDate` and `daysLeft` are null without expiry |
| `Movement` | `serialNumbers` |
| `TransferLine` | `serialNumbers`; `expiryDate` may be null |
| Goods receipt line | `serialNumbers` |
| Sale return line | `serialNumbers` |
| `SellableProduct` | `productType`, `trackingMode`, `unitOfMeasure`, `requiresSerialAtSale` |

```text
SerialUnit { uid, productUid, serialNumber, batchUid, batchNumber, branchUid, status, saleReference, soldAt, createdAt }
```

## 12. Finance — `finance.view` / `finance.manage`

| Route | |
|---|---|
| `GET /expenses?category=&from=&to=` | page of `Expense` |
| `POST /expenses` **[branch]** | `{ "category", "description", "amount", "expenseDate", "paymentMethod", "reference" }` → `Expense`. `paymentMethod`: `CASH` (default) | `BANK_TRANSFER` | `CARD` | `CHEQUE` | `MOBILE_MONEY`, else `422 invalid_payment_method`; `expenseDate` not after today in the pharmacy's time zone (`422 future_date`). |
| `GET /accounts` | `Account[]` with `balance` (chart of accounts) |
| `GET /journal?from=&to=` | page of `JournalEntry` |
| `GET /finance/supplier-payments?supplierUid=` | page of `SupplierPayment` |
| `POST /finance/supplier-payments` | `{ "supplierUid", "amount", "currency", "rate", "paymentMethod", "reference", "goodsReceiptUids": [], "paidOn" }` → `SupplierPayment` (201). Posts Dr 2000 Accounts Payable / Cr the payment account, and the realised exchange difference to 7000. Methods as for expenses (`BANK_TRANSFER` default); `paidOn` not in the future. The receipts named are settled in order, each up to what is still open of it; paying more than they leave open → `422 overpayment`. A payment naming no receipt is on account. |
| `GET /finance/reconciliation?days=7` **[branch optional]** | Documents recorded more than 15 minutes and at most `days` (1–92) ago with no journal entry (sales, returns, vouchers, receipts, supplier returns and payments, customer payments, remittances, stock adjustments, expenses), oldest first. An empty list means the books hold every one. |

- Expense `category`: `RENT` | `UTILITIES` | `SALARIES` | `TRANSPORT` | `MAINTENANCE` | `MARKETING` | `SUPPLIES` | `LICENCES` | `OTHER`.
- `Account`: `{ "code": "1000", "name": "Cash", "type": "ASSET", "balance": 0 }`. Types: `ASSET` | `LIABILITY` | `EQUITY` | `REVENUE` | `EXPENSE`.
- `JournalEntry`: `{ "uid", "number", "entryDate", "sourceType", "sourceReference", "description", "branchUid", "lines": [{ "accountCode", "accountName", "debit", "credit", "currency", "foreignAmount" }], "createdAt" }`. The journal is in the base currency. A line may note the foreign `currency` and `foreignAmount` behind it, for reference; both are null otherwise.
- `SupplierPayment`: `{ uid, number, supplierUid, supplierName, amount, currency, exchangeRate, baseAmount, bookedBaseAmount, fxDifference, paymentMethod, reference, goodsReceiptUids, paidOn, createdAt, createdBy }`.
  - `baseAmount` is the amount at the payment rate.
  - `bookedBaseAmount` is the amount at the rate the settled receipts were booked at, weighted by their totals. With no receipts named, it equals `baseAmount`.
  - `fxDifference` = `baseAmount − bookedBaseAmount`. A positive value is a realised loss (Dr 7000 FX Gains/Losses); a negative one is a gain (Cr 7000).
  - Named receipts must be the supplier's (`422 supplier_mismatch`) and in the payment's currency (`422 currency_mismatch`).
- The default chart gains `7000 FX Gains/Losses` (EXPENSE).
- Sales, returns, goods receipts and expenses post journal entries automatically, a few moments after they are recorded.

## 13. Reports — `dashboard.view` / `reports.view`

`X-Branch-Id` is optional on all of these. Without it, a report covers every branch the caller may use.

| Route | |
|---|---|
| `GET /dashboard?days=30` | `Dashboard` |
| `GET /reports/sales?from=&to=` | `{ "rows": [{ "date", "invoices", "gross", "discount", "tax", "net", "cost", "profit" }], "totals": { … } }` |
| `GET /reports/products?from=&to=&limit=20` | `[{ "productUid", "productName", "quantity", "net", "cost", "profit" }]`, best sellers first |
| `GET /reports/profit-loss?from=&to=` | `{ "revenue", "costOfSales", "grossProfit", "expenses": [{ "category", "amount" }], "totalExpenses", "netProfit" }` |
| `GET /reports/payments?from=&to=` | `[{ "method", "amount", "count" }]` |

`Dashboard`:
```json
{ "currency": "TZS", "periodDays": 30,
  "sales": { "value": 45250.00, "previous": 40220.00, "changePercent": 12.5 },
  "purchases": { "value": 22680.00, "previous": 20941.00, "changePercent": 8.3 },
  "grossProfit": { "value": 22570.00, "previous": 19508.00, "changePercent": 15.7 },
  "transactions": { "value": 324, "previous": 297, "changePercent": 9.1 },
  "customers": 1245,
  "stock": { "inStock": 768, "lowStock": 23, "outOfStock": 9, "expiringSoon": 56, "expired": 2 },
  "salesTrend": [ { "date": "2026-10-04", "total": 1210.50 } ],
  "topProducts": [ { "productUid": "…", "productName": "Paracetamol 500mg", "quantity": 1640, "net": 2460.00 } ],
  "paymentMix": [ { "method": "CASH", "amount": 18100.00, "count": 140 } ],
  "expiringBatches": [ ExpiringBatch ],
  "recentSales": [ SaleSummary ] }
```

## 14. Integrations — `integrations.manage`

Outbound webhooks deliver domain events to external systems (payments, accounting, ERP, a delivery app, a national reporting system) as **CloudEvents 1.0** JSON.

| Route | |
|---|---|
| `GET /webhooks` | `Webhook[]` |
| `POST /webhooks` | `{ "name", "url", "eventTypes": ["sale.completed"], "active": true }` → `Webhook` with its `secret`, shown once |
| `PUT /webhooks/{uid}` | `{ "name", "url", "eventTypes", "active" }` |
| `DELETE /webhooks/{uid}` | 200 |
| `GET /webhooks/{uid}/deliveries` | page of `{ uid, eventType, eventId, status, responseStatus, attempts, lastError, createdAt, deliveredAt }` |
| `GET /webhooks/event-types` | `["sale.completed","sale.returned","goods.received","stock.adjusted","stock.low","product.changed","tenant.registered"]` |

A delivery is `POST <url>` with these headers:
- `Content-Type: application/cloudevents+json`
- `X-Pharmacy-Signature: sha256=<hex HMAC-SHA256 of the body with the webhook secret>`

The body:

```json
{ "specversion": "1.0", "id": "…", "source": "pharmacy-core/tenants/<tenantUid>", "type": "sale.completed",
  "time": "…", "datacontenttype": "application/json", "data": { } }
```

## 16. Currencies and exchange rates

A tenant has one **base currency** (its `currency`, fixed at registration). Every price, stock cost, total, the ledger, the dashboard and every report is in it. A tenant may also accept foreign currencies (`Settings.acceptedCurrencies`) for:
- payments at the POS;
- purchase orders and goods receipts;
- supplier payments.

| Route | |
|---|---|
| `GET /currencies` (TENANT) | `[{ "code": "TZS", "name": "Tanzanian Shilling", "minorUnits": 2, "symbol": "TSh" }]`: the offered ISO 4217 currencies |
| `GET /fx/rates?quote=` (`finance.view`) | page of `ExchangeRate` `{ uid, base, quote, rate, effectiveOn, source, createdAt }`, newest first. `quote` matches either side of the pair. |
| `GET /fx/rates/latest?base=USD&quote=TZS&on=` (TENANT) | `{ base, quote, rate, effectiveOn, source, stale }`: the rate in force on the day (default today) |
| `POST /fx/rates` (`finance.manage`) | `{ "base": "USD", "quote": "TZS", "rate": 2650, "effectiveOn" }` → `ExchangeRate` (source `MANUAL`; recording the same day again replaces it) |
| `POST /fx/rates/refresh` (`finance.manage`) | `ExchangeRate[]` pulled from the configured provider for the accepted currencies, stored as foreign → base; empty with no provider |

Rules:
1. **What a rate means.** A rate is units of `quote` per 1 `base`. Documents quote theirs as base units per 1 document unit: USD → TZS 2650.
2. **Locked rates.** A sale's payment, a purchase order, a goods receipt and a supplier payment each store the rate they used, so a later rate never changes them.
3. **Rate lookup.** `latest` uses the most recent rate effective on or before the day, or the inverse of the opposite pair.
4. **Missing rates.** With no rate, the API refuses with `422 rate_missing` (field `currency`): record one under Finance → Exchange rates.
5. **Stale rates.** `stale: true` when the rate took effect more than 7 days before the day. Clients show a warning; the API still uses it.
6. **Realised differences.** Settling a foreign-currency payable or receivable at a rate other than the one it was booked at realises an exchange difference, posted to 7000 FX Gains/Losses.
7. **Rounding.** Conversions round **half-even** (banker's rounding) to the target currency's ISO 4217 minor units, so repeated conversions do not drift. Clients format amounts with `Intl.NumberFormat` and the currency, which takes its minor units and rounds half-up for display.
8. **Daily refresh.** A configured provider (`pharmacy.integrations.fx.provider=openexchangerates`, with `base-url` and `api-key`) is refreshed every day at 06:00 UTC for every tenant that accepts a foreign currency.

## 15. Health
`GET /actuator/health` (public) → `{ "status": "UP" }`.

## Files

Stored files and attachments: the pharmacy logo, prescription images, supplier invoice scans. The browser moves the bytes directly to and from storage with short-lived presigned URLs; the API only signs them and keeps the record.

Rules:
- **Types:** PNG, JPEG, WebP and PDF only, or `415 unsupported_type`.
- **Size:** at most 10 MB, or `413 file_too_large`.
- **Owner types:** `TENANT`, `SALE`, `PURCHASE_ORDER`, `GOODS_RECEIPT`, `PRODUCT`, `CUSTOMER`, `SUPPLIER`, `EXPENSE`, `OTHER`.
- **Purposes:** `LOGO`, `PRESCRIPTION`, `INVOICE`, `PHOTO`, `ATTACHMENT` (the default).
- **Permissions:** any signed-in tenant user may attach files to records. Files with `ownerType=TENANT` (the logo) need `settings.manage`.

| Route | |
|---|---|
| `POST /files/uploads` | `{ "name", "contentType", "size", "ownerType", "ownerUid", "purpose" }` → `201 UploadTicket` |
| `PUT <uploadUrl>` | The raw bytes, with the ticket's `headers` (`Content-Type`). Sent by the browser directly, without `Authorization`. Valid 10 minutes. |
| `POST /files/uploads/{uid}/complete` | → `File`. Checks the bytes arrived, then records size and SHA-256. `422 upload_missing` if they did not arrive. |
| `POST /files` | `multipart/form-data` with `file`, `ownerType`, `ownerUid` and `purpose` → `201 File`. A direct upload through the API, for small files and scripts. |
| `GET /files?ownerType=&ownerUid=` | `File[]`, newest first |
| `GET /files/{uid}/url` | `{ "url", "expiresAt" }`: a presigned GET valid 5 minutes, for an `<img src>` or a link |
| `GET /files/{uid}/download` | `302` to the presigned GET |
| `DELETE /files/{uid}` | 200 |

```text
UploadTicket { fileUid, uploadUrl, headers: { "Content-Type": "image/png" }, expiresAt }
File         { uid, name, contentType, size, ownerType, ownerUid, purpose, sha256, createdAt, createdBy }
```

Storage is chosen by `pharmacy.files.storage`:
- `local` (the default): files live in a directory (`pharmacy.files.local.root`). The presigned URLs are API URLs, `/api/v1/files/blob/{key}?exp=&ct=&sig=`. They are public, and authorised only by an HMAC-SHA256 signature over the method, key, expiry and content type, made with `pharmacy.files.signing-secret`. They return `403 invalid_signature` or `403 url_expired` otherwise.
- `s3`: AWS S3, MinIO or Cloudflare R2 (`pharmacy.files.s3.bucket`, `.region`, `.endpoint`, `.access-key`, `.secret-key`). The URLs are S3 presigned URLs. The bucket needs a CORS rule allowing `PUT` and `GET` from the web app's origin.

Object keys are `tenants/{tenantUid}/{yyyy}/{mm}/{fileUid}`, so a tenant's files are easy to export or erase.


## Notifications

Email, SMS, WhatsApp and in-app messages are built from templates (English and Swahili built in, each overridable per tenant). They go out through the configured providers from an outbox with retries.

Without `X-Tenant-Id`, the inbox routes work at platform scope (for example, a root administrator's own messages).

### Inbox — any signed-in user (`AUTHENTICATED`; `X-Tenant-Id` optional)

| Route | |
|---|---|
| `GET /notifications?unreadOnly=&cursor=&limit=` | page of `InAppNotification`, newest first |
| `GET /notifications/unread-count` | `{ "count": 3 }` |
| `POST /notifications/{uid}/read` | `{ "count" }` (the new unread count) · 404 `notification_not_found` |
| `POST /notifications/read-all` | `{ "count": 0 }` |
| `POST /notifications/stream-ticket` | → `{ "ticket": "pst_…" }`, valid 60 seconds, for one stream |
| `GET /notifications/stream?ticket=&tenant=` | Server-Sent Events, `text/event-stream`. Takes a ticket, never the session token (a URL ends up in logs); closes once the session ends. `401 stream_ticket_expired` |

`InAppNotification`:

```json
{ "uid": "…", "title": "Low stock: Paracetamol 500mg", "body": "Paracetamol 500mg is low at Head Office: 18 left (reorder level 200).",
  "link": "/inventory/stock", "severity": "warning", "read": false, "createdAt": "…" }
```

`severity` is `info`, `warning` or `critical`. `link` is an app route, or null.

**The live stream.** The browser's `EventSource` cannot send headers, so this one route accepts `?ticket=` (from `POST /notifications/stream-ticket`, 60 seconds) and `?tenant=<tenantUid>` in place of `Authorization` and `X-Tenant-Id`. The session token itself is never accepted in the URL. It sends these events:

- `ready` on connect, with data `{ "unreadCount": 3, "latest": null }`;
- `notification` per new in-app message, with data `{ "unreadCount": 4, "latest": InAppNotification }`;
- a `: ping` comment every 25 s.

The stream closes after 30 minutes, and `EventSource` reconnects by itself.

### Preferences — the signed-in user, in the tenant (`TENANT`)

| Route | |
|---|---|
| `GET /notification-preferences` | `[{ "category": "stock", "channels": ["IN_APP", "EMAIL", "SMS", "WHATSAPP"] }, …]` |
| `PUT /notification-preferences` | `[{ "category": "stock", "channels": ["IN_APP"] }]` → the full list · 422 `unknown_category` |

| Category | Covers |
|---|---|
| `stock` | low stock, expiry digest |
| `sales` | receipts |
| `reports` | daily summary |
| `customers` | overdue reminders |
| `tenant` | welcome |
| `access` | password reset, sign-in codes, new devices, invitations |

`access` is always on: changes to it are ignored.

A category with no saved choice uses every channel. A message goes out only on the channels that are both requested and chosen, and only when the recipient has an address on that channel.

### Templates and delivery log — `settings.manage` (`TENANT`)

| Route | |
|---|---|
| `GET /notification-templates` | `NotificationTemplate[]`: every key × channel × locale in effect |
| `PUT /notification-templates/{key}/{channel}/{locale}` | `{ "subject", "body" }` → `NotificationTemplate` (`custom: true`) · 422 `unknown_template` |
| `DELETE /notification-templates/{key}/{channel}/{locale}` | back to the built-in wording |
| `POST /notification-templates/preview` | `{ "key", "channel", "locale", "subject", "body" }` (a null `body` previews the wording in effect) → `{ "subject", "text", "html", "sampleParams" }` |
| `GET /notification-deliveries?channel=&status=&cursor=&limit=` | page of `NotificationDelivery` |

```text
NotificationTemplate { key, channel, locale ("en" | "sw"), subject (null for SMS/WhatsApp), body, custom }
NotificationDelivery { uid, templateKey, channel, to, status (QUEUED | SENDING | SENT | FAILED), provider,
                       attempts, lastError, createdAt, sentAt }
```

Channels are `EMAIL`, `SMS`, `WHATSAPP` and `IN_APP`. Template keys, with their placeholders:

| Key | Placeholders |
|---|---|
| `stock.low` | `productName`, `quantity`, `reorderLevel`, `branchName` |
| `stock.expiring` | `count`, `days`, `items` |
| `sale.receipt` | `pharmacyName`, `number`, `total`, `date` |
| `daily.summary` | `pharmacyName`, `sales`, `transactions`, `topProduct` |
| `customer.overdue` | `customerName`, `amount`, `pharmacyName` |
| `access.invitation` | `name`, `inviterName`, `pharmacyName`, `link` |
| `access.password_reset` | `name`, `minutes`, `link` |
| `access.new_sign_in` | `name`, `device`, `ip`, `time` |
| `access.two_factor_code` | `code`, `minutes` |
| `tenant.welcome` | `name`, `pharmacyName`, `link` |

Placeholders are written `{{name}}`. Email values are HTML-escaped.

**What sends what:**

| Event or schedule | Notification | Recipients and channels |
|---|---|---|
| `stock.low` | `stock.low` | users with `inventory.adjust` at that branch; in-app and email; once per product, branch and day |
| `sale.completed` with a customer phone, when `Settings.smsReceipts` is true | `sale.receipt` | the customer, by SMS |
| `tenant.registered` | `tenant.welcome` | the new administrators; email and in-app |
| 07:00 tenant-local time | `stock.expiring` | users with `inventory.adjust` |
| 18:00 tenant-local time | `daily.summary` | users with `reports.view` |

`Settings` gains `smsReceipts` (boolean, default false), readable and writable through `GET/PUT /settings`.

**Providers.** These are configuration on the API, not in this contract. Each channel defaults to `log`, which only logs the message.

| Channel | Providers |
|---|---|
| Email | `smtp` |
| SMS | `africastalking`, `twilio`, `beem` |
| WhatsApp | `twilio` |

A failed send is retried 1, 5, 15 and 60 minutes later, and stays `FAILED` after 5 attempts.

## Receivables (customer accounts and credit sales)

Every amount is in the base currency. An account opens the first time a customer buys on credit or pays on account.

**Entries.** Each entry has a signed `amount` and an `openAmount`:

| Entry | Sign | `openAmount` means |
|---|---|---|
| `CHARGE` (a credit sale), `CREDIT_APPLIED` (account credit spent at the till) | positive (debit) | what is still unpaid |
| `PAYMENT`, `CREDIT_NOTE`, `WRITE_OFF` | negative (credit) | what is still unallocated |

- An unallocated credit is a deposit or an overpayment. The till can spend it as tender `ACCOUNT_CREDIT`.
- `balance` is what the customer owes. A negative balance means the customer holds credit.

**Permissions.** `sales.credit` lets a cashier sell on account; the presets give it to ADMIN, BRANCH_MANAGER and PHARMACIST. `finance.write_off` lets a user write off a bad debt; the presets give it to ADMIN, BRANCH_MANAGER and ACCOUNTANT.

| Route | |
|---|---|
| `GET /receivables/accounts?q=&withBalance=true` | page of `Account`, by name · `finance.view` |
| `GET /receivables/accounts/{customerUid}` | `Account`. Any tenant permission, so the POS can show the credit available. |
| `GET /receivables/accounts/{customerUid}/statement?from=&to=` | `{ account, from, to, openingBalance, entries: page of Entry }`. Entries come in date order; each carries `runningBalance`. · `finance.view` |
| `POST /receivables/payments` | `{ customerUid, amount, method, reference, date, allocateTo: [chargeUid], currency, exchangeRate }` → `Entry` (201) · `finance.manage` |
| `POST /receivables/accounts/{customerUid}/hold` | `{ onHold, reason }` → `Account` · `finance.manage` |
| `POST /receivables/write-offs` | `{ customerUid, amount, reason }` → `Entry` (201) · `finance.write_off` |
| `GET /receivables/ageing?asOf=` | `[{ customerUid, customerName, current, d31to60, d61to90, over90, total }]`, open charges by age since the sale · `finance.view` |
| `GET /receivables/totals` | `{ outstanding, overdue, unallocatedCredit, accountsWithBalance }` · `finance.view` |

`Account`:
```json
{ "customerUid": "…", "customerName": "City Clinic Ltd", "creditLimit": 2000000, "balance": 20000,
  "available": 1980000, "overdue": 0, "oldestOpenDate": "2026-11-03", "onHold": false, "holdReason": null,
  "unallocatedCredit": 0, "termsDays": 30 }
```

`Entry`:
```json
{ "uid": "…", "type": "PAYMENT", "reference": "RCP-000001", "saleUid": null, "amount": -10000.00,
  "openAmount": 0, "date": "2026-11-03", "dueDate": null, "note": "MPESA-QX7Y2", "method": "MOBILE_MONEY",
  "currency": null, "foreignAmount": null, "exchangeRate": null,
  "allocations": [{ "entryUid": "…", "reference": "INV-HQ-000011", "saleUid": "…", "amount": 10000.00 }],
  "runningBalance": 20000.00, "createdAt": "…", "createdBy": "…" }
```

**Payment rules**:
- A payment settles open charges oldest first, or the charges named in `allocateTo`, in that order. Whatever is left stays on the account as credit.
- A payment in a foreign currency records the `currency`, the `foreignAmount` and the `exchangeRate` (base units per 1; null means the day's rate). The account is credited the base value rounded to cents. The sub-cent rounding posts to `7000 FX Gains/Losses`.

**Refusals:**

| Status | Reason |
|---|---|
| 422 | `credit_limit_exceeded`, `account_on_hold`, `insufficient_account_credit`, `write_off_exceeds_open`, `charge_not_open`, `customer_not_found` |
| 403 | `credit_not_permitted` |

**Ledger** (posted after commit, idempotent):

| Event | Dr | Cr |
|---|---|---|
| Payment on account | cash / bank / mobile money | 1100 |
| Write-off | 6096 Bad Debts | 1100 |
| Refund kept on account, when the return was refunded by another method | that method's account | 1100 |

### Paying at the POS — `POST /sales` additions

`CheckoutRequest` gains these fields (all optional):

```json
{ "memberUid": "…", "preauthNumber": "PA-2211", "diagnosisCode": "B54",
  "balanceToAccount": true, "excessToAccount": false }
```

- `PaymentMethod` gains `ACCOUNT_CREDIT`: it spends the customer's deposits, overpayments and credit notes, and maps to account 1100. `CREDIT`, `INSURANCE` and `ACCOUNT_CREDIT` need a named customer, otherwise 422 `customer_required`; a walk-in can never owe. `CREDIT`, `INSURANCE` and `ACCOUNT_CREDIT` are settled in the base currency (422 `base_currency_only`).
- **Partial payment** (`balanceToAccount`, which needs `sales.credit`): whatever the tenders leave unpaid becomes a `CREDIT` payment. The account is checked against its credit limit and hold, then charged, due on the sale date plus the customer's terms.
- **Full credit sale**: send no payments with `balanceToAccount: true`, or send one `CREDIT` payment of the total.
- **Mixed tender**: any combination of cash, card, mobile money, insurance, credit and account credit, in base or accepted foreign currencies. Coverage is checked in the base currency. Change is always given in base cash, and only from cash.
- **Non-cash overpayment**: refused with 422 `overpaid_non_cash`, unless `excessToAccount` is true for a named customer. Then the excess of the last base-currency card or mobile money payment stays on the account as credit, recorded as a payment on account.
- **Insured sale** (`memberUid`):
  - The sale must carry an `INSURANCE` payment of exactly the covered amount. When the till sends none, one is added. A different amount is refused with 422 `insurance_amount_mismatch`.
  - The patient pays the co-pay and the not-covered part with any tender, including on account.
  - A claim is raised in the same transaction.
  - 422 `member_inactive`, `preauth_required`, `member_required`.
- **Returns** (`POST /sales/{uid}/returns`):
  - The insurer's share of the refund (in proportion to the insurance paid) comes off the claim, which is voided if it is a fully returned DRAFT.
  - Every way the sale was paid gets back its share, in proportion: the cash share in cash, a mobile money, card, bank or cheque share by that method, a credit or account-credit share to the customer's account (a credit note), a voucher share back onto the voucher, and the insurer's share off the claim. Shares add up over several returns; once the sale is fully returned each way has had back exactly what it kept. `SaleReturn.refunds` lists them: `[{ "method", "amount", "reference" }]`.
  - `refundMethod`, if sent, must be one of the sale's ways; never `INSURANCE`; `CREDIT`/`ACCOUNT_CREDIT` only when the sale was partly on or from the account (never a walk-in); `VOUCHER` only for a voucher-paid sale. Else `422 refund_method_not_allowed`.
  - Refunds are in the base currency; a foreign-currency refund only for a sale paid one way (`422 foreign_refund_not_allowed`).
  - When a return brings back a line's last units it refunds what is left of the line, to the cent.

`POST /pos/coverage-quote` **[branch]** · `sales.sell` takes `{ memberUid, customerUid, pricePlan, discountPercent, lines: [{ productUid, quantity }], preauthNumber }` and returns `CoverageQuote`, priced exactly as checkout will (plan, branch prices, promotions, quantity breaks). The member must belong to the customer: `422 member_not_of_customer` (also at checkout).

When two tills need the same stock or customer at once and the database gives up waiting, the answer is `409 busy_try_again` with `Retry-After`: send the same request again (with the same idempotency key).

```json
{ "memberUid": "…", "covered": 11520.00, "copay": 1280.00, "notCovered": 0, "patientPays": 1280.00,
  "notes": [], "lines": [{ "productUid": "…", "lineTotal": 11000.00, "covered": 9900.00, "copay": 1100.00,
  "notCovered": 0, "excluded": false }] }
```

## Insurance

Payers, schemes and members need `partners.view` / `partners.manage`. Claims, batches and remittances need `finance.view` / `finance.manage`.

| Route | |
|---|---|
| `GET /insurance/payers` · `POST` · `PUT /{uid}` | `{ code, name, contact, email, paymentTermsDays, requiresPreauth, active }` |
| `GET /insurance/schemes?payerUid=` · `POST` · `PUT /{uid}` | `{ payerUid, name, copayType: NONE\|PERCENT\|FIXED, copayValue, annualLimit, excludedProductTypes: [], excludedProductUids: [], active }` |
| `GET /insurance/members?q=` or `?customerUid=` · `POST` · `PUT /{uid}` | `{ customerUid, schemeUid, memberNumber, principalName, validFrom, validUntil, active }` |
| `GET /insurance/members/current?customerUid=` | the membership valid today (any tenant permission), or 404 `member_not_found` |
| `GET /insurance/claims?payerUid=&status=` · `GET /{uid}` | page of `Claim` |
| `POST /insurance/claims/{uid}/adjudicate` | `{ status (REJECTED to reject all, else derived), lines: [{ productUid, approvedAmount, rejectionReason }], note, rejectedTo: WRITE_OFF\|PATIENT_ACCOUNT }` → `Claim` |
| `GET /insurance/claims/totals` | `{ pendingClaims, pendingAmount, draftClaims, approvedUnpaid }` |
| `GET /insurance/batches?payerUid=` · `POST { payerUid, from, to }` | The payer's DRAFT claims of the period become SUBMITTED in a batch (422 `no_claims_in_period`). |
| `GET /insurance/batches/{uid}/export?format=CSV\|JSON` | `text/csv`, or `application/fhir+json`: a FHIR R4 `Bundle` (type `collection`) of `Claim` resources. Each Claim has claim type `pharmacy`, an ICD-10 diagnosis, the member as `insurance.coverage` and one item per line. |
| `GET /insurance/remittances?payerUid=` · `POST { payerUid, amount, reference, date, perClaim: { claimUid: amount } }` | Spreads the payment over approved claims, oldest first, or as given. Claims become `PAID` when settled. 422 `remittance_exceeds_due`, `remittance_mismatch`. |

**Claim statuses:** `DRAFT` → `SUBMITTED` → `APPROVED` \| `PARTIALLY_APPROVED` \| `REJECTED` → `PAID`, plus `VOIDED`.

**Co-pay rules:**
- Excluded items are not covered at all.
- A `PERCENT` co-pay applies to each covered line. A `FIXED` co-pay is charged per claim and spread over the covered lines.
- The insurer's share is capped at what is left of the member's annual limit (`usedThisYear`); the excess goes to the patient as not covered.

**Ledger:**

| Event | Dr | Cr |
|---|---|---|
| Insured sale | 1110 for the covered share | (posted with the sale) |
| Remittance | 1010 | 1110 |
| Rejected part, `WRITE_OFF` | 6095 Insurance Rejections | 1110 |
| Rejected part, `PATIENT_ACCOUNT` (a charge also lands on the patient's account) | 1100 | 1110 |
| Insurer share of a return | the refund method's account | 1110 |

**Dashboard.** `Dashboard` gains `credit`: `{ receivables, overdue, customerCredit, accountsWithBalance, pendingClaims, pendingClaimsAmount }`.


## 17. Security — `/auth` (the signed-in user's own account; public steps marked)

All `/auth` responses carry `Cache-Control: no-store`. Every API response carries these headers:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Referrer-Policy: no-referrer`
- `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'`

### Sign-in second step (public)
| Route | Body → Response |
|---|---|
| `POST /auth/sign-in/verify` | `{ "challengeToken", "code" }` → `201 Session` |

`code` is a 6-digit TOTP code, or a recovery code such as `k7mq-2xhz`.

A challenge lasts 5 minutes and takes 5 tries, then `401 challenge_expired`. A wrong code gives `401 invalid_code` and counts towards the account lockout. A TOTP code is never accepted twice.

### Two-step verification (TOTP, RFC 6238) — `AUTHENTICATED`
| Route | Body → Response |
|---|---|
| `GET /auth/security` | `SecurityOverview` |
| `POST /auth/2fa/setup` | → `{ "secret": "JBSW…", "otpauthUri": "otpauth://totp/Pharmacy%20Core:…" }` (show as QR); `409 two_factor_already_enabled` |
| `POST /auth/2fa/enable` | `{ "code" }` → `{ "recoveryCodes": [10 codes] }`, shown once; `422 invalid_code`, `422 two_factor_not_started` |
| `POST /auth/2fa/disable` | `{ "code" }` (TOTP or recovery code) → `200` |
| `POST /auth/2fa/recovery-codes` | `{ "code" }` → `{ "recoveryCodes" }`; the old codes stop working |

`SecurityOverview`: `{ "twoFactorEnabled", "twoFactorEnabledAt", "recoveryCodesLeft", "passkeys", "twoFactorRequired" }`.

### Passkeys (WebAuthn, discoverable credentials)
| Route | Scope | Body → Response |
|---|---|---|
| `GET /auth/passkeys` | AUTHENTICATED | `[{ uid, name, transports, createdAt, lastUsedAt }]` |
| `POST /auth/passkeys/register/options` | AUTHENTICATED | → `{ "ceremonyUid", "options": { "publicKey": {…} } }` |
| `POST /auth/passkeys/register` | AUTHENTICATED | `{ "ceremonyUid", "credential", "name" }` → `201 Passkey`; `422 passkey_rejected`, `409 passkey_exists` |
| `DELETE /auth/passkeys/{uid}` | AUTHENTICATED | `200` |
| `POST /auth/passkeys/sign-in/options` | PUBLIC | → `{ "ceremonyUid", "options": { "publicKey": {…} } }` (no email needed) |
| `POST /auth/passkeys/sign-in` | PUBLIC | `{ "ceremonyUid", "credential" }` → `201 Session`; `401 passkey_rejected`, `401 ceremony_expired` |

How the pieces fit:
- **Options.** They are in the WebAuthn JSON form, with base64url binary values. Pass `options.publicKey` to `navigator.credentials.create`/`get`, after decoding `challenge`, `user.id` and the credential ids.
- **Credential.** `credential` is the browser credential serialised: `{ id, rawId, type, response: { clientDataJSON, attestationObject | authenticatorData, signature, userHandle, transports }, clientExtensionResults }`, binary values in base64url.
- **Ceremony lifetime.** A ceremony lasts 2 minutes.
- **Second factor.** A passkey sign-in counts as two-step: it opens a session directly.
- **Configuration:**
  - `pharmacy.access.webauthn.rp-id` (default `localhost`; the web app's domain in production);
  - `pharmacy.access.webauthn.rp-origins` (default `http://localhost:4200,http://localhost:4300`).

### Password reset (public)
| Route | Body → Response |
|---|---|
| `POST /auth/password-reset` | `{ "email" }` → always `202` |
| `POST /auth/password-reset/complete` | `{ "token", "newPassword" }` → `200`; `422 invalid_token`, `422 weak_password` |

The email carries `{pharmacy.access.web-url}/reset-password?token=prt_…`. The link works for 30 minutes and once only. Completing a reset signs the user out everywhere and clears any lockout.

### Invitations
| Route | Scope | Body → Response |
|---|---|---|
| `GET /users/invitations` | TENANT · `users.manage` | `Invitation[]`, newest first |
| `POST /users/invitations` | TENANT · `users.manage` | `{ "name", "email", "roleUid", "allBranches", "branchUids" }` → `201 Invitation`; `409 already_a_member` |
| `DELETE /users/invitations/{uid}` | TENANT · `users.manage` | `200` (revoke) |
| `GET /auth/invitations/{token}` | PUBLIC | `{ email, name, tenantUid, tenantName, roleName, invitedBy, expiresAt, status, existingUser }` |
| `POST /auth/invitations/accept` | PUBLIC | `{ "token", "name", "password" }` → `201 Session`, or `202` challenge; `422 invitation_not_open`, `401 invalid_credentials` |

How invitations work:
- **The email.** It carries `{web-url}/accept-invitation?token=piv_…` and is valid 7 days. Inviting the same email again revokes the earlier invitation.
- **New account.** For a new email, `password` becomes the new account's password.
- **Existing account.** For an existing account, it must be that account's current password; the invitee proves they own it.
- `Invitation`: `{ uid, email, name, roleUid, roleName, allBranches, branchUids, status, invitedBy, expiresAt, createdAt }`. `status` is `PENDING`, `ACCEPTED`, `EXPIRED` or `REVOKED`.

### Sessions — `AUTHENTICATED`
| Route | Body → Response |
|---|---|
| `GET /auth/sessions` | `[{ uid, method, ip, userAgent, createdAt, lastSeenAt, expiresAt, current }]` |
| `DELETE /auth/sessions/{uid}` | `200`: signs that device out |

`method`: `PASSWORD`, `TWO_FACTOR`, `PASSKEY` or `INVITATION`. A sign-in from a device (user agent) the user never used before sends `access.new_sign_in` by email and in-app.

### Configuration
| Property | Default | Meaning |
|---|---|---|
| `pharmacy.access.session-hours` | 12 | Session length |
| `pharmacy.access.require-2fa-for-root` | false | Root administrators must use two-step |
| `pharmacy.access.encryption-key` | local default (warned) | AES-256-GCM key material for TOTP secrets; set in production |
| `pharmacy.access.web-url` | `http://localhost:4300` | Base of the links in emails |
| `pharmacy.access.lockout.max-failures` / `window-minutes` / `lock-minutes` / `ip-max-failures` | 5 / 15 / 15 / 20 | Brute-force protection |
| `pharmacy.edge.auth-requests-per-minute` | 20 | Rate limit per IP on the public `/auth` routes |

## 18. Audit

Every create, update and delete of a record is written to an append-only audit trail, with the changed values before and after.

What it stores:
- **Bookkeeping left out.** Bookkeeping columns and usage counters are not stored.
- **Secrets masked.** Any field named like password, secret, token, hash, recovery, credential, publicKey or otp is stored as `***`.
- **Security events.** These are recorded as well:
  - `SIGN_IN`, `SIGN_IN_FAILED`, `ACCOUNT_LOCKED`, `SIGN_OUT`, `SESSION_REVOKED`;
  - `PASSWORD_CHANGED`, `PASSWORD_RESET_REQUESTED`, `PASSWORD_RESET`;
  - `TWO_FACTOR_ENABLED`, `TWO_FACTOR_DISABLED`, `RECOVERY_CODE_USED`, `RECOVERY_CODES_REGENERATED`;
  - `PASSKEY_REGISTERED`, `PASSKEY_REMOVED`;
  - `INVITATION_ACCEPTED`, `ROLE_PERMISSIONS_CHANGED`;
  - `ROOT_ACTED_IN_TENANT` (once an hour per root administrator and tenant).

| Route | Scope | Response |
|---|---|---|
| `GET /audit?entityType=&entityUid=&actor=&from=&to=&cursor=&limit=` | TENANT · `settings.manage` | page of `AuditEntry` (the tenant's own) |
| `GET /platform/audit?…` (same filters) | PLATFORM · `platform.manage` | page of `AuditEntry` (platform scope: tenants, users, root administrators) |

`AuditEntry`: `{ uid, at, actor, action, entityType, entityUid, tenantUid, branchUid, before, after, correlationId }`.
- `action` is `CREATE`, `UPDATE`, `DELETE` or one of the events above.
- `actor` is the email, `service:<name>` or `anonymous`.
- `actor` matches by "contains"; `from` and `to` are inclusive UTC days.

The trail is partitioned by month, and row-level security shows a tenant only its own rows. The web app must not send `X-Tenant-Id` on `/platform/*` routes; it would scope them to that tenant.

## 19. Approvals and pharmacy controls (Batch 4, owner's rules of 23 Nov 2026)

A second person approves a sensitive action. It is the same for every action:

1. The request is refused `403 approval_required`. The problem's extra members say what needs approving: `action` and `amount` (base currency, or a percent for `DEEP_DISCOUNT` and `DEEP_PROMOTION`).
2. On the same screen a manager types their sign-in and password:

   `POST /approvals` (TENANT, any signed-in member) `{ "action", "amount", "reference"?, "login", "password" }` → `201 { "token": "pap_…", "action", "amount", "approvedBy", "expiresAt" }`.

   The approver holds `approvals.grant` in this pharmacy and branch and is never the requester. Refusals: `approver_is_requester` (422), `not_an_approver` (403), `invalid_credentials` (422), `invalid_action` (422). A wrong password counts towards the approver's lock-out.
3. The same request is sent again with the header `X-Approval: pap_…`. The approval is single use, lasts two minutes, and covers that action for the requester up to its amount. A used, expired, smaller or other person's approval is `403 approval_invalid`.

One request may need two approvals, for example a late return of a damaged controlled medicine. `X-Approval` is a list header (RFC 9110), so the request is sent with `X-Approval: pap_a, pap_b`; each action uses its own. An approval of another action does not stand in: that action is asked for (`approval_required`). Refused requests are not recorded by `Idempotency-Key`, so the retry keeps its key. Approvals are kept in `platform.approvals` and audited as `APPROVAL_GRANTED` and `APPROVAL_USED`. A root administrator needs none.

| Action | When | Amount |
|---|---|---|
| `REFUND` | a return's refund above the limit | refund total |
| `LATE_RETURN` | always, after the return window | refund total |
| `VOUCHER` | a voucher above the limit | value |
| `DEEP_DISCOUNT` | always, a till discount above `maxDiscountPercent` | percent |
| `DEEP_PROMOTION` | always, saving an active promotion deeper than `maxDiscountPercent` | percent |
| `SHIFT_VARIANCE` | closing a shift whose cash is off by more than the limit | the difference |
| `STOCK_WRITE_OFF` | an adjustment, destruction or stock-take loss above the limit | value at cost |
| `CONTROLLED_WRITE_OFF` | always, any write-off, destruction, count loss or transfer shortfall of a controlled medicine | value at cost |
| `LATE_OPENING_STOCK` | always, opening stock in a branch that has sold, received goods or received a transfer | value at cost |
| `SCHEDULE_CHANGE` | always, changing a product's dispensing class (form or import) | 0 |
| `TRANSFER_SHORTFALL` | units short on a received transfer, above the limit | value at cost |
| `PERIOD_UNLOCK` | always, moving the books' lock to an earlier date or removing it | 0 |
| `CREDIT_WRITE_OFF` | always, writing off a customer's debt | the write-off |

### Settings
`Settings` gains:
- `approvalLimit`: TZS 50,000 by default, `invalid_amount` when negative.
- `returnWindowDays`: 7, between 0 and 365, else `invalid_return_window`.
- `requireOpenShift`: on for new pharmacies (`pharmacy.tenancy.require-open-shift-by-default`), off for pharmacies that existed before.

With `requireOpenShift`, a sale, a return and a voucher need the person's own open shift at the branch: `409 shift_required`.

### Returns (`POST /sales/{uid}/returns`)
- Each line takes `disposition`: `SHELF`, `QUARANTINE` (held aside) or `DAMAGED` (written off at cost).
- A prescription or controlled line (by the class stored on the sale line, `lines[].schedule`) defaults to `QUARANTINE`, and `SHELF` is refused with `422 restock_not_allowed`.
- Return lines show `uid`, `disposition`, `heldQuantity`, `releasedOutcome`, `releasedAt` and `releasedBy`.
- `GET /sale-returns/held` (`sales.view`) lists the branch's returns with held units.
- `POST /sales/returns/{returnUid}/lines/{lineUid}/release` (`sales.refund`) `{ "outcome": "SHELF" | "DAMAGED" }`. A prescription or controlled line can only be written off. Refusals: `nothing_held`, `invalid_outcome`.

### Sales and promotions
- Sale lines store `schedule`, `promotionUid` and `promotionName`; a sale stores `pricePlanSource` (`CUSTOMER`, `CHOSEN` or null).
- `DELETE /promotions/{uid}` switches the promotion off and returns it: promotions are never deleted.

### Stock
- **Adjustments and destructions.** A reduction needs a `note`: `422 note_required`. When a manager approves, they become the destruction's witness; the typed witness is then optional. A destruction shows `witnessApproved`, `approvedByUid` and `typedWitnessName`.
- **Stock-takes are blind.** While a count is `OPEN`, `expected`, `variance`, `varianceValue` and `lossValue` are null and `blind` is true. `GET /stock-counts/{uid}/review` (`inventory.adjust`) shows them before posting.
- **Opening stock.** Into an existing batch at a different unit cost it is `422 batch_cost_differs` (field `unitCost`, member `batchUnitCost`).
- **Repack, break bulk and compound** follow the pack link or formula: `no_recipe`, `recipe_mismatch`, `inputs_below_formula`. A prescription or controlled input cannot become a product of a lower class: `schedule_downgrade`.
- **Controlled-drugs register.** `GET /controlled-drugs` lists every product that was ever controlled, with `schedule` and `controlledNow`.
- **Transfers.**
  - The person who sent a transfer cannot receive it: `422 receiver_is_sender`.
  - `POST /transfers/{uid}/receive` takes `{ "note", "lines": [{ "batchUid" | "productUid", "receivedQuantity", "serialNumbers"? }] }`. Lines not named arrived in full. Units short need a note and are a loss of the sending branch.
  - `GET /transfers?overdue=true&days=7` lists transfers in transit longer than `days` (1 to 365, else `invalid_days`).
  - `TransferView` gains `dispatchedBy`, `dispatchedByUid`, `receivedBy` and `daysInTransit`. Lines gain `receivedQuantity`.

### Roles, books and credit
- **Role presets.** New pharmacies get split presets: Pharmacist has no `sales.refund`, `inventory.adjust`, `catalogue.manage` or `partners.credit_limit`. New permissions: `approvals.grant`, `finance.period_lock` and `partners.credit_limit`. Pharmacies that existed keep their roles; the backfills keep what each role could do.
- **Recommended roles** (`users.manage`):
  - `GET /roles/recommendations` → `{ presetsVersion, showNotice, dismissed, decision, decidedAt, decidedBy, roles: [{ roleUid, code, name, add: [], remove: [] }] }`.
  - `POST /roles/recommendations/apply` sets the built-in roles to the presets (`409 last_admin`, `403 permission_not_held`).
  - `POST /roles/recommendations/dismiss` keeps them.
- **Period lock.** `PUT /finance/period-lock` needs `finance.period_lock`; moving it back needs `PERIOD_UNLOCK`.
- **Credit limits.** Setting or changing a customer's credit limit needs `partners.credit_limit`: `403 credit_limit_not_allowed`. A customer `PUT` without `creditLimit` keeps the current limit. `PUT /customers/{uid}/credit-limit` `{ "creditLimit" }` changes only the limit.
- **Supplier returns.** A supplier return without `goodsReceiptUid` needs `purchasing.manage`: `403 supplier_return_needs_receipt`.

## 20. Integrations, bulk data and isolation (Batch 5, 23 Nov 2026)

### Email server (SMTP) providers
- **Host:** must be a public mail server. The rules are the same as for webhook targets, so no private, loopback, link-local or `*.internal` address: `422 smtp_host_not_allowed` (field `settings.host`).
- **Port:** 25, 465, 587, 2525 or 2587, otherwise `422 smtp_port_not_allowed` (field `settings.port`). Port 465 uses implicit TLS; every other port requires STARTTLS, with the server name checked. The `tls` option is gone.
- **Changing the host or port** drops the saved secrets: send `secrets.password` again with the change.
- **When the rules apply:** on save, on "Send test" and before every send.
- **Dev and tests only:** `pharmacy.notifications.smtp.allowed-hosts` skips the host and port rules.

### SMS allowance
- **What is capped:** the plan's `sms_per_month`, but only for SMS sent through the platform's account. A pharmacy with its own active SMS provider is not capped.
- **How it counts:** SENT messages in the pharmacy's calendar month, in its own time zone.
- **Over the cap:**
  - The message is FAILED with `reason: "sms_cap_reached"`, and the admins get one inbox notice a month.
  - "Send test" answers `422 sms_cap_reached`.
- **Delivery rows** carry a new field, `reason`.

### Webhooks (Standard Webhooks)
- **Headers:**
  - `webhook-id`: the event id, the same on every retry.
  - `webhook-timestamp`: Unix seconds.
  - `webhook-signature`: `v1,<base64 HMAC-SHA256("id.timestamp.body")>`, keyed with the base64 part of `whsec_…`.
  - Receivers should reject a timestamp more than 5 minutes off.
  - `X-Pharmacy-Signature` is deprecated and will be removed next release.
- **The secret** is sealed at rest. It is shown once, on create and on rotation.
- **Rotation:** `POST /webhooks/{uid}/rotate-secret` returns `{…, secret, previousSecretUntil}`. For 24 hours the signature header carries two space-separated `v1,` values, new and old.
- **Limit:** at most 10 webhooks per pharmacy, `422 webhook_limit`.

### Payments
- **The SIMULATOR adapter** exists only in the `sim`, `local` and `test` profiles. Elsewhere it is `422 adapter_not_available`.
- **Push payments:** a push request used for another method or branch is `422 payment_mismatch`.

### Imports
- **Workbooks** are checked before reading. Refusals (all 422):
  - `file_too_many_entries`: more than 1,000 parts.
  - `file_too_large_uncompressed`: more than 50 MB unpacked.
  - `file_too_compressed`: a part expands more than 100:1.
  - `file_not_allowed`: encrypted, or an unknown packing.
- **Workbooks and CSV:**
  - `cell_too_long`: a cell longer than 32,767 characters.
  - `too_many_columns`: data past column 200.
  - `too_many_rows`: more than 5,000 rows (as before).

### The pharmacy's full export (background job)
- **Starting:** `POST /data-export` (`settings.manage`, allowed while unpaid or closed) answers `202 { uid, status, requestedAt, requestedBy }`. One export runs at a time: `409 export_running`, with the running job's `uid` in the problem.
- **Polling:** `GET /data-export/{uid}` gives `status`: `PENDING`, `RUNNING`, `READY`, `FAILED` or `EXPIRED`. When `READY` it also gives:
  - `downloadUrl`: a signed link, opened without auth headers, valid 15 minutes. GET the status again for a fresh one.
  - `downloadExpiresAt`;
  - `expiresAt`: the file is kept 7 days;
  - `sizeBytes`.

  An unknown uid is `404 export_not_found`.
- **How it is built:** in short paged transactions, so it is not one snapshot; the manifest says so.
- **Columns:** they are explicit. Root's internal notes are left out, and former staff appear with the details the pharmacy kept (`former_*`).
- **The old routes:** `GET /data-export` and root's `GET /platform/tenants/{uid}/export` still work, and also take the one-at-a-time slot.

### Spreadsheet exports
Text cells starting with `=`, `+`, `-`, `@`, tab or carriage return get a leading `'` (OWASP CSV injection). Numbers stay numeric.

### Retention, legal hold and erasure
- **Legal hold:** the nightly purge skips held pharmacies. An audit month is kept while any held pharmacy has entries in it.
- **Date limits:** retention cut-offs are never later than `now()`, and audit entries are kept at least 2 years (`audit.minimum_retention()`).
- **`erase_tenant`:** checks the cooling-off against `now()`. It also removes the pharmacy's event publications. File deletes that fail are retried nightly.

### People, roles and branches
- **Users cap:** counts active members plus pending invitations. It is checked on invite, add, managed login, reactivation and acceptance. Refusal: `422 plan_limit_reached`.
- **`PUT /users/{membershipUid}`:** a missing `active` keeps the member as they are.
- **A former member** is shown by the details this pharmacy kept, both in the cashier report and in the account locks list (active members only).
- **Roles:**
  - `GET /roles` lists this pharmacy's roles only.
  - Writes to roles, memberships, role permissions and membership branches must belong to the current pharmacy (row-level security WITH CHECK).
- **Branches:** a branch named in the body must be one the caller may use: `403 branch_not_allowed`. This covers branch prices, and recalls when the batch is held at other branches.
- **New permission `data.export`** ("Export people's data in bulk"):
  - Needed, together with the view permission, for `GET /customers/export` and the controlled-drugs register export.
  - Presets: Administrator, Branch Manager and Accountant.
  - Cashiers, pharmacists and store keepers no longer bulk-export customers, a privacy fix (PDPA).
- **`GET /receivables/accounts/{uid}`** needs `sales.sell` or `finance.view`.
- **`GET /insurance/members/current`** needs `partners.view`.
- **`access.session_pharmacies`** is reached only through SECURITY DEFINER functions.

## 21. Operations (Batch 6, 23 Nov 2026)

All root only (`platform.manage`, PLATFORM scope).

| Route | Response |
|---|---|
| `GET /platform/jobs` | Per scheduled job: `lastRun`, `lastSucceededAt`, `lastFailure` (with `failures: [{tenantUid, error}]`), `lastSkippedAt`, `failedLast24Hours` |
| `GET /platform/jobs/{job}/runs` | The job's recent runs: instance, start, end, outcome `OK`/`FAILED`/`SKIPPED`, message, failures |
| `GET /platform/keys` | The keyring: active key per purpose (`totp`, `providers`, `webhooks`), counts per key id per sealed column, `toReseal`, `unknownKeys`, `removable` per key, the last re-seal run |
| `POST /platform/keys/reseal` | `202`; re-seals every value under the active keys in batches, on one instance. `409 reseal_running` while one runs |

Probes on the API port: `GET /livez`, `GET /readyz`. Metrics: `/actuator/prometheus` on the
management port only.

`POST /files/uploads/{uid}/complete` works again for browser uploads (it answered 404 since
Batch 1: the permission check could not see a pending upload).

## 22. Performance changes (Batch 7, 23 Nov 2026)

**Lists and exports**
- Lists ordered by uid count at most 10,001 rows. Above that, `total` is `10000` and `totalCapped: true`; on an exact total it is `false`, and `null` when there is no total. A first page that holds the whole list, and a short last page, are not counted.
- Exports read in keyset pages of 500.

**Load limits**
- `503 server_busy` with `Retry-After: 2` when an instance has 100 requests in progress. It is refused before any work.
- `503 database_busy` (`Retry-After`) for a statement over 30 s, or no free connection within 10 s.
- A lock wait over 5 s answers `409 busy_try_again`.

**Sales**
- Payments are checked before the sale's transaction. A push request still pending is `422 payment_declined`.
- The tax receipt is issued after commit. `SaleView.fiscalDue` is true until `fiscalReference` is set; with no fiscal provider it is false.

**Price lists**
- `GET /price-lists` returns plans with `items: null` and `itemCount`.
- `GET /price-lists/{uid}` returns one plan with its items.
- `PUT /price-lists/{uid}` with `items` null or `[]` keeps the plan's items.

**Customer statements.** `GET /receivables/accounts/{uid}/statement`:
- Without `from`, it covers the 90 days up to `to`; `to` defaults to the pharmacy's today.
- `openingBalance` is the balance brought forward.
- It pages with cursor and limit (at most 100 per page).
- `from` after `to` is `422 invalid_period`.

**Reports**
- `/reports/sales`, `/products`, `/profit-loss`, `/payments`, `/vat` and `/cashiers` refuse a period over 366 days: `422 period_too_long`.
- `invalid_range` now means only that `from` is after `to`.

**Finance**
- New: `GET /finance/reconciliation/balances?days=7` (`finance.view`) returns `[{accountCode, day, branchUid, journalDebit, journalCredit, readModelDebit, readModelCredit}]`, the days where the daily balances disagree with the journal. An empty list means they agree.
- Journal numbers are now `JV-<year>-000001` per pharmacy and year, without gaps. Old numbers stay.
- Balances, the trial balance, the balance sheet and VAT read the daily balances. The stock list reads the per-product stock summary.

## 23. The till and refusals (Batch 8, 24 Nov 2026)

**POST /sales idempotency**
- **Same key, same request:** returns the original sale. The request's fingerprint ignores key order and spacing.
- **Same key after a sale, different request:** `422 idempotency_key_reused`, with members `saleUid` and `saleNumber`.
- **Refused first attempt:** nothing is recorded, so the key stays free.
- **Concurrent duplicates:** they wait for the first, then get the same answer.
- **Lookup:** `GET /sales?idempotencyKey=<key>` returns that sale, or an empty page.

**Push payments**
- `POST /pos/payment-requests` takes `saleKey`.
- At checkout, a request bound to another sale's key is `422 payment_mismatch`. A request without a key is accepted.

**Products by uid:** `GET /pos/products?uids=a,b,c` (`sales.sell`), at most 100. Unknown products are left out; more than 100 is `422 too_many_uids` (`max`).

**Problem members:** every refusal, including those raised at the edge, carries its extra members. The main ones:

| Reason | Members |
|---|---|
| `insufficient_stock` | `product`, `available`, `requested` (or `batch`) |
| `payment_short` | `paid`, `total` |
| `credit_limit_exceeded` | `limit`, `customer`, `available`, `currency` |
| `plan_limit_reached` | `limit`, `plan`, `resource`, `used` |
| `future_date` | `date`, `today` |
| `session_idle` | `idleMinutes` |

Amounts are JSON numbers; dates are ISO.

**Suspension**
- The membership (sign-in and `/auth/me`) carries `suspensionKind` (UNPAID, DISPUTE, ABUSE, OTHER; null unless suspended) and `legalHold`.
- `tenant_suspended`, `tenant_suspended_read_only` and `tenant_closed` carry the same members.

**Web**
- The till locks the cart after an uncertain checkout and resends byte-identical requests.
- Parked sales hold only uids, expire after 12 hours, and survive sign-out.
- Every refusal reason has an English and Swahili line, with values.

