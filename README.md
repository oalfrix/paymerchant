# Merchant Payment API

## Installation & Integration Guide

Use this guide to connect your website to the Central Payment API and accept payments through PesaPal.

---

## 1. What You Need

Before starting, you need:

* A merchant account
* Your **Merchant API Key**
* A website
* Your website's payment/order page

Your API key is generated from your merchant dashboard.

**Important:** Never publish your API key inside frontend HTML or JavaScript that customers can view.

---

# 2. How the Payment System Works

Your website sends payment information to your own server.

Your server then sends the request to the Central Payment API.

```text
Customer
   ↓
Your Website
   ↓
Your Server / Backend
   ↓
Central Payment API
   ↓
PesaPal
   ↓
Customer completes payment
```

The Central Payment API handles communication with PesaPal.

---

# 3. Central Payment API

### Create Payment

**Method**

```text
POST
```

**Endpoint**

```text
https://lxrdkforhtiudmghydqx.supabase.co/functions/v1/central-create-payment
```

### Required Header

```http
Content-Type: application/json
x-api-key: YOUR_MERCHANT_API_KEY
```

---

# 4. Payment Request

Send JSON similar to:

```json
{
  "order_id": "ORDER-1001",
  "amount": 1000,
  "currency": "KES",
  "customer_phone": "0712345678",
  "customer_name": "John Doe",
  "customer_email": "john@example.com",
  "description": "Product purchase",
  "metadata": {
    "product_id": "product-001",
    "product_name": "T-Shirt"
  }
}
```

### Fields

| Field            | Required | Description                             |
| ---------------- | -------- | --------------------------------------- |
| `order_id`       | Yes      | Your unique order number                |
| `amount`         | Yes      | Payment amount                          |
| `currency`       | Yes      | Usually `KES`                           |
| `customer_phone` | Yes      | Customer phone number                   |
| `customer_name`  | No       | Customer name                           |
| `customer_email` | No       | Customer email                          |
| `description`    | No       | Description of payment                  |
| `metadata`       | No       | Additional information about your order |

---

# 5. Example Response

A successful request returns information similar to:

```json
{
  "success": true,
  "payment_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "order_tracking_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "merchant_reference": "xxxxxxxx-ORDER-1001",
  "checkout_url": "https://pay.pesapal.com/..."
}
```

The most important field for your website is:

```text
checkout_url
```

Redirect the customer to this URL so they can complete their payment.

---

# 6. Simple Website Integration

If you have a simple HTML website, you should use a small backend/serverless function.

**Do not put your API key directly into this code:**

```javascript
x-api-key: "pk_live_..."
```

Customers can inspect browser JavaScript and obtain the key.

Instead:

```text
Website
   ↓
/api/create-payment
   ↓
Your server
   ↓
Central Payment API
```

---

# 7. Example Frontend Code

Your Pay button can call your own backend:

```html
<button onclick="payNow()">
  Pay Now
</button>

<script>
async function payNow() {

  const payment = {
    order_id: "ORDER-" + Date.now(),
    amount: 1000,
    currency: "KES",
    customer_phone: "0712345678",
    customer_name: "",
    customer_email: "",
    description: "Product purchase"
  };

  const response = await fetch("/api/create-payment", {
    method: "POST",

    headers: {
      "Content-Type": "application/json"
    },

    body: JSON.stringify(payment)
  });

  const result = await response.json();

  if (!response.ok || !result.success) {
    alert(result.error || "Unable to start payment");
    return;
  }

  window.location.href = result.checkout_url;
}
</script>
```

Your `/api/create-payment` endpoint should then communicate with the Central Payment API using your secret API key.

---

# 8. Netlify Integration

If your website is hosted on Netlify, you can use a Netlify Function.

Create:

```text
your-website/
│
├── index.html
│
├── netlify.toml
│
└── netlify/
    └── functions/
        └── create-payment.js
```

Your `netlify.toml`:

```toml
[build]
  publish = "."
  functions = "netlify/functions"
```

---

# 9. Netlify Function

Create:

```text
netlify/functions/create-payment.js
```

Use:

```javascript
exports.handler = async (event) => {

  const headers = {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Headers": "Content-Type",
    "Access-Control-Allow-Methods": "POST, OPTIONS",
    "Content-Type": "application/json"
  };

  if (event.httpMethod === "OPTIONS") {
    return {
      statusCode: 204,
      headers,
      body: ""
    };
  }

  if (event.httpMethod !== "POST") {
    return {
      statusCode: 405,
      headers,
      body: JSON.stringify({
        error: "Method not allowed"
      })
    };
  }

  try {

    const data = JSON.parse(event.body || "{}");

    const CENTRAL_API_URL =
      process.env.CENTRAL_API_URL ||
      "https://lxrdkforhtiudmghydqx.supabase.co/functions/v1/central-create-payment";

    const MERCHANT_API_KEY =
      process.env.MERCHANT_API_KEY;

    if (!MERCHANT_API_KEY) {
      return {
        statusCode: 500,
        headers,
        body: JSON.stringify({
          error: "Payment configuration is missing"
        })
      };
    }

    const response = await fetch(
      CENTRAL_API_URL,
      {
        method: "POST",

        headers: {
          "Content-Type": "application/json",
          "x-api-key": MERCHANT_API_KEY
        },

        body: JSON.stringify(data)
      }
    );

    const result = await response.json();

    if (!response.ok) {
      return {
        statusCode: response.status,
        headers,
        body: JSON.stringify({
          error:
            result.error ||
            "Payment request failed"
        })
      };
    }

    return {
      statusCode: 200,
      headers,

      body: JSON.stringify({
        success: true,

        payment_id:
          result.payment_id,

        order_tracking_id:
          result.order_tracking_id ||
          result.tracking_id,

        merchant_reference:
          result.merchant_reference,

        checkout_url:
          result.checkout_url ||
          result.redirect_url
      })
    };

  } catch (error) {

    console.error(error);

    return {
      statusCode: 500,
      headers,

      body: JSON.stringify({
        error: "Unable to start payment"
      })
    };
  }
};
```

---

# 10. Add Your API Key to Netlify

Do **not** put your API key inside `index.html`.

In Netlify:

```text
Site configuration
      ↓
Environment variables
      ↓
Add variable
```

Create:

```text
MERCHANT_API_KEY
```

Set its value to the API key generated from your merchant dashboard.

You can also add:

```text
CENTRAL_API_URL
```

with:

```text
https://lxrdkforhtiudmghydqx.supabase.co/functions/v1/central-create-payment
```

---

# 11. Frontend Pay Button

Your website can now simply call:

```text
/.netlify/functions/create-payment
```

Example:

```javascript
async function payNow() {

  const response = await fetch(
    "/.netlify/functions/create-payment",
    {
      method: "POST",

      headers: {
        "Content-Type": "application/json"
      },

      body: JSON.stringify({
        order_id: "ORDER-" + Date.now(),
        amount: 1000,
        currency: "KES",
        customer_phone: "0712345678",
        customer_name: "",
        customer_email: "",
        description: "Product purchase"
      })
    }
  );

  const result = await response.json();

  if (!response.ok || !result.success) {
    alert(result.error || "Payment failed");
    return;
  }

  window.location.href =
    result.checkout_url;
}
```

---

# 12. Important: Unique Order IDs

Every payment should have a unique `order_id`.

Good:

```text
ORDER-1001
ORDER-1002
ORDER-1003
```

Or:

```javascript
const orderId = "ORDER-" + Date.now();
```

Do not reuse the same order ID for different payments.

The Central Payment API prevents duplicate orders for the same merchant.

---

# 13. Payment Status

Creating the payment does **not** mean that the customer has paid.

The initial status is normally:

```text
PENDING
```

Possible statuses are:

```text
PENDING
COMPLETED
FAILED
REVERSED
INVALID
```

Your website should only treat:

```text
COMPLETED
```

as a successful payment.

Do not mark an order as paid merely because the customer was redirected to PesaPal.

---

# 14. Security Rules

### Never expose your API key

Do not put this in:

* HTML
* frontend JavaScript
* GitHub repositories
* public source code
* browser localStorage
* browser cookies accessible to JavaScript

Keep it in your server environment variables.

### Never trust the customer's payment amount

For a real ecommerce website, the server should determine the correct price from your product/order database.

For example, don't rely solely on:

```json
{
  "amount": 1
}
```

sent by the customer's browser.

Your backend should verify the product and calculate the amount.

---

# 15. Recommended Customer Flow

A normal payment flow should look like:

```text
1. Customer selects product
          ↓
2. Customer enters phone number
          ↓
3. Website creates order
          ↓
4. Website calls your backend
          ↓
5. Backend calls Central Payment API
          ↓
6. Central API creates PesaPal payment
          ↓
7. API returns checkout_url
          ↓
8. Customer is redirected to PesaPal
          ↓
9. Customer completes payment
          ↓
10. Payment status is verified
          ↓
11. Order is marked COMPLETED
```

---

# 16. Testing

Start with a small test amount.

After clicking Pay Now, verify that:

1. Your backend returns HTTP `200`.
2. The response contains `success: true`.
3. The response contains `checkout_url`.
4. The customer is redirected to PesaPal.
5. The payment appears in your merchant dashboard.
6. The final payment status becomes `COMPLETED` after a successful payment.

---

# 17. API Key Rotation

If you believe your API key has been exposed:

1. Log in to your merchant dashboard.
2. Generate/rotate your API key.
3. Replace the old key in your server environment variables.
4. Redeploy your website.

The old key should no longer be used.

**Never send your API key to support staff or paste it into public code.**

---

# 18. Quick Integration Checklist

Before going live:

* [ ] Merchant account created
* [ ] API key generated
* [ ] API key stored as a server environment variable
* [ ] Central API endpoint configured
* [ ] Payment endpoint created
* [ ] Pay button connected
* [ ] `checkout_url` handled
* [ ] Unique order IDs implemented
* [ ] Payment status verification implemented
* [ ] Server validates product prices
* [ ] API key is not exposed in frontend code
* [ ] Successful payment tested

---

## Central API Endpoint

```text
POST https://lxrdkforhtiudmghydqx.supabase.co/functions/v1/central-create-payment
```

## Merchant API Header

```text
x-api-key: YOUR_MERCHANT_API_KEY
```

## Successful Payment Response

```json
{
  "success": true,
  "payment_id": "...",
  "order_tracking_id": "...",
  "merchant_reference": "...",
  "checkout_url": "https://pay.pesapal.com/..."
}
```

**The merchant's API key is the only credential required to authenticate payment requests to the Central Payment API. PesaPal credentials remain with the Central Payment API and must never be placed on merchant websites.**
