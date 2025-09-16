
# PipraPay Payment Gateway — **Comprehensive API & Webhook Guide**

**Version:** 1.0 • **Last updated:** 2025-09-16 08:33:56

> This document is a fully self-contained, production-ready guide for integrating the PipraPay gateway into your backend.
> It explains the API endpoints, the exact request/response contracts, security model, and a robust webhook strategy where **the webhook is handled in your backend only** — **no webhook needs to be configured on any dashboard/website**.  
> Instead, you **pass your backend webhook URL dynamically** in `create-charge` via the `webhook_url` field.

---
## Contributors

- **Name:** Fattain Naime  
- **Website:** [https://iamnaime.info.bd](https://iamnaime.info.bd)  

**Acknowledgments:**  
This documentation is community-authored with attention to clarity and completeness. Additional improvements, peer reviews, and real-world implementation feedback are welcome.  

---

## Table of Contents

1. [At a Glance](#at-a-glance)  
2. [Base URL & Authentication](#base-url--authentication)  
3. [End-to-End Payment Flow](#end-to-end-payment-flow)  
4. [API Reference](#api-reference)  
    - [Create Charge](#create-charge)  
    - [Verify Payment](#verify-payment)  
5. [Webhooks (Backend-Only)](#webhooks-backend-only)  
    - [Webhook Event Payload](#webhook-event-payload)  
    - [Security & Signature Validation](#security--signature-validation)  
    - [Recommended Webhook Handlers (PHP/Node/Python)](#recommended-webhook-handlers-phpnodepython)  
    - [Idempotency, Retries & Ordering](#idempotency-retries--ordering)  
6. [Metadata & Reconciliation](#metadata--reconciliation)  
7. [Error Handling & Troubleshooting](#error-handling--troubleshooting)  
8. [Frequently Asked Questions (FAQ)](#frequently-asked-questions-faq)  
9. [Change Log](#change-log)

---

## At a Glance

- **Create a charge** → Redirect your customer to the PipraPay-hosted checkout page.  
- **Get paid** → Customer completes payment using supported methods (e.g., *bKash, nagad*).  
- **Receive webhook (your backend only)** → PipraPay sends a POST to your backend `webhook_url` with payment result.  
- **Verify** → Call `verify-payments` using `pp_id` to confirm final status server-to-server.  
- **No dashboard webhook configuration required** → Pass `webhook_url` *per charge* from your backend.

---

## Base URL & Authentication

**Production Base URL**  
```
https://payment.builderhall.com
```

**Authentication**  
All authenticated requests must include your secret API key in the header:
```
mh-piprapay-api-key: <YOUR_API_KEY>
```
- Keep this key **server-side only**. Do **not** expose it to browsers or mobile apps.  
- Rotate keys if compromised.

---

## End-to-End Payment Flow

```text
Your Backend                  Customer Browser                PipraPay
     |                               |                          |
1.   | POST /api/create-charge       |                          |
     |------------------------------>|                          |
     |    returns {pp_id, pp_url}    |                          |
2.   |<------------------------------|                          |
3.   | redirect user to pp_url       |------------------------->| Checkout
     |                               |     customer pays        |
4.   |  (async) Webhook to your backend (POST) with result <----|
5.   |  POST /api/verify-payments to confirm         |
6.   |  update your order/db and fulfill                        |
```

**Key integration choices**  
- **Webhook-first**: Receive asynchronous status in your backend.  
- **Verification**: Use `verify-payments` to confirm status.  
- **No static dashboard webhook**: Pass `webhook_url` dynamically per charge from your backend.  

---

## API Reference

### Create Charge

**Endpoint**  
```
POST https://payment.builderhall.com/api/create-charge
```

**Headers**
```
Accept: application/json
Content-Type: application/json
mh-piprapay-api-key: <YOUR_API_KEY>
```

**Request Body**
```json
{
  "full_name": "Demo",
  "email_mobile": "01999999999",
  "amount": "10",
  "metadata": {
    "invoiceid": "inv123"
  },
  "redirect_url": "https://piprapay.com",
  "return_type": "GET",
  "cancel_url": "https://piprapay.com",
  "webhook_url": "https://your-backend.example.com/api/piprapay/webhook",
  "currency": "BDT"
}
```
**Field Details**

| Field | Type | Required | Notes |
|---|---|:---:|---|
| `full_name` | string | ✔ | Customer’s full name for display/reconciliation. |
| `email_mobile` | string | ✔ | Customer contact (email or mobile). |
| `amount` | string/number | ✔ | Charge amount. Use smallest currency unit if instructed by provider; here examples use whole BDT. |
| `metadata` | object | ✔ | Arbitrary key-value pairs. Use to pass `invoiceid`, `order_id`, etc. Returned in the webhook/verify response. |
| `redirect_url` | string (URL) | ✔ | Where the customer is sent after successful/attempted payment. |
| `return_type` | `"GET"` | ✔ | Method PipraPay uses when redirecting the customer to `redirect_url`. |
| `cancel_url` | string (URL) | ✔ | Where to send user if they cancel from checkout. |
| `webhook_url` | string (URL) | ✔ | **Your backend endpoint** to receive asynchronous payment results. **Do not configure webhooks on any dashboard**; pass this URL here instead. |
| `currency` | string | ✔ | Currency code. Example: `"BDT"`. |

**Successful Response**
```json
{
  "status": true,
  "pp_id": 789234894,
  "pp_url": "https://payment.builderhall.com/payment/789234894"
}
```
- `pp_id` is the unique payment identifier (string or number).  
- `pp_url` is the hosted checkout URL. Redirect the user here.

**Typical Usage (curl)**
```bash
curl --request POST   --url https://payment.builderhall.com/api/create-charge   --header 'accept: application/json'   --header 'content-type: application/json'   --header 'mh-piprapay-api-key: <YOUR_API_KEY>'   --data '{
    "full_name": "Demo",
    "email_mobile": "01994493830",
    "amount": "10",
    "metadata": {"invoiceid": "inv123"},
    "redirect_url": "https://yourapp.example.com/checkout/return",
    "return_type": "GET",
    "cancel_url": "https://yourapp.example.com/checkout/cancel",
    "webhook_url": "https://yourapp.example.com/api/piprapay/webhook",
    "currency": "BDT"
}'
```

---

### Verify Payment

**Endpoint**  
```
POST https://payment.builderhall.com/api/verify-payments
```

**Headers**
```
Accept: application/json
Content-Type: application/json
mh-piprapay-api-key: <YOUR_API_KEY>
```

**Request Body**
```json
{
  "pp_id": "789234894"
}
```

**Response**
```json
{
  "pp_id": "789234894",
  "customer_name": "Demo",
  "customer_email_mobile": "01999999999",
  "payment_method": "bKash Personal",
  "amount": 10,
  "fee": 0,
  "refund_amount": 0,
  "total": 10,
  "currency": "BDT",
  "metadata": {
    "invoiceid": "inv123"
  },
  "sender_number": "01999999999",
  "transaction_id": "121212",
  "status": "completed",
  "date": "2025-09-16 14:20:48"
}
```

**When to use**  
- Immediately after receiving a webhook.  
- When the customer returns to your `redirect_url` but you haven’t received the webhook yet.  
- For back office reconciliation, refunds tracking, status checks.

**Expected `status` values** (typical patterns)  
- `completed` — funds captured / payment finished.  
- `pending` — customer started but not finished (or awaiting confirmation).  
- `failed` / `cancelled` — payment not completed.  
> Always **treat only `completed`** as successful for fulfillment.

---

## Webhooks (Backend-Only)

### Key Principles

- **Do not configure any webhook in a dashboard or third-party site.**  
- **Always pass your backend webhook URL** in `create-charge.webhook_url`.  
- Your webhook endpoint **must accept** a POST with a JSON body and inspect the `mh-piprapay-api-key` header for authorization (see below).  
- Your endpoint **must return HTTP 200** quickly after basic validation (you can process business logic asynchronously).

### Webhook Event Payload

**Example (from PipraPay)**
```json
{
  "pp_id": "181055228",
  "customer_name": "Demo",
  "customer_email_mobile": "demo@gmail.com",
  "payment_method": "bKash Personal",
  "amount": "10",
  "fee": "0",
  "refund_amount": "0",
  "total": 10,
  "currency": "BDT",
  "metadata": {
    "invoiceid": "uouyo"
  },
  "sender_number": "568568568",
  "transaction_id": "io[io[o",
  "status": "completed",
  "date": "2025-06-26 13:34:13"
}
```

**Field notes**
- Values can be strings or numbers depending on context; parse accordingly.  
- `metadata` echoes what you passed in `create-charge` and is ideal for mapping to your internal order/invoice.  
- Use `pp_id` + your `invoiceid` to deduplicate/process idempotently.

### Security & Signature Validation

PipraPay includes your API key in the webhook request header. Validate it strictly before trusting the payload.

**Headers to check (any one may be present)**  
- `mh-piprapay-api-key`  
- `Mh-Piprapay-Api-Key`  
- `HTTP_MH_PIPRAPAY_API_KEY` (server/CGI fallback)

Reject the request with **401 Unauthorized** if the header value does not exactly match your server-side API key.

> **Tip:** Limit your webhook endpoint to accept traffic only from PipraPay IPs if they publish them (or behind an allowlist, VPN, or WAF).

---

### Recommended Webhook Handlers (PHP/Node/Python)

#### PHP (reference implementation from provider, simplified)

```php
<?php
$rawData = file_get_contents("php://input");
$data = json_decode($rawData, true);
$headers = getallheaders();

$received_api_key = '';
if (isset($headers['mh-piprapay-api-key'])) {
    $received_api_key = $headers['mh-piprapay-api-key'];
} elseif (isset($headers['Mh-Piprapay-Api-Key'])) {
    $received_api_key = $headers['Mh-Piprapay-Api-Key'];
} elseif (isset($_SERVER['HTTP_MH_PIPRAPAY_API_KEY'])) {
    $received_api_key = $_SERVER['HTTP_MH_PIPRAPAY_API_KEY'];
}

if ($received_api_key !== "YOUR_API") {
    http_response_code(401);
    echo json_encode(["status" => false, "message" => "Unauthorized request."]);
    exit;
}

$pp_id = $data['pp_id'] ?? '';
$status = $data['status'] ?? 'unknown';
$metadata = $data['metadata'] ?? [];

// Idempotency: only process each pp_id once
// (e.g., check a 'payments' table if pp_id is already marked processed)

if ($status === 'completed') {
    // Mark order paid, fulfill, send receipt, etc.
} else {
    // Mark failed/pending/etc., notify customer if needed
}

http_response_code(200);
echo json_encode(['status' => true, 'message' => 'Webhook received']);
```

#### Node.js (Express)

```js
import express from 'express';
import bodyParser from 'body-parser';

const app = express();
app.use(bodyParser.json());

const API_KEY = process.env.PIPRAPAY_API_KEY;

app.post('/api/piprapay/webhook', (req, res) => {
  const headers = req.headers;
  const received =
    headers['mh-piprapay-api-key'] ||
    headers['Mh-Piprapay-Api-Key'] ||
    headers['http_mh_piprapay_api_key'];

  if (received !== API_KEY) {
    return res.status(401).json({ status: false, message: 'Unauthorized request.' });
  }

  const {
    pp_id, status, metadata, amount, currency, transaction_id
  } = req.body || {};

  // TODO: implement idempotency using pp_id and/or metadata.invoiceid
  // TODO: queue work; acknowledge fast
  res.status(200).json({ status: true, message: 'Webhook received' });
});

app.listen(3000, () => console.log('Webhook listening on :3000'));
```

#### Python (Flask)

```python
from flask import Flask, request, jsonify
import os

app = Flask(__name__)
API_KEY = os.environ.get("PIPRAPAY_API_KEY")

@app.post("/api/piprapay/webhook")
def piprapay_webhook():
    received = (
        request.headers.get("mh-piprapay-api-key")
        or request.headers.get("Mh-Piprapay-Api-Key")
        or request.headers.get("HTTP_MH_PIPRAPAY_API_KEY")
    )
    if received != API_KEY:
        return jsonify({"status": False, "message": "Unauthorized request."}), 401

    data = request.get_json(silent=True) or {}
    pp_id = data.get("pp_id")
    status = data.get("status", "unknown")

    # TODO: implement idempotency & queue processing
    return jsonify({"status": True, "message": "Webhook received"}), 200

if __name__ == "__main__":
    app.run(port=3000)
```

---

### Idempotency, Retries & Ordering

- **Idempotency**: Persist each `pp_id` (and/or `metadata.invoiceid`) and ensure you **process it only once**.  
- **Retries**: Webhooks may be retried on network errors. Always make your processing idempotent and return 2xx on success.  
- **Ordering**: If multiple events arrive out of order, use `date` or perform a fresh **Verify** call to confirm the latest state.

---

## Metadata & Reconciliation

- Use `metadata.invoiceid` (or your own IDs) to tie the payment back to your internal order/invoice/customer.  
- The same `metadata` object is echoed in both **webhooks** and **verify** responses, simplifying reconciliation.  
- Store: `pp_id`, `status`, `amount`, `currency`, `transaction_id`, `payment_method`, `date`, and `metadata` for audits.

---

## Error Handling & Troubleshooting

### Common HTTP Statuses
- **200**: Success.  
- **400**: Validation problem (missing/invalid fields).  
- **401**: Unauthorized (missing/invalid `mh-piprapay-api-key`).  
- **5xx**: Temporary server issue; implement retries with exponential backoff.

### Debug Checklist
- Confirm `mh-piprapay-api-key` header is present and correct for both API calls and webhooks.  
- Ensure your **webhook URL is reachable** over the public internet (HTTPS strongly recommended).  
- Check firewall/WAF/allowlist rules.  
- Log full raw webhook request for diagnostics (avoid logging secrets).  
- If webhook not received, use **Verify Payment** with `pp_id` as fallback.  
- For redirect flows, always persist `pp_id` and `invoiceid` before sending the user to `pp_url`.

---

## Frequently Asked Questions (FAQ)

**Q: Do I need to configure a webhook in a dashboard or third-party site?**  
**A:** No. This integration is **backend-only**. Pass your webhook endpoint in `create-charge.webhook_url` from your server.

**Q: Is `verify-payments` required if I already trust the webhook?**  
**A:** It’s optional but recommended for defense-in-depth (e.g., if your webhook queue is delayed or to double-check final status).

**Q: Which status should I treat as success?**  
**A:** `completed` only. For all other statuses, do not fulfill the order.

**Q: What currency is supported in examples?**  
**A:** Examples show `BDT`. Use the currency agreed upon with PipraPay.

**Q: Can I pass custom data?**  
**A:** Yes. Add keys in `metadata` (e.g., `invoiceid`, `customer_id`). They are echoed back in webhook and verify responses.

---

## Change Log

- **v1.0** — Initial comprehensive guide with backend-only webhook strategy, endpoint docs, handlers in PHP/Node/Python, and best practices.

---

© 2025 PipraPay Integration Guide (community-authored). For production secrets & contracts, always confirm with your provider.
