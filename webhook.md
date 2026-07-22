# Webhook

NipaMail sends webhooks for transactional message status changes when your application has a registered and verified webhook URL.

## Delivery

NipaMail sends an HTTP `POST` request to your configured webhook URL.

### Request headers

| Field | Required | Description |
| --- | --- | --- |
| `Content-Type` | Yes | Always `application/json`. |
| `Authorization` | Yes | The authorization value configured on the webhook, forwarded as-is. |
| `X-Nipa-Signature` | Conditional | HMAC-SHA256 signature of the JSON request body. Present when `client_secret` is available for the credential used to send the message. |

### Request body

```json
{
    "id": "<message id>",
    "status": "<message status>",
    "result": "<message result>",
    "sender": "Sender <sender@example.com>",
    "recipient": "recipient@example.com",
    "cost": "1.00",
    "response": {
        "code": 250,
        "content": "<provider response>"
    },
    "timestamp": 1767225600000,
    "created": 1767225600000
}
```

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Transactional message ID. |
| `status` | MessageLogStatus | Latest delivery status recorded for the message. |
| `result` | EmailMessageResult | High-level result: `Processing`, `Ok`, `HardBounce`, `SoftBounce`, or `Error`. |
| `sender` | string | Sender email/name originally used. |
| `recipient` | string | Recipient email address. |
| `cost` | string | Credit cost formatted with two decimal places. |
| `response.code` | number | Transport/MTA response code when available. |
| `response.content` | string | Provider response content when available. |
| `timestamp` | number | Event timestamp from provider (epoch ms). |
| `created` | number | Timestamp when the status log was created (epoch ms). |

For possible `status` values, see the [`MessageLogStatus` meaning](/transactional.md?id=messagelogstatus-meaning) page.

## Webhook response

Return any `2xx` HTTP status code to acknowledge the webhook.

NipaMail follows redirects up to 5 times and waits up to 10 seconds for each request. If the request fails, times out, or receives a non-success response, NipaMail retries up to 3 attempts with exponential backoff starting at 5 seconds.

### Recommended response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
    "received": true
}
```

Your response body is only stored for webhook delivery logs; it does not change message delivery status.

## Verify HMAC signature

When `X-Nipa-Signature` is present, verify that the signature sent by NipaMail is valid before processing the webhook.

NipaMail creates the `X-Nipa-Signature` value as:

```text
hex(HMAC_SHA256(client_secret, raw_json_body))
```

Important details:

| Item | Description |
| --- | --- |
| Algorithm | HMAC-SHA256 |
| Header | `X-Nipa-Signature` |
| Secret | The application credential `client_secret` |
| Signed payload | The exact raw JSON request body sent by NipaMail |
| Encoding | Lowercase hexadecimal digest |

`client_secret` is available from the application setting for the credential used to send the message. Use `client_secret` to verify the signature received from NipaMail.

### Node.js / Express example

```js
const crypto = require("crypto");
const express = require("express");

const app = express();

app.post(
    "/webhooks/nipamail",
    express.raw({ type: "application/json" }),
    (req, res) => {
        const nipaSignature = req.header("X-Nipa-Signature");
        const ClientSecret = process.env.NIPAMAIL_CLIENT_SECRET;

        if (!verifyNipaSignature(nipaSignature, req.body, ClientSecret)) {
            return res.status(401).json({ message: "Invalid signature" });
        }

        const event = JSON.parse(req.body.toString("utf8"));

        // Process the webhook event.
        res.status(200).json({ received: true });
    }
);

function verifyNipaSignature(nipaSignature, rawBody, ClientSecret) {
    const expectedSignature = crypto
        .createHmac("sha256", ClientSecret)
        .update(rawBody)
        .digest("hex");

    const signatureBuffer = Buffer.from(nipaSignature || "", "hex");
    const expectedBuffer = Buffer.from(expectedSignature, "hex");

    return signatureBuffer.length === expectedBuffer.length &&
        crypto.timingSafeEqual(signatureBuffer, expectedBuffer);
}
```

### Python / Flask example

```python
import hashlib
import hmac
import json
import os
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.post("/webhooks/nipamail")
def nipamail_webhook():
    nipa_signature = request.headers.get("X-Nipa-Signature", "")
    ClientSecret = os.environ["NIPAMAIL_CLIENT_SECRET"].encode("utf-8")
    raw_body = request.get_data()

    if not verify_nipa_signature(nipa_signature, raw_body, ClientSecret):
        return jsonify({"message": "Invalid signature"}), 401

    event = json.loads(raw_body)

    # Process the webhook event.
    return jsonify({"received": True}), 200

def verify_nipa_signature(nipa_signature, raw_body, ClientSecret):
    expected_signature = hmac.new(ClientSecret, raw_body, hashlib.sha256).hexdigest()

    return hmac.compare_digest(nipa_signature, expected_signature)
```

### Java / Spring Boot example

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class NipaMailWebhookController {
    private final ObjectMapper objectMapper = new ObjectMapper();
    private final String ClientSecret = System.getenv("NIPAMAIL_CLIENT_SECRET");

    @PostMapping("/webhooks/nipamail")
    public ResponseEntity<?> receiveWebhook(
            @RequestHeader(value = "X-Nipa-Signature", required = false) String nipaSignature,
            HttpServletRequest request
    ) throws Exception {
        byte[] rawBody = request.getInputStream().readAllBytes();

        if (!verifyNipaSignature(nipaSignature, rawBody, ClientSecret)) {
            return ResponseEntity.status(401).body("{\"message\":\"Invalid signature\"}");
        }

        JsonNode event = objectMapper.readTree(rawBody);

        // Process the webhook event.
        return ResponseEntity.ok("{\"received\":true}");
    }

    private static boolean verifyNipaSignature(
            String nipaSignature,
            byte[] rawBody,
            String ClientSecret
    ) throws NoSuchAlgorithmException, InvalidKeyException {
        String expectedSignature = hmacSha256Hex(ClientSecret, rawBody);

        return timingSafeEquals(nipaSignature, expectedSignature);
    }

    private static String hmacSha256Hex(String secret, byte[] payload)
            throws NoSuchAlgorithmException, InvalidKeyException {
        Mac mac = Mac.getInstance("HmacSHA256");
        SecretKeySpec keySpec = new SecretKeySpec(
                secret.getBytes(StandardCharsets.UTF_8),
                "HmacSHA256"
        );
        mac.init(keySpec);
        return HexFormat.of().formatHex(mac.doFinal(payload));
    }

    private static boolean timingSafeEquals(String signature, String expected) {
        if (signature == null) {
            return false;
        }

        byte[] signatureBytes;
        byte[] expectedBytes;

        try {
            signatureBytes = HexFormat.of().parseHex(signature);
            expectedBytes = HexFormat.of().parseHex(expected);
        } catch (IllegalArgumentException ex) {
            return false;
        }

        return signatureBytes.length == expectedBytes.length
                && MessageDigest.isEqual(signatureBytes, expectedBytes);
    }
}
```

For Spring Boot 2, replace `jakarta.servlet.http.HttpServletRequest` with `javax.servlet.http.HttpServletRequest`.

### Verification checklist

| Check | Why it matters |
| --- | --- |
| Read the raw body before JSON parsing | Whitespace and key order affect the HMAC digest. |
| Compare signatures with a timing-safe comparison | Prevents leaking signature details through timing differences. |
| Return `401` for invalid signatures | Prevents forged webhook events from being processed. |
| Return `2xx` only after durable processing | NipaMail treats `2xx` as accepted and will not retry. |
