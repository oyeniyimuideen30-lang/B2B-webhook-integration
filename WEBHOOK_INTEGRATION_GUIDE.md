# WayaBank B2B Webhook Integration Guide

This document outlines the technical requirements for merchants to integrate with WayaBank's unified webhook system.

## 1. Overview
WayaBank uses webhooks to notify your system of events in real-time. Instead of multiple URLs, we use a single **Unified Webhook URL** for all product events (Virtual Accounts, Transfers, etc.).

## 2. Shared Secret (Security)
All webhooks are signed using your **Live API Key** as the HMAC secret. 
> [!IMPORTANT]
> Keep your Live API Key secure. If it is compromised, rotate it immediately and update your webhook verification logic.

## 3. Webhook Request Format
WayaBank will send a `POST` request to your configured Webhook URL with the following headers and JSON body.

### HTTP Headers
| Header | Description |
| :--- | :--- |
| `Content-Type` | `application/json` |
| `x-waya-signature` | HMAC SHA-512 signature of the raw request body. |
| `x-waya-timestamp` | Unix timestamp (seconds) of when the request was generated. |

### JSON Payload Structure
```json
{
  "eventId": "evt_123abc456def",
  "eventType": "virtual_account.credit",
  "productCode": "VIRTUAL-ACCOUNT",
  "merchantId": "YOUR_MERCHANT_ID",
  "timestamp": "2026-03-02T14:15:00",
  "data": { ... }
}
```

## 4. Event Schemas (Product Specific)
The `data` object's structure varies based on the `productCode` and `eventType`.

### 4.1. Virtual Account Credit (`productCode: VIRTUAL-ACCOUNT`)
Sent when a virtual account receives an inflow.
```json
"data": {
  "accountNumber": "0123456789",
  "amount": 5000.00,
  "currency": "NGN",
  "bankName": "WayaBank",
  "senderName": "JOHN DOE",
  "sessionID": "000013230226123456789",
  "transactionReference": "WAYA-12345-ABC"
}
```

### 4.2. TSQ - Transaction Status Query (`productCode: TSQ`)
Sent when a status query update is triggered for a pending transaction.
```json
"data": {
  "originalReference": "TRANS-999-XYZ",
  "currentStatus": "FAILED",
  "failureReason": "Incorrect beneficiary details",
  "lastCheckedAt": "2026-03-02T14:20:00"
}
```

## 5. Signature Verification (Implementation)
To ensure the request came from WayaBank and has not been tampered with, you **must** verify the `x-waya-signature`.

### Verification Steps:
1. Retrieve the raw JSON request body.
2. Retrieve the `x-waya-signature` header.
3. Compute an HMAC SHA-512 hash of the raw body using your **Live API Key** as the key.
4. Compare your computed hash with the header value. They must match exactly.

### Example (Node.js/JavaScript):
```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
    const computedSignature = crypto
        .createHmac('sha512', secret)
        .update(payload) // payload must be the raw string body
        .digest('hex');
    
    return computedSignature === signature;
}
```

### Example (Java):
```java
public boolean verifySignature(String payload, String signature, String secret) throws Exception {
    Mac hmacSha512 = Mac.getInstance("HmacSHA512");
    SecretKeySpec secretKey = new SecretKeySpec(secret.getBytes(), "HmacSHA512");
    hmacSha512.init(secretKey);
    
    byte[] hash = hmacSha512.doFinal(payload.getBytes());
    String computedSignature = bytesToHex(hash);
    
    return computedSignature.equalsIgnoreCase(signature);
}
```

## 5. Best Practices
- **Respond Quickly**: Your endpoint should return an `HTTP 200 OK` as soon as the signature is verified. Process heavy logic asynchronously.
- **Idempotency**: Use `eventId` to prevent processing the same notification twice.
- **HTTPS Only**: WayaBank only supports `https://` endpoints for webhooks for data security.
- **Timestamp Check**: (Optional) Verify that the `x-waya-timestamp` is within a reasonable window (e.g., last 5 minutes) to prevent replay attacks.
