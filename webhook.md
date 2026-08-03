# Webhooks

NipaMail sends an HTTP webhook when the delivery status of a transactional message changes. Configure the webhook separately for each application in the NipaMail portal.

## Configure a webhook

1. Open **Application** in the NipaMail portal and select an application.
2. Open the **Webhook Configuration** tab.
3. Enter the URL of an endpoint that accepts HTTP `POST` requests. Use HTTPS for production endpoints.
4. Set the required `Authorization` header value.
5. Select **Verify & Save** to test and save the endpoint.

The endpoint must be publicly reachable by NipaMail. Keep the configured authorization value secret and validate it before processing a webhook.

## Webhook request

NipaMail sends a JSON payload to the configured endpoint.

```http
POST <your-webhook-url>
Content-Type: application/json
Authorization: <configured-authorization-value>
```

### Example payload

```json
{
  "id": "019fc698-55d4-7048-b46d-3bd4e4c0f3ae",
  "result": "Ok",
  "status": "Delivered",
  "sender": "No-Reply <noreply@example.com>",
  "recipient": "test@example.com",
  "cost": 1,
  "response": {
    "code": 250,
    "content": "OK"
  },
  "timestamp": 1785743300,
  "created": 1785743300
}
```

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Transactional message ID. |
| `result` | string | High-level result of the delivery attempt, for example `Ok`. |
| `status` | MessageLogStatus | Latest delivery status recorded for the message. |
| `sender` | string | Sender name and email address used for the message. |
| `recipient` | string | Recipient email address. |
| `cost` | number | Credit cost of the message. |
| `response.code` | number | SMTP or provider response code. |
| `response.content` | string | SMTP or provider response message. |
| `timestamp` | number | Time of the delivery event as a Unix timestamp in seconds. |
| `created` | number | Time the status event was recorded as a Unix timestamp in seconds. |

For possible `status` values, see [`MessageLogStatus` meaning](transactional.md?id=messagelogstatus-meaning).

## Acknowledge a webhook

Return a `2xx` HTTP status code after the webhook has been accepted for processing. Process each event idempotently because the same event can be delivered more than once.

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "received": true
}
```

Return the response promptly. For longer-running work, store or enqueue the event before returning the acknowledgement.

## Node.js / Express example

```js
const express = require("express");

const app = express();

app.use(express.json());

app.post("/webhooks/nipamail", (req, res) => {
  const authorization = req.header("Authorization");

  if (authorization !== process.env.NIPAMAIL_WEBHOOK_AUTHORIZATION) {
    return res.status(401).json({ message: "Invalid authorization" });
  }

  const event = req.body;

  // Store or enqueue the event before acknowledging it.
  console.log(event.id, event.status);

  return res.status(200).json({ received: true });
});
```

Set `NIPAMAIL_WEBHOOK_AUTHORIZATION` to the same value configured in the portal. Use a constant-time comparison instead of a direct string comparison when the authorization value is a security-sensitive secret.

## Integration checklist

| Check | Recommendation |
| --- | --- |
| Endpoint | Use a publicly reachable HTTPS URL. |
| Method | Accept HTTP `POST` requests. |
| Content type | Parse the request as `application/json`. |
| Authorization | Compare the header with the value configured in the portal. |
| Acknowledgement | Return a `2xx` response promptly after accepting the event. |
| Duplicate events | Use `id`, `status`, and `timestamp` to make processing idempotent. |
| Observability | Log delivery failures without logging the authorization value. |
