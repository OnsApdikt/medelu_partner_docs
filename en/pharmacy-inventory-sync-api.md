---
layout: default
title: Pharmacy inventory sync API
---

[Français](/pharmacy-inventory-sync-api.html)

# Pharmacy inventory sync API

Push pharmacy product stock and pricing from your inventory system into Medelu.

**Method:** `POST` only  
**URL:** `https://buwduscjszvjhotjrydd.supabase.co/functions/v1/pharmacy-inventory-sync`  
**Auth:** HMAC-SHA256 per inventory system (`verify_jwt` is `false`; no Supabase JWT required)

Medelu provides out of band:

- `hmac_secret` for your inventory system
- Your inventory system name for the `X-Inventory-System` header (case-sensitive; Medelu assigns this value; use it exactly as provided)
- Which pharmacy CNS codes are enabled for your integration on Medelu’s platform

---

## Required Headers

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `X-Inventory-System` | Your inventory system name as assigned by Medelu (must match exactly, case-sensitive) |
| `X-Inventory-Signature` | `timestamp=<unix_seconds>,signature=<hex_hmac>`. `timestamp` is when the request is prepared (now), not when inventory last changed in your system. |

---

## Request body

**When to set `timestamptz`:** Use the time when you **prepare and send this API request**, not when stock or prices last changed in your inventory system. The signature header `timestamp` and body `timestamptz` must both reflect that moment (within ±1 second). Because of the 5 minute request window, a timestamp from an older inventory export will be rejected.

JSON object:

```json
{
  "pharmacy_code": "811111-11",
  "timestamptz": "2026-06-26T14:30:00Z",
  "products": [
    {"cefip": "0144099", "price": 9.99, "stock": 6, "tva": 3.00},
    {"cefip": "0104906", "price": 12.45, "stock": 56, "tva": 17.00}
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `pharmacy_code` | string | Luxembourg pharmacy CNS code (`pharmacies.cns_code`) |
| `timestamptz` | string | ISO 8601 time when **this API request is prepared** (same instant as the signature `timestamp`, ±1s). **Do not** use when inventory last changed in your system.
| `products` | array | One or more product rows |
| `products[].cefip` | string | CEFIP product code (6-digit codes get a leading `0` normalized server-side) |
| `products[].price` | number | Current price |
| `products[].stock` | integer | Stock quantity (whole number) |
| `products[].tva` | number | VAT rate |

---

## Signing (HMAC-SHA256)

Set `timestamp` and `timestamptz` at the moment you build the request (right before signing and sending). Do not reuse timestamps from your inventory database or export files.

1. Choose `timestamp` = **now**, as **integer Unix seconds** when preparing the request (not milliseconds, not the inventory’s last-modified time).
2. Set `timestamptz` in the JSON body to the same instant as `timestamp` (±1 second).
3. Serialize the JSON body to a **single string** (exact bytes you will send in the HTTP body).
4. Build the signed payload:

   ```
   signed_payload = "<timestamp>" + "." + <raw JSON body>
   ```

5. Compute:

   ```
   signature = HMAC_SHA256(hmac_secret, signed_payload) → lowercase hex
   ```

6. Send header:

   ```
   X-Inventory-Signature: timestamp=<timestamp>,signature=<signature>
   ```

### Timestamp rules

| Check | Rule |
|-------|------|
| Header `timestamp` format | Digits only, integer Unix **seconds** at request preparation time (not milliseconds, e.g. values ≥ `1000000000000`) |
| Request validity window | Header `timestamp` must be within **5 minutes (300 seconds)** of Medelu server time, both past and future |
| Body vs header | `timestamptz` must be set when preparing the request and match header `timestamp` within **±1 second** |

### 5-minute request window (replay protection)

Each request includes a `timestamp` in `X-Inventory-Signature`. This is the time the request was **signed and sent**, not when your inventory data last changed. Medelu compares it to the current server time when the request arrives.

- **Allowed:** `timestamp` is at most **5 minutes older** or **5 minutes newer** than server time (300 seconds each way).
- **Rejected:** If the timestamp is more than 5 minutes in the past or more than 5 minutes in the future, the API returns **`400`** with message `Request timestamp outside allowed window`.
- **Implication:** Pick `timestamp` and `timestamptz` immediately before signing. Compute the HMAC and send the HTTP request right away. Do not queue, batch-sign, or retry the same signed payload later.
- **Implication:** You cannot reuse a previously signed request body + signature; each sync must be signed fresh with a current timestamp.
- **Common mistake:** Using the inventory system’s “last updated” or export timestamp will fail if that time is more than 5 minutes ago. Always stamp the request at send time.

The 5-minute window applies only to the header `timestamp`. The body field `timestamptz` is not checked against server time directly. It must only match the header `timestamp` within ±1 second, and both must be set when preparing the request.

**Important:** The HMAC is computed over the **exact raw request body bytes**. Whitespace, key order, and number formatting must be identical between signing and sending. Pretty-printed JSON with different spacing will fail signature verification.

---

## Responses

All responses are JSON with `Content-Type: application/json`.

### Success (`200`)

When at least one product is upserted:

```json
{
  "success": true,
  "message": "Inventory synced successfully",
  "processed_products": 2
}
```

When every CEFIP in the payload is unknown to Medelu (silently skipped) or the `products` array is empty after filtering:

```json
{
  "success": true,
  "message": "No products to sync",
  "processed_products": 0
}
```

### Error status codes

Each row is an exact `message` value returned by the API.

| HTTP | `message` | When |
|------|-----------|------|
| `400` | `timestamp must be unix seconds (integer, no decimals, not milliseconds)` | `X-Inventory-Signature` `timestamp` is not valid integer Unix seconds |
| `400` | `Request timestamp outside allowed window` | Header `timestamp` is more than 5 minutes before or after Medelu server time (often caused by using an old inventory timestamp instead of request preparation time) |
| `400` | `Invalid JSON body` | Request body is not valid JSON |
| `400` | `Invalid request format. Required: pharmacy_code, timestamptz, products array` | JSON parsed but missing required top-level fields |
| `400` | `timestamptz does not match signature timestamp` | Body `timestamptz` does not match header `timestamp` within ±1 second (both must be set at request preparation time) |
| `400` | `Invalid product payload` | One or more product fields invalid; response includes an `errors` string array |
| `401` | `unauthorized` | Missing `X-Inventory-System` or `X-Inventory-Signature`; malformed signature header; unknown, disabled, or misconfigured inventory system |
| `401` | `unauthorized: invalid signature` | HMAC signature does not match the request body and timestamp |
| `403` | `Pharmacy not authorized for this inventory system` | `pharmacy_code` exists but is not linked to your inventory system on Medelu’s side |
| `404` | `Pharmacy with provided CNS code not found` | No pharmacy with that `pharmacy_code` |
| `405` | `Method not allowed` | Request method is not `POST` |
| `500` | `Error validating products` | Server failed while checking CEFIP codes |
| `500` | `Error updating inventory` | Server failed while writing inventory data |
| `500` | `Internal server error` | Unexpected server error |

Example `400` with product validation errors:

```json
{
  "success": false,
  "message": "Invalid product payload",
  "errors": [
    "products[0].stock must be a valid whole number"
  ]
}
```

---

## Examples

Replace `YOUR_HMAC_SECRET` and `YOUR_INVENTORY_SYSTEM_NAME` with the values Medelu shared with you.

`timestamp` and `timestamptz` are set to **now** when you run the example (request preparation time), not an inventory last-modified time.

<div class="code-tabs" data-code-tabs>
  <div class="code-tabs__bar" role="tablist" aria-label="Code examples">
    <button type="button" class="code-tabs__tab" role="tab" aria-selected="true" data-tab-target="curl">curl (macOS / Linux)</button>
    <button type="button" class="code-tabs__tab" role="tab" aria-selected="false" data-tab-target="nodejs">Node.js</button>
  </div>

  <div class="code-tabs__panel" role="tabpanel" data-tab-panel="curl">
    <p>On Linux, replace <code>date -u -r "$TIMESTAMP"</code> with <code>date -u -d "@$TIMESTAMP"</code>.</p>
{% highlight bash linenos %}
SECRET='YOUR_HMAC_SECRET'
INVENTORY_SYSTEM='YOUR_INVENTORY_SYSTEM_NAME'
TIMESTAMP=$(date +%s)
TIMESTAMPTZ=$(date -u -r "$TIMESTAMP" '+%Y-%m-%dT%H:%M:%SZ')
BODY=$(printf '{"pharmacy_code":"811111-11","timestamptz":"%s","products":[{"cefip":"0144099","price":9.99,"stock":6,"tva":3.00},{"cefip":"0104906","price":12.45,"stock":56,"tva":17.00}]}' "$TIMESTAMPTZ")
HMAC_SIGNATURE=$(printf '%s.%s' "$TIMESTAMP" "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -i --request POST 'https://buwduscjszvjhotjrydd.supabase.co/functions/v1/pharmacy-inventory-sync' \
  --header "X-Inventory-System: ${INVENTORY_SYSTEM}" \
  --header "X-Inventory-Signature: timestamp=${TIMESTAMP},signature=${HMAC_SIGNATURE}" \
  --header 'Content-Type: application/json' \
  --data "$BODY"
{% endhighlight %}
  </div>

  <div class="code-tabs__panel" role="tabpanel" data-tab-panel="nodejs" hidden>
    <p>Uses the built-in <code>crypto</code> module. <code>bodyString</code> must be the exact string sent as the request body.</p>
{% highlight javascript linenos %}
import crypto from "node:crypto";
// or const crypto = require("node:crypto");

const HMAC_SECRET = process.env.INVENTORY_HMAC_SECRET; // shared out of band
const INVENTORY_SYSTEM = process.env.INVENTORY_SYSTEM_NAME; // assigned by Medelu
const URL =
  "https://buwduscjszvjhotjrydd.supabase.co/functions/v1/pharmacy-inventory-sync";

function signInventoryRequest(secret, timestamp, bodyString) {
  const signedPayload = `${timestamp}.${bodyString}`;
  return crypto
    .createHmac("sha256", secret)
    .update(signedPayload, "utf8")
    .digest("hex");
}

async function syncInventory() {
  const timestamp = Math.floor(Date.now() / 1000); // now, when preparing the request
  const timestamptz = new Date(timestamp * 1000).toISOString(); // same instant as timestamp

  const body = {
    pharmacy_code: "811111-11",
    timestamptz,
    products: [
      { cefip: "0144099", price: 9.99, stock: 6, tva: 3.0 },
      { cefip: "0104906", price: 12.45, stock: 56, tva: 17.0 },
    ],
  };

  // Stable JSON: no extra spaces; same string for HMAC and fetch body
  const bodyString = JSON.stringify(body);
  const signature = signInventoryRequest(HMAC_SECRET, timestamp, bodyString);

  const response = await fetch(URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Inventory-System": INVENTORY_SYSTEM,
      "X-Inventory-Signature": `timestamp=${timestamp},signature=${signature}`,
    },
    body: bodyString,
  });

  const result = await response.json();
  console.log(response.status, result);
}

syncInventory().catch(console.error);
{% endhighlight %}
  </div>
</div>
