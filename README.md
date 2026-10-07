# SteadFast Courier API Integration Guide

> Developer-focused reference for integrating SteadFast Courier with an e-commerce / OMS application.
>
> **Primary reference:** https://documenter.getpostman.com/view/26211192/2sAYJ1mhpk
>
> **Important:** This document is an integration reference, not a replacement for the official SteadFast documentation. API credentials, endpoint availability, request limits, response schemas, and webhook behavior should always be verified against the current SteadFast account/documentation before production deployment.

---

## 1. Overview

This document defines the integration structure for using SteadFast Courier as a delivery provider inside an e-commerce, ERP, or Order Management System (OMS).

Typical integration capabilities include:

- Create a single courier order
- Create multiple/bulk courier orders
- Track an order
- Check delivery status
- Search by invoice
- Search by consignment ID
- Search by tracking code
- Check merchant/account balance
- Create and manage return requests
- Read payment information
- Retrieve police-station information where supported
- Receive delivery/tracking updates through webhook mechanisms where enabled

---

## 2. Security Rules

### Never commit credentials

Do **not** place real credentials in:

- GitHub repositories
- frontend JavaScript
- Next.js client components
- public environment variables
- screenshots
- README examples
- database seed files

Use server-side environment variables.

Example:

```env
STEADFAST_API_KEY=your_api_key
STEADFAST_API_SECRET=your_api_secret
```

The exact credential names should be adapted to the application's environment convention.

### Recommended architecture

```text
Customer
   |
   v
Storefront
   |
   v
Application Backend / API
   |
   +----> SteadFast API
   |
   +----> Database
   |
   +----> Webhook Endpoint
              ^
              |
        SteadFast Events
```

The browser should never directly call protected courier endpoints.

---

## 3. Configuration

Recommended application configuration:

```env
STEADFAST_API_KEY=
STEADFAST_API_SECRET=
STEADFAST_BASE_URL=
STEADFAST_WEBHOOK_SECRET=
```

Only define variables that are actually required by the current SteadFast integration.

### Production checklist

- [ ] API credentials are stored server-side
- [ ] `.env` is included in `.gitignore`
- [ ] Production secrets are configured on the server
- [ ] HTTPS is enabled
- [ ] Webhook endpoint is protected
- [ ] Webhook processing is idempotent
- [ ] API errors are logged without exposing secrets
- [ ] Request/response payloads containing customer PII are handled securely

---

## 4. Core Order Data Model

A normalized OMS order should keep its own internal order ID and store SteadFast identifiers separately.

Recommended fields:

```text
internal_order_id
merchant_invoice
courier_provider
courier_consignment_id
courier_tracking_code
courier_status
recipient_name
recipient_phone
recipient_address
cod_amount
delivery_type
courier_created_at
courier_updated_at
```

Example normalized object:

```json
{
  "order_id": "ORD-10001",
  "invoice": "INV-10001",
  "courier": "steadfast",
  "consignment_id": null,
  "tracking_code": null,
  "status": "pending",
  "recipient": {
    "name": "John Doe",
    "phone": "01712345678",
    "address": "Dhaka, Bangladesh"
  },
  "cod_amount": 1500
}
```

---

# 5. Create Single Order

Create a courier consignment from the OMS.

### Conceptual request

```http
POST <STEADFAST_CREATE_ORDER_ENDPOINT>
Authorization: Bearer <API_SECRET>
Content-Type: application/json
```

### Request body

A commonly documented SteadFast order structure contains fields such as:

```json
{
  "invoice": "INV-10001",
  "recipient_name": "John Doe",
  "recipient_phone": "01712345678",
  "recipient_address": "House 10, Road 5, Dhaka",
  "cod_amount": 1500,
  "note": "Please handle with care",
  "alternative_phone": "01812345678",
  "recipient_email": "customer@example.com",
  "item_description": "Women's Clothing",
  "total_lot": 1,
  "delivery_type": 0
}
```

### Important fields

| Field | Purpose |
|---|---|
| `invoice` | Merchant's unique order/invoice reference |
| `recipient_name` | Customer name |
| `recipient_phone` | Customer phone number |
| `recipient_address` | Delivery address |
| `cod_amount` | Cash collection amount |
| `note` | Delivery note/instruction |
| `alternative_phone` | Optional secondary phone |
| `recipient_email` | Optional customer email |
| `item_description` | Parcel/product description |
| `total_lot` | Number of lots/parcels where supported |
| `delivery_type` | Delivery mode |

> Verify the exact required/optional fields and allowed values against the current official SteadFast API documentation before implementation.

---

## 6. Create Bulk Orders

Bulk order creation is useful when an OMS needs to process multiple approved orders together.

Conceptual payload:

```json
{
  "orders": [
    {
      "invoice": "INV-10001",
      "recipient_name": "John Doe",
      "recipient_phone": "01712345678",
      "recipient_address": "Dhaka",
      "cod_amount": 1500
    },
    {
      "invoice": "INV-10002",
      "recipient_name": "Jane Doe",
      "recipient_phone": "01812345678",
      "recipient_address": "Chattogram",
      "cod_amount": 2200
    }
  ]
}
```

### OMS recommendation

For bulk processing:

1. Select approved orders.
2. Validate every order locally.
3. Generate/verify unique invoices.
4. Send the batch.
5. Store each returned courier identifier.
6. Mark failed orders individually.
7. Retry only failed items when appropriate.
8. Do not duplicate successful consignments.

---

# 7. Order Status / Tracking

An OMS should support multiple lookup strategies.

Common identifiers:

```text
consignment_id
invoice
tracking_code
```

### By Consignment ID

Conceptual request:

```http
GET <STEADFAST_STATUS_BY_CONSIGNMENT_ENDPOINT>
```

Example identifier:

```text
consignment_id=123456
```

### By Invoice

```http
GET <STEADFAST_STATUS_BY_INVOICE_ENDPOINT>
```

Example:

```text
invoice=INV-10001
```

### By Tracking Code

```http
GET <STEADFAST_STATUS_BY_TRACKING_ENDPOINT>
```

Example:

```text
tracking_code=TRACK123
```

### Recommended OMS lookup order

```text
Internal Order ID
       |
       v
Courier Consignment ID
       |
       +---- if unavailable ----> Invoice
       |
       +---- if unavailable ----> Tracking Code
```

---

# 8. Status Synchronization

Do not treat the storefront order status and courier status as the same database field.

Recommended structure:

```text
OMS status
Courier status
Courier raw status
Last courier sync
```

Example:

```json
{
  "oms_status": "shipped",
  "courier_status": "in_transit",
  "courier_raw_status": "in_transit",
  "last_synced_at": "2026-10-07T10:00:00+06:00"
}
```

### Suggested synchronization flow

```text
Order Approved
      |
      v
Create SteadFast Consignment
      |
      v
Save Consignment ID
      |
      v
Courier Processing
      |
      +---- Webhook Update
      |
      +---- Scheduled Status Sync
      |
      v
Update OMS
      |
      v
Update Customer-facing Status
```

---

# 9. Account Balance

Where supported, the integration can retrieve the merchant's SteadFast account balance.

Conceptual method:

```http
GET <STEADFAST_BALANCE_ENDPOINT>
```

Application use cases:

- Dashboard balance widget
- Low-balance warning
- Pre-dispatch validation
- Finance dashboard
- Courier account monitoring

Example normalized response:

```json
{
  "provider": "steadfast",
  "balance": 12500
}
```

> The normalized example above is an application-level structure. It should not be interpreted as the official SteadFast response schema.

---

# 10. Return Requests

Return functionality should be connected to the OMS return/exchange workflow.

Conceptual request:

```json
{
  "consignment_id": 10000042,
  "reason": "Customer requested return"
}
```

Recommended internal workflow:

```text
Customer Return Request
        |
        v
OMS Return Created
        |
        v
Validate Courier Consignment
        |
        v
Create Courier Return Request
        |
        v
Save Return Reference
        |
        v
Track Return Status
```

### Return record

Recommended fields:

```text
return_id
order_id
consignment_id
reason
courier_return_id
courier_status
requested_at
completed_at
```

---

# 11. Payments

Where supported by the API, courier payment information can be used for reconciliation.

Recommended OMS payment reconciliation fields:

```text
payment_id
courier_provider
payment_date
amount
reference
consignments
reconciled
reconciled_at
```

Suggested flow:

```text
SteadFast Payment Data
        |
        v
Import / Sync
        |
        v
Match Consignments
        |
        v
Calculate Expected COD
        |
        v
Compare Received Amount
        |
        v
Mark Reconciled / Discrepancy
```

---

# 12. Police Station / Location Data

If the current SteadFast API account exposes police-station/location data, the OMS can use it for:

- Address assistance
- Delivery-area validation
- Admin address selection
- Customer address normalization
- Reporting

Do not hardcode location data if an official API endpoint is available and the data is expected to change.

---

# 13. Webhook Architecture

Webhook support should be implemented as a server-to-server endpoint.

Example application endpoint:

```http
POST /api/webhooks/steadfast
```

Recommended flow:

```text
SteadFast
   |
   | POST webhook
   v
Webhook Controller
   |
   +--> Verify authenticity
   |
   +--> Validate payload
   |
   +--> Identify order
   |
   +--> Store raw event
   |
   +--> Check idempotency
   |
   +--> Update order
   |
   +--> Queue notifications
   |
   v
HTTP 200
```

### Important

Webhook processing must be **idempotent**.

The same webhook may be delivered more than once. Never create a second shipment, second order, or duplicate financial transaction merely because an event was received twice.

Recommended table:

```text
courier_webhook_events
```

Fields:

```text
id
provider
event_id
event_type
payload_hash
payload
processed
processed_at
created_at
```

Recommended unique constraint:

```text
(provider, event_id)
```

If the provider does not supply a unique event ID, use a carefully designed deterministic idempotency key based on the event payload and relevant identifiers.

---

# 14. Queue-Based Processing

For production OMS systems, webhook and courier synchronization should preferably use a queue.

```text
Webhook
   |
   v
Validate
   |
   v
Store Event
   |
   v
Queue
   |
   v
Worker
   |
   +--> Update Order
   +--> Update Courier Status
   +--> Send Telegram
   +--> Send Customer Notification
   +--> Write Audit Log
```

Recommended retry strategy:

```text
Attempt 1
   |
   v
Attempt 2
   |
   v
Attempt 3
   |
   v
Dead Letter / Failed Jobs
```

Use exponential backoff for transient failures.

---

# 15. Error Handling

Courier integrations should distinguish between:

### Validation errors

Examples:

- Missing phone number
- Missing recipient address
- Invalid COD amount
- Invalid invoice
- Missing required courier data

Action:

```text
Do not retry automatically.
Fix the order data first.
```

### Authentication errors

Examples:

- Invalid API key
- Invalid API secret
- Expired/invalid credentials

Action:

```text
Stop repeated retries.
Alert administrator.
```

### Network errors

Examples:

- Timeout
- DNS failure
- Temporary connection failure
- 5xx server error

Action:

```text
Retry with backoff.
```

### Business errors

Examples:

- Courier rejected the order
- Invalid delivery configuration
- Unsupported destination

Action:

```text
Store error.
Show actionable message.
Do not blindly retry.
```

---

# 16. Logging

Never log:

```text
API secret
API key
Authorization token
Customer password
Payment credentials
```

Safe log example:

```json
{
  "provider": "steadfast",
  "order_id": "ORD-10001",
  "invoice": "INV-10001",
  "action": "create_order",
  "status": "success",
  "duration_ms": 420
}
```

For debugging, store a sanitized version of request/response payloads where permitted.

---

# 17. Database Design

A production OMS should not put all courier information directly into the main orders table.

Recommended:

### `orders`

```text
id
invoice
customer_id
status
grand_total
created_at
updated_at
```

### `shipments`

```text
id
order_id
courier
consignment_id
tracking_code
courier_status
delivery_type
cod_amount
created_at
updated_at
```

### `courier_events`

```text
id
shipment_id
provider
event_type
external_event_id
payload
processed
created_at
```

### `courier_returns`

```text
id
shipment_id
reason
external_return_id
status
created_at
updated_at
```

---

# 18. Laravel Service Architecture

Recommended structure:

```text
app/
├── Services/
│   └── Courier/
│       └── SteadFast/
│           ├── SteadFastClient.php
│           ├── SteadFastOrderService.php
│           ├── SteadFastTrackingService.php
│           ├── SteadFastReturnService.php
│           └── SteadFastWebhookService.php
│
├── Jobs/
│   ├── CreateSteadFastShipment.php
│   ├── SyncSteadFastStatus.php
│   └── ProcessSteadFastWebhook.php
│
└── Http/
    └── Controllers/
        └── Webhooks/
            └── SteadFastWebhookController.php
```

---

# 19. Next.js / Node.js Architecture

For a Next.js application:

```text
src/
├── lib/
│   └── courier/
│       └── steadfast/
│           ├── client.ts
│           ├── orders.ts
│           ├── tracking.ts
│           ├── returns.ts
│           └── webhook.ts
│
├── app/
│   └── api/
│       └── webhooks/
│           └── steadfast/
│               └── route.ts
```

Keep all secret-dependent operations server-side.

---

# 20. Example Server-side Client

```ts
const apiKey = process.env.STEADFAST_API_KEY;
const apiSecret = process.env.STEADFAST_API_SECRET;
const baseUrl = process.env.STEADFAST_BASE_URL;

if (!apiKey || !apiSecret || !baseUrl) {
  throw new Error("SteadFast configuration is missing.");
}
```

Never expose these through:

```env
NEXT_PUBLIC_STEADFAST_API_KEY
NEXT_PUBLIC_STEADFAST_API_SECRET
```

---

# 21. Create Order — Application Example

```ts
export async function createSteadFastOrder(payload: unknown) {
  const response = await fetch(
    `${process.env.STEADFAST_BASE_URL}/create-order`,
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Api-Key": process.env.STEADFAST_API_KEY!,
        "Secret-Key": process.env.STEADFAST_API_SECRET!,
      },
      body: JSON.stringify(payload),
    }
  );

  if (!response.ok) {
    throw new Error(`SteadFast request failed: ${response.status}`);
  }

  return response.json();
}
```

> **Important:** The exact authentication header names and endpoint path must be taken from the current official SteadFast documentation/account configuration. Do not copy this example into production unchanged.

---

# 22. Idempotency Strategy

Before creating a courier order:

```text
1. Find OMS order
2. Check shipment record
3. If courier consignment already exists:
      return existing shipment
4. Otherwise:
      create courier order
5. Save courier identifiers
6. Commit transaction
```

Pseudo-code:

```ts
const existingShipment = await findShipment(order.id);

if (existingShipment?.consignmentId) {
  return existingShipment;
}

const courierOrder = await createSteadFastOrder(order);

await saveShipment({
  orderId: order.id,
  consignmentId: courierOrder.consignment_id,
  trackingCode: courierOrder.tracking_code,
});

return courierOrder;
```

This prevents accidental duplicate consignments.

---

# 23. OMS Courier Workflow

Recommended production workflow:

```text
NEW ORDER
   |
   v
APPROVED
   |
   v
COURIER QUEUE
   |
   v
STEADFAST CREATE
   |
   +---- Failed ---> RETRY / MANUAL REVIEW
   |
   v
SHIPMENT CREATED
   |
   v
PICKED UP
   |
   v
IN TRANSIT
   |
   +---- Delivered
   |
   +---- Cancelled
   |
   +---- Returned
   |
   v
FINAL STATUS
```

---

# 24. Admin Dashboard

Recommended courier dashboard widgets:

### Courier Account

- Current balance
- Total shipments
- Pending shipments
- Delivered
- Cancelled
- Returned
- Delivery success rate
- COD collected
- COD pending
- Courier charges

### Shipment table

```text
Order
Customer
Phone
Courier
Consignment ID
Tracking Code
COD
Courier Status
OMS Status
Created
Updated
Actions
```

Actions:

```text
Track
View Details
Print Invoice
Print Courier Label
Sync Status
Request Return
View Timeline
```

---

# 25. Shipment Timeline

Store every important courier event.

Example:

```text
07 Oct 2026 10:10
Shipment created

07 Oct 2026 12:30
Courier picked up parcel

08 Oct 2026 09:20
Parcel in transit

09 Oct 2026 15:45
Delivered
```

This is much better than overwriting a single status field.

---

# 26. Security Checklist

- [ ] HTTPS only
- [ ] Secrets server-side
- [ ] No secrets in Git
- [ ] No secrets in frontend bundle
- [ ] Webhook authenticity verification
- [ ] Idempotent webhook processing
- [ ] Rate limiting
- [ ] Request validation
- [ ] Audit logging
- [ ] PII-aware logging
- [ ] Database backups
- [ ] Queue retry limits
- [ ] Failed-job monitoring
- [ ] Admin-only courier configuration
- [ ] Separate development and production credentials

---

# 27. GitHub Repository Checklist

Recommended files:

```text
docs/
└── STEADFAST_API.md

src/
└── services/
    └── steadfast/

.env.example
.gitignore
README.md
```

`.env.example`:

```env
STEADFAST_BASE_URL=
STEADFAST_API_KEY=
STEADFAST_API_SECRET=
STEADFAST_WEBHOOK_SECRET=
```

Never commit:

```text
.env
.env.local
.env.production
real API keys
real API secrets
customer exports
production database dumps
```

---

# 28. Testing Strategy

Before production:

### Unit tests

- Order payload validation
- COD validation
- Invoice uniqueness
- Status mapping
- Webhook signature/authentication validation
- Idempotency

### Integration tests

- Create order
- Fetch status
- Invoice lookup
- Tracking lookup
- Balance lookup
- Return request

### Failure tests

- Invalid credentials
- Timeout
- 4xx response
- 5xx response
- Duplicate webhook
- Duplicate order submission
- Invalid payload
- Missing shipment

---

# 29. Production Deployment Checklist

```text
[ ] Production API credentials configured
[ ] Production base URL verified
[ ] Server IP/network access verified
[ ] HTTPS active
[ ] Webhook URL publicly reachable
[ ] Webhook authentication configured
[ ] Queue worker running
[ ] Cron/scheduler running if polling is used
[ ] Database migration completed
[ ] Logging configured
[ ] Error monitoring configured
[ ] Backup configured
[ ] Test order completed
[ ] Test tracking completed
[ ] Test webhook completed
[ ] Duplicate webhook tested
[ ] Duplicate order protection tested
```

---

# 30. Reference

Primary documentation supplied for this project:

**SteadFast Postman Documentation**

https://documenter.getpostman.com/view/26211192/2sAYJ1mhpk

Additional publicly available implementation references were used only to cross-check common SteadFast integration concepts such as order creation, bulk orders, tracking identifiers, balance, returns, payments, and webhook handling. These secondary references should not override the official SteadFast documentation.

---

## Disclaimer

This file is intended as a technical integration guide for an OMS/e-commerce codebase.

The exact SteadFast API endpoint paths, authentication headers, required parameters, response schemas, webhook contract, and available features can change. Always verify the live official API documentation before deploying a new integration or changing production behavior.

**Last reviewed:** 2026-10-07
