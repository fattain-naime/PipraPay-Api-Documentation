# PipraPay API — Practical Integration Guide (v1.2)

*Last updated: September 17, 2025*

This guide distills the official PipraPay API reference into a concise, developer‑friendly format you can drop into your project wiki or repo.

`Overview` • `Auth` • `Create Charge` • `Verify Payment` • `Webhooks` • `Examples` • `Fallback strategy`

---

## 1) Overview

PipraPay exposes a focused set of REST endpoints to start a payment (“create charge”), verify a payment by its `pp_id`, and receive asynchronous status via webhooks.

- Docs hub: [piprapay.readme.io](https://piprapay.readme.io/reference/overview)
- Sandbox base URL: `https://sandbox.piprapay.com`
- Content type: `application/json`

> **Note**  
> Create Charge returns identifiers to your redirect URL, and full payment details can be retrieved by verifying with the *pp_id*.

---

## 2) Environments

- **Sandbox**: `https://sandbox.piprapay.com`  
- **Production (self‑hosted)**: your deployed domain (e.g., `https://pay.yourdomain.com`)

---

## 3) Authentication

All requests require an API Key supplied via headers. Webhook requests from PipraPay include the header `mh-piprapay-api-key`, which you should validate on your server.

```
mh-piprapay-api-key: <YOUR_API_KEY>
```

---

## 4) End‑to‑end Flow at a Glance

1. **Create a charge** with customer details, amount, and your `redirect_url` and `webhook_url`.
2. Customer completes checkout in PipraPay’s hosted flow.
3. After success, PipraPay posts identifiers to your `redirect_url` and fires your `webhook_url` (if provided).
4. Read the `pp_id` (from webhook or redirect).
5. **Verify** using `POST /api/verify-payments` with that `pp_id`.
6. Mark the order as paid and fulfill.

---

## 5) Create Charge

**Endpoint**: `POST https://sandbox.piprapay.com/api/create-charge`

| Field          | Type        | Notes                                      |
|---------------|-------------|--------------------------------------------|
| `full_name`   | string      | Customer name                              |
| `email_mobile`| string      | Customer email *or* mobile number          |
| `amount`      | string      | Amount as string (e.g., `"10"`)            |
| `metadata`    | object      | Optional JSON (e.g., `{'order_id':'123'}`) |
| `redirect_url`| string (URL)| Browser redirect after success             |
| `return_type` | string      | `GET` or `POST`                            |
| `cancel_url`  | string (URL)| Redirect if the user cancels               |
| `webhook_url` | string (URL)| Server-to-server notifications             |
| `currency`    | string      | Currency code (e.g., `BDT`)                |

**Return identifiers**

- `invoice_id` sent to your return URL (GET/POST)  
- `pp_id` may be posted to your `redirect_url`

**cURL**

```bash
curl --request POST   --url https://sandbox.piprapay.com/api/create-charge   --header 'accept: application/json'   --header 'content-type: application/json'   --data '{
    "full_name": "Demo",
    "email_mobile": "demo@gmail.com",
    "amount": "10",
    "metadata": {"order_id": "ORD-10001"},
    "redirect_url": "https://example.com/pay/success",
    "return_type": "POST",
    "cancel_url": "https://example.com/pay/cancel",
    "webhook_url": "https://example.com/api/piprapay/webhook",
    "currency": "BDT"
  }'
```

**Node.js (axios)**

```javascript
import axios from "axios";

const res = await axios.post("https://sandbox.piprapay.com/api/create-charge",
  {
    full_name: "Demo",
    email_mobile: "demo@gmail.com",
    amount: "10",
    metadata: { order_id: "ORD-10001" },
    redirect_url: "https://example.com/pay/success",
    return_type: "POST",
    cancel_url: "https://example.com/pay/cancel",
    webhook_url: "https://example.com/api/piprapay/webhook",
    currency: "BDT"
  },
  {
    headers: {
      "accept": "application/json",
      "content-type": "application/json"
    }
  }
);

console.log(res.data);
```

**PHP (cURL)**

```php
<?php
$ch = curl_init("https://sandbox.piprapay.com/api/create-charge");
curl_setopt_array($ch, [
  CURLOPT_POST => true,
  CURLOPT_HTTPHEADER => [
    "accept: application/json",
    "content-type: application/json",
  ],
  CURLOPT_POSTFIELDS => json_encode([
    "full_name" => "Demo",
    "email_mobile" => "demo@gmail.com",
    "amount" => "10",
    "metadata" => ["order_id" => "ORD-10001"],
    "redirect_url" => "https://example.com/pay/success",
    "return_type" => "POST",
    "cancel_url" => "https://example.com/pay/cancel",
    "webhook_url" => "https://example.com/api/piprapay/webhook",
    "currency" => "BDT"
  ]),
  CURLOPT_RETURNTRANSFER => true,
]);
$body = curl_exec($ch);
curl_close($ch);
echo $body;
```

**Python (requests)**

```python
import requests

resp = requests.post(
  "https://sandbox.piprapay.com/api/create-charge",
  headers={"accept":"application/json","content-type":"application/json"},
  json={
    "full_name":"Demo",
    "email_mobile":"demo@gmail.com",
    "amount":"10",
    "metadata":{"order_id":"ORD-10001"},
    "redirect_url":"https://example.com/pay/success",
    "return_type":"POST",
    "cancel_url":"https://example.com/pay/cancel",
    "webhook_url":"https://example.com/api/piprapay/webhook",
    "currency":"BDT"
  }
)
print(resp.json())
```

> *Reference: Create Charge API. See official docs for live examples and response schemas.*

---

## 6) Verify Payment

**Endpoint**: `POST https://sandbox.piprapay.com/api/verify-payments`

| Field  | Type   | Notes                         |
|--------|--------|-------------------------------|
| `pp_id`| string | PipraPay Payment ID to verify |

**cURL**

```bash
curl --request POST   --url https://sandbox.piprapay.com/api/verify-payments   --header 'accept: application/json'   --header 'content-type: application/json'   --data '{"pp_id":"181055228"}'
```

> *Reference: Verify Payment API.*

---

## 7) Webhooks

> **Security**: Validate the `mh-piprapay-api-key` header on each webhook. Only process events if it matches your configured API key.

### Official PHP validation example

From [Validate Webhook](https://piprapay.readme.io/reference/validate-webhook) page:

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
    $received_api_key = $_SERVER['HTTP_MH_PIPRAPAY_API_KEY']; // fallback if needed
}

if ($received_api_key !== "YOUR_API") {
    status_header(401);
    echo json_encode(["status" => false, "message" => "Unauthorized request."]);
    exit;
}

$pp_id = $data['pp_id'] ?? '';
$customer_name = $data['customer_name'] ?? '';
$customer_email_mobile = $data['customer_email_mobile'] ?? '';
$payment_method = $data['payment_method'] ?? '';
$amount = $data['amount'] ?? 0;
$fee = $data['fee'] ?? 0;
$refund_amount = $data['refund_amount'] ?? 0;
$total = $data['total'] ?? 0;
$currency = $data['currency'] ?? '';
$status = $data['status'] ?? '';
$date = $data['date'] ?? '';
$metadata = $data['metadata'] ?? [];

http_response_code(200);
echo json_encode(['status' => true, 'message' => 'Webhook received']);
```

### Example webhook payload

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
  "metadata": { "invoiceid": "uouyo" },
  "sender_number": "568568568",
  "transaction_id": "io[io[o",
  "status": "completed",
  "date": "2025-06-26 13:34:13"
}
```

> *Reference: Validate Webhook page. Use the payload’s `pp_id` to verify.*

---

## 8) Putting It Together — Minimal Server Flow

> **Important about `pp_id` sources:**
>
> - PipraPay sends `pp_id` in two ways:  
>   1) via your **webhook_url** (server-to-server, preferred)  
>   2) via your **redirect_url** (browser flow)
> - **Prioritize the webhook** (reliable, signed with API key).
> - **Use the redirect `pp_id` as a fallback** if the webhook is delayed or not received.
> - Implement idempotency so both paths converge safely.

1. Call **Create Charge** with `redirect_url`, `cancel_url`, `webhook_url`.
2. Handle redirect (GET/POST) and capture `pp_id` if present.
3. Process webhook: validate header, read `pp_id`, enqueue processing.
4. Call **Verify Payment** with `pp_id`.
5. Mark order paid and fulfill on verified success.

---

## 9) Troubleshooting & Tips

- **Fallback safety**: if the webhook doesn’t reach your server, verify using the `pp_id` captured at `redirect_url`.
- **Idempotency**: use `pp_id` as the idempotency key when updating orders.
- **Quick ACK**: reply `200` to webhooks promptly; process heavy work async.
- **Currency**: pass `currency` explicitly (e.g., `BDT`).

---

## 10) Reference Links

- [Overview](https://piprapay.readme.io/reference/overview)
- [Create Charge](https://piprapay.readme.io/reference/create-charge)
- [Verify Payment](https://piprapay.readme.io/reference/verify-payments)
- [Validate Webhook](https://piprapay.readme.io/reference/validate-webhook)
