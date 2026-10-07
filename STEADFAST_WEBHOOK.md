# Steadfast Courier Webhook Documentation

> Integration reference for receiving Steadfast Courier webhook events
> in a Laravel/API application.

## Overview

Steadfast Courier can send an HTTP `POST` request to a configured
webhook endpoint whenever a parcel or account-related event changes.

The webhook endpoint must:

-   Use HTTPS.
-   Accept JSON `POST` requests.
-   Validate the authentication token.
-   Verify the `X-Signature` HMAC-SHA256 signature against the **raw
    request body**.
-   Use `Idempotency-Key` to prevent duplicate processing when Steadfast
    retries the same event.
-   Return a `2xx` response when the event is successfully
    accepted/processed.

### Example project endpoint

``` text
POST https://api.elaraborka.com/api/webhooks/couriers/steadfast
```

> Replace the endpoint with the URL configured for your own application.

------------------------------------------------------------------------

## Webhook Request

### HTTP Method

``` http
POST
```

### Content Type

``` http
Content-Type: application/json
```

### Headers

  -----------------------------------------------------------------------
  Header                              Description
  ----------------------------------- -----------------------------------
  `Content-Type`                      `application/json`

  `Authorization`                     `Bearer <your-auth-token>`

  `X-Signature`                       HMAC-SHA256 signature of the raw
                                      request body, keyed with the auth
                                      token

  `Idempotency-Key`                   Identifier used to recognize
                                      retries of the same event

  `User-Agent`                        `Steadfast-Webhook/1.0`
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Authentication

The configured webhook authentication token is sent as:

``` http
Authorization: Bearer <TOKEN>
```

The same token is also used as the HMAC secret for `X-Signature`.

### Important Security Rule

Never expose the webhook token in:

-   GitHub repositories
-   frontend JavaScript
-   screenshots
-   logs
-   public documentation
-   client-side API responses
-   chat messages

Store it as a server-side secret, for example:

``` env
STEADFAST_WEBHOOK_TOKEN=your-secret-token
```

Do not commit `.env` files containing the real token.

------------------------------------------------------------------------

# X-Signature Verification

Steadfast signs the **raw HTTP request body** using HMAC-SHA256 and the
configured authentication token.

Conceptually:

``` text
signature = HMAC-SHA256(raw_request_body, webhook_token)
```

The calculated signature must be compared with:

``` http
X-Signature: <signature>
```

Use a timing-safe comparison such as PHP `hash_equals()`.

## Why the Raw Body Matters

Do not parse the JSON and then re-encode it before calculating the
signature.

The signature must be calculated from the exact raw bytes received by
the server.

Correct flow:

``` text
HTTP Request
    |
    +--> Read raw body
    |
    +--> Calculate HMAC-SHA256(raw body, token)
    |
    +--> Compare with X-Signature
    |
    +--> Parse JSON
    |
    +--> Validate/process event
```

------------------------------------------------------------------------

# PHP Example

``` php
$body = file_get_contents('php://input');

$token = getenv('STEADFAST_TOKEN');

$expected = hash_hmac('sha256', $body, $token);

$given = $_SERVER['HTTP_X_SIGNATURE'] ?? '';

if (!hash_equals($expected, $given)) {
    http_response_code(401);
    exit;
}

$event = json_decode($body, true);
```

For a Laravel application, the same principle should be implemented
using the framework's request handling while preserving access to the
original raw body for signature verification.

------------------------------------------------------------------------

# Idempotency

Steadfast provides:

``` http
Idempotency-Key: <event-key>
```

When Steadfast retries the same event, the same idempotency key is used.

The receiver should store and check this value before processing the
event again.

### Recommended behavior

``` text
Receive webhook
      |
      v
Verify authentication/signature
      |
      v
Read Idempotency-Key
      |
      +---- Already processed? ----> Return 2xx
      |
      v
Process event
      |
      v
Store event/key
      |
      v
Return 2xx
```

This prevents duplicate:

-   order updates
-   tracking history entries
-   notifications
-   financial updates
-   status transitions

------------------------------------------------------------------------

# Retry Behavior

According to the Steadfast webhook specification:

### Successful response

Any `2xx` response indicates success.

### Server error / unavailable

If the server:

-   returns `5xx`, or
-   cannot be reached,

Steadfast retries the event:

1.  After approximately **30 seconds**
2.  After approximately **2 minutes**

There are **3 attempts in total**.

### `4xx` response

A `4xx` response is **not retried**.

Therefore, the application should distinguish between:

-   invalid/unauthorized request → appropriate `4xx`
-   temporary server failure → appropriate `5xx`
-   successfully accepted event → `2xx`

------------------------------------------------------------------------

# Webhook Events

The documented event types are:

  Event                    Description
  ------------------------ ----------------------------------------------
  `delivery_status`        A parcel's delivery status changed
  `tracking_update`        A new tracking step was added
  `consignment_update`     A parcel was changed, such as address or COD
  `return_list_accepted`   Receipt of a return list was confirmed
  `payment_request`        A payout was requested
  `cancel_request`         A cancellation request was raised or decided
  `return_request`         A return request was raised
  `pickup_request`         A pickup was requested
  `user_update`            Account details changed

------------------------------------------------------------------------

# `delivery_status` Event

The documented `delivery_status` event contains fields such as:

``` json
{
  "notification_type": "delivery_status",
  "consignment_id": 12345,
  "invoice": "INV-67890",
  "cod_amount": 1500.0,
  "status": "delivered",
  "delivery_charge": 100.0,
  "tracking_message": "Your package has been delivered successfully.",
  "updated_at": "2025-03-02 12:45:30"
}
```

## Fields

  Field                 Description
  --------------------- ----------------------------------------------
  `notification_type`   Event type
  `consignment_id`      Steadfast consignment identifier
  `invoice`             Invoice/reference associated with the parcel
  `cod_amount`          COD amount supplied by the provider
  `status`              Delivery status
  `delivery_charge`     Delivery charge supplied by the provider
  `tracking_message`    Provider tracking message
  `updated_at`          Provider event update timestamp

The documented status values for `delivery_status` include:

``` text
pending
delivered
partial_delivered
cancelled
unknown
```

------------------------------------------------------------------------

# Example `delivery_status` Request

``` http
POST /api/webhooks/couriers/steadfast HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer <TOKEN>
X-Signature: <HMAC-SHA256-SIGNATURE>
Idempotency-Key: <EVENT-ID>
User-Agent: Steadfast-Webhook/1.0
```

``` json
{
  "notification_type": "delivery_status",
  "consignment_id": 12345,
  "invoice": "INV-67890",
  "cod_amount": 1500.0,
  "status": "delivered",
  "delivery_charge": 100.0,
  "tracking_message": "Your package has been delivered successfully.",
  "updated_at": "2025-03-02 12:45:30"
}
```

------------------------------------------------------------------------

# Event Processing Architecture

A secure implementation should separate provider data from internal OMS
data.

``` text
Steadfast Webhook
        |
        v
Raw Request Body
        |
        +--> Verify Bearer authentication
        |
        +--> Verify X-Signature
        |
        +--> Validate Idempotency-Key
        |
        +--> Validate JSON
        |
        +--> Validate event type
        |
        +--> Correlate consignment/invoice
        |
        +--> Store provider event
        |
        +--> Apply safe status update
        |
        +--> Update tracking history
        |
        v
      2xx
```

------------------------------------------------------------------------

# Provider Data vs OMS Data

Provider values should not blindly overwrite internal OMS financial
values.

### OMS data

Examples:

``` text
subtotal
discount
delivery_charge
grand_total
advance
due_amount
```

### Provider data

Examples:

``` text
cod_amount
delivery_charge
provider status
tracking_message
provider timestamps
provider event identifiers
```

Keep the two sources distinguishable.

For example:

``` text
OMS Delivery Charge
Provider Delivery Charge
```

should not automatically be treated as the same field unless the
business logic explicitly establishes that relationship.

Likewise:

``` text
cod_amount
```

should not automatically be labeled as "collected cash" unless provider
semantics confirm that interpretation.

------------------------------------------------------------------------

# Event Ordering

Webhook delivery can involve retries and delayed events.

A robust implementation should protect against an older event
overwriting newer state.

Recommended principles:

-   Preserve the original provider timestamp.
-   Preserve the webhook receipt timestamp.
-   Preserve the provider event identity when available.
-   Make processing idempotent.
-   Detect duplicate events.
-   Prevent stale events from regressing a newer status.
-   Keep provider history append-only where appropriate.

------------------------------------------------------------------------

# Unknown or Additional Fields

Provider payloads may contain fields not currently documented or
expected by the application.

The receiver should:

-   tolerate additional JSON fields
-   safely handle optional fields
-   safely handle `null`
-   reject malformed payloads
-   avoid assuming undocumented semantics
-   avoid converting free-text into structured data without evidence

Do not fabricate:

-   rider information
-   hub information
-   dispatch IDs
-   delivery attempts
-   settlement values
-   collection status

unless the provider payload/contract actually supplies and defines them.

------------------------------------------------------------------------

# Webhook Security Checklist

Before enabling the webhook in production:

-   [ ] HTTPS is enabled.
-   [ ] Bearer token is stored server-side.
-   [ ] Token is never logged.
-   [ ] Raw request body is used for HMAC verification.
-   [ ] `X-Signature` is verified.
-   [ ] Timing-safe comparison is used.
-   [ ] `Idempotency-Key` is persisted/checked.
-   [ ] Duplicate callbacks are safe.
-   [ ] Payload size is limited.
-   [ ] JSON validation is enforced.
-   [ ] Event type is validated.
-   [ ] Consignment/invoice correlation is validated.
-   [ ] Provider messages are safely rendered.
-   [ ] Secrets are redacted from logs.
-   [ ] Customer PII is minimized.
-   [ ] Out-of-order events cannot regress newer state.
-   [ ] 4xx and 5xx responses are used appropriately.
-   [ ] No production data reset/destructive operation is used for
    testing.

------------------------------------------------------------------------

# Laravel Implementation Notes

A Laravel implementation should ensure the webhook route is publicly
reachable without normal Admin authentication middleware while still
enforcing the provider's own webhook authentication.

Example route:

``` php
Route::post(
    '/webhooks/couriers/steadfast',
    [SteadfastWebhookController::class, 'receive']
);
```

The controller should verify the raw body signature before trusting the
JSON payload.

A middleware can also be used for:

-   request-size limiting
-   raw request handling
-   authentication/signature verification
-   common webhook validation

Keep provider webhook authentication separate from normal:

``` text
auth:sanctum
approved.device
Admin RBAC
```

because Steadfast is an external provider, not an authenticated Admin
browser session.

------------------------------------------------------------------------

# Testing Strategy

## Unit/Feature Tests

Test at minimum:

1.  Valid signed webhook
2.  Missing `Authorization`
3.  Invalid Bearer token
4.  Missing `X-Signature`
5.  Invalid signature
6.  Modified request body
7.  Missing `Idempotency-Key`
8.  Duplicate idempotency key
9.  Malformed JSON
10. Missing `consignment_id`
11. Missing `invoice`
12. Unknown event type
13. `delivery_status`
14. `tracking_update`
15. `consignment_update`
16. `cancel_request`
17. `return_request`
18. `payment_request`
19. Out-of-order events
20. Concurrent duplicate delivery
21. Provider message containing HTML/script
22. Provider financial fields
23. Safe `5xx` handling
24. Safe `4xx` handling

------------------------------------------------------------------------

# Real Webhook Testing

Use an existing Steadfast consignment whenever possible.

Do not create unnecessary test orders just to test webhook delivery.

During a real test, capture a sanitized record of:

-   request timestamp
-   HTTP status
-   response body
-   request headers
-   `notification_type`
-   `consignment_id`
-   `invoice`
-   `status`
-   `cod_amount`
-   `delivery_charge`
-   `tracking_message`
-   `updated_at`
-   `Idempotency-Key`
-   `X-Signature` presence/verification result

Never publish:

-   real webhook token
-   API secret
-   cookies
-   authorization credentials
-   unnecessary customer PII

------------------------------------------------------------------------

# Important Limitations

The webhook documentation establishes the authentication/signature
mechanism and documented event structure.

It does **not by itself establish** that webhook payloads contain
structured:

-   rider name/phone
-   hub name/phone
-   dispatch ID
-   delivery-attempt history
-   complete courier journey history
-   settlement allocation

Those capabilities must be verified from actual provider payloads or
provider documentation before implementation.

------------------------------------------------------------------------

# Production Deployment Principles

When deploying webhook support to an existing OMS:

1.  Back up the production database.
2.  Create a new release rather than overwriting the active release.
3.  Preserve the production `.env`.
4.  Preserve existing storage/uploads.
5.  Apply only additive, reviewed migrations.
6.  Verify the webhook route before activating it.
7.  Verify HTTPS and public reachability.
8.  Configure the Steadfast callback URL.
9.  Configure the webhook authentication token securely.
10. Perform a controlled real webhook test.
11. Verify idempotency and ordering.
12. Monitor logs for errors without logging secrets.

Never use:

``` bash
php artisan migrate:fresh
```

or:

``` bash
php artisan db:wipe
```

on production.

------------------------------------------------------------------------

# Reference

This document is based on the Steadfast Webhook specification supplied
for the integration, including:

-   Webhook endpoint configuration
-   Bearer authentication
-   HMAC-SHA256 `X-Signature`
-   `Idempotency-Key`
-   retry behavior
-   documented webhook event types
-   `delivery_status` example payload
-   request verification examples

------------------------------------------------------------------------

## License / Usage

This file is intended as an integration reference for development and
internal documentation.

Do not publish private API credentials, webhook tokens, customer PII, or
production secrets in this repository.
