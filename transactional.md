# Transactional message sending

## Send the transactional message

### Request
```http
POST /v1/transactional/messages
```

#### Headers
| Field | Required | Description |
| --- | --- | --- |
| `Content-Type` | Yes | `application/json` |
| `Authorization` | Yes | Use `Bearer <TOKEN>` |

#### Request body
```json
{
  "type": "EMAIL" | "SMS",
  "message": {
    "sender": "<sender name>",
    "recipient": "<target email>",
    "subject": "<string>",
    "template_id": "<template identifier>",     // required when html is absent
    "html": "<base64 encoded html>",            // required when template_id is absent
    "template_values": {
      "[key]": "value"
    },
    "attachments": [
      {
        "type": "ASSET" | "RAW",
        "content": "<base64 payload or asset id>",
        "file_name": "<download name>"
      }
    ],
  }
}
```

| Field | Required | Description |
| --- | --- | --- |
| `type` | Yes | `EMAIL` for email message |
| `message.sender` | Yes | Sender domain registered. |
| `message.recipient` | Yes | Single recipient email address. |
| `message.subject` | Yes | Must satisfy these conditions: It should not be empty, contain emojis, or start with 'Re:' or 'Fwd:'. |
| `message.template_id` | Conditional | Provide when not supplying `message.html`; references stored template. |
| `message.html` | Conditional | Base64-encoded HTML content when no template is referenced. |
| `message.template_values` | No | Key/value map injected into template rendering. |
| `message.attachments[].type` | No | Determines the `content` type: `ASSET` references asset ID from the asset manager, `RAW` uses inlined base64. |
| `message.attachments[].content` | No | Payload or asset identifier depending on `type`. |
| `message.attachments[].file_name` | No | Friendly download filename surfaced to recipients. |

---

### Response

**201 – Created**

Message created.

```json
{
  "id": "<message id>",
  "status": "<status>",
  "updated_at": "<timestamp>",
  "cost": <number>
}
```
| Field | Description |
| --- | --- |
| `id` | Transactional message ID |
| `status` | Message status |
| `updated_at` | Timestamp of latest updated. |
| `cost` | Credit consumed (stored as Integer, multiply by 100). |

**400 – Bad Request**

When a malformed request body supplied, throws `ValidationError`.

```json
{
  "type": "ValidationError",
  "errors": [
    {
      "target": {
        "type": "PUSH"
      },
      "property": "type",
      "children": [],
      "constraints": {
        "isOneOf": "type must be one of 'EMAIL', 'SMS'"
      }
    }
  ]
}
```

**401 – Unauthorized**

The specified credentials is invalid, or restricted by CIDR.

```json
{
  "type": "UnauthorizedError",
  "message": "Invalid credentials or restricted by CIDR."
}
```

**404 – Not Found**

When the `template_id` specified is not exists, throws `TemplateNotFoundError`. 

```json
{
  "type": "TemplateNotFoundError",
  "message": "The template specified could not be found."
}
```

**404 – Not Found**

When one or more attachments referenced in the `attachments[].content` with type `ASSET` is not exists in the asset manager, throws `AssetNotFoundError`. 

```json
{
  "type": "AssetNotFoundError",
  "message": "One or more of the specified assets could not be found."
}
```

**406 – Not Acceptable**

When the sender domain specified is not one of registered sender domains, throws `InvalidSenderError`.

```json
{
  "type": "InvalidSenderError",
  "message": "The specified sender domain is not registered."
}
```

**406 – Not Acceptable**

When the credit balance on the account is insufficient, throws `InsufficientCreditError`.

```json
{
  "type": "InsufficientCreditError",
  "message": "Your credit balance is insufficient."
}
```


**406 – Not Acceptable**

When the total size of attachment specified exceeds the attachment size limit, throws `TotalAttachmentSizeExceededError`.

```json
{
  "type": "TotalAttachmentSizeExceededError",
  "message": "The total size of attachments exceeds the allowed limit."
}
```

**500 – Internal Server Error**

When catastrophic errors occured, throws `InternalServerError`

```json
{
  "type": "InternalServerError",
  "message": "Something went wrong, please try again later."
}
```

### Example: inline HTML with `curl`

Encode your HTML in base64 (no newlines) and place it in `message.html`.

```bash
curl -X POST https://api.nipamail.com/v1/transactions \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "EMAIL",
    "message": {
      "sender": "<sender display name> <<sender@yourdomain.com>>",
      "recipient": "<recipient@example.com>",
      "subject": "<subject line>",
      "html": "<base64-encoded-html>"
    }
  }'
```

### Example: template with variables using `curl`

Provide a stored `template_id` and pass values to render inside the template.

```bash
curl -X POST https://api.nipamail.com/v1/transactions \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "EMAIL",
    "message": {
      "sender": "<sender display name> <<sender@yourdomain.com>>",
      "recipient": "<recipient@example.com>",
      "subject": "<subject line>",
      "template_id": "<template id>",
      "template_values": {
        "first_name": "<first name>",
        "cta_url": "<cta url>"
      }
    }
  }'
```

### Example: template with variables and asset attachment using `curl`

Reference an uploaded asset by ID for attachments.

```bash
curl -X POST https://api.nipamail.com/v1/transactions \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "EMAIL",
    "message": {
      "sender": "<sender display name> <<sender@yourdomain.com>>",
      "recipient": "<recipient@example.com>",
      "subject": "<subject line>",
      "template_id": "<template id>",
      "template_values": {
        "full_name": "<full name>",
        "statement_month": "<statement month>"
      },
      "attachments": [
        {
          "type": "ASSET",
          "content": "<asset id>",
          "file_name": "<attachment name>"
        }
      ]
    }
  }'
```

### Example: template with variables and raw base64 attachment using `curl`

Inline a file by base64-encoding it and setting `attachments[].type` to `RAW`.

```bash
curl -X POST https://api.nipamail.com/v1/transactions \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "EMAIL",
    "message": {
      "sender": "<sender display name> <<sender@yourdomain.com>>",
      "recipient": "<recipient@example.com>",
      "subject": "<subject line>",
      "template_id": "<template id>",
      "template_values": {
        "full_name": "<full name>",
        "statement_month": "<statement month>"
      },
      "attachments": [
        {
          "type": "RAW",
          "content": "<base64-encoded-file>",
          "file_name": "<attachment name>"
        }
      ]
    }
  }'
```
---
## Inquiry the transactional message status
### Request
```http
GET /v1/transactional/messages/{transactional_message_id}
```
#### Path parameters
| Field | Required | Description |
| --- | --- | --- |
| `transactional_message_id` | Yes | Targeted transactional message ID |
#### Headers
| Field | Required | Description |
| --- | --- | --- |
| `Authorization` | Yes | Use `Bearer <TOKEN>` |

### Response

**200 – OK**

Returns the targeted transactional message detail.

```json
{
  "id": "<message id>",
  "status": "<status>",
  "sender": "<sender name> <<sender@example.com>>",
  "recipient": "<recipient@example.com>",
  "opened": <number>,
  "clicked": <number>,
  "cost": <number>,
  "created_at": "<timestamp>",
  "updated_at": "<timestamp>",
  "status_logs": [
    {
      "status": "<status>",
      "response": {
        "code": <number>,
        "content": "<provider response>"
      },
      "timestamp": <epoch-ms>,
      "created": <epoch-ms>,
      "error_message": "<error message | null>"
    },
    {
      "status": "<status>",
      "response": {
        "code": <number>,
        "content": "<provider response>"
      },
      "timestamp": <epoch-ms>,
      "created": <epoch-ms>,
      "error_message": "<error message | null>"
    },
    {
      "status": "<status>",
      "response": {
        "code": <number>,
        "content": "<provider response>"
      },
      "timestamp": <epoch-ms>,
      "created": <epoch-ms>,
      "error_message": "<error message | null>"
    }
  ]
}

```
| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Unique message log identifier returned in `MessageLogDetailResponse`. |
| `status` | MessageLogStatus | Latest delivery status recorded for the message. |
| `sender` | string | Sender email/name originally used. |
| `recipient` | string | Recipient email address. |
| `opened` | number | Total tracked open events. |
| `clicked` | number | Total tracked click events. |
| `cost` | number | Credit cost incurred for delivery, stored in an integer (multiply by 100) |
| `created_at` | Date | When the log entry was created. |
| `updated_at` | Date | When the log entry last changed. |
| `status_logs` | StatusLogResponse[] | Chronological list of delivery state changes. |
| `status_logs[].status` | MessageLogStatus | Status recorded for the log entry. |
| `status_logs[].response.code` | number | Transport/MTA response code. |
| `status_logs[].response.content` | string | Raw provider response body. |
| `status_logs[].timestamp` | number | Event timestamp from provider (epoch ms). |
| `status_logs[].created` | number | Timestamp when the status log was persisted (epoch ms). |
| `status_logs[].error_message` | string | Optional error message describing the failure cause. |

#### `MessageLogStatus` meaning
| Status | Meaning |
| --- | --- |
| `Created` | Message request persisted but not yet submitted to the provider. |
| `Submitting` | Currently being handed off to the provider. |
| `Accepted` | Provider acknowledged receipt of the message. |
| `Delivered` | Provider confirmed the recipient’s server accepted the message. |
| `HardBounce` | Permanent failure (invalid address, policy rejection). |
| `SoftBounce` | Temporary failure (mailbox full, transient error). |
| `ProviderRejected` | Provider actively rejected the message outside of bounce semantics. |
| `Deferred` | Provider deferred the message; will retry later. |
| `Spam` | Provider flagged the message as spam. |
| `TimedOut` | Delivery attempts timed out before completion. |
| `Feedback` | Recipient issued a feedback/complaint (feedback loop). |
| `Delayed` | Message is intentionally delayed or postponed by the system/provider. |
| `TechnicalError` | Internal system error prevented sending. |
| `InsufficientCredit` | Sending failed because the tenant lacks credits. |
| `InvalidRecipient` | Provided recipient address failed validation. |

**401 – Unauthorized**

The specified credentials is invalid, or restricted by CIDR.

```json
{
  "type": "UnauthorizedError",
  "message": "Invalid credentials or restricted by CIDR."
}
```

**404 – Not Found**

When the targeted transactional message is not exists, throws `MessageNotFoundError`. 

```json
{
  "type": "MessageNotFoundError",
  "message": "The message specified could not be found."
}
```
