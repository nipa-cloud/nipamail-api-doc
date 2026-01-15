# Transactional message sending

## Send the transactional message

### Request

```http
POST /v1/messages
```

#### Headers

| Field           | Required | Description          |
| --------------- | -------- | -------------------- |
| `Content-Type`  | Yes      | `application/json`   |
| `Authorization` | Yes      | Use `Bearer <TOKEN>` |

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
    ]
  }
}
```

| Field                             | Required    | Description                                                                                                   |
| --------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------- |
| `type`                            | Yes         | `EMAIL` for email message                                                                                     |
| `message.sender`                  | Yes         | Sender domain registered.                                                                                     |
| `message.recipient`               | Yes         | Single recipient email address.                                                                               |
| `message.subject`                 | Yes         | Must satisfy these conditions: It should not be empty, contain emojis, or start with 'Re:' or 'Fwd:'.         |
| `message.template_id`             | Conditional | Provide when not supplying `message.html`; references stored template.                                        |
| `message.html`                    | Conditional | Base64-encoded HTML content when no template is referenced.                                                   |
| `message.template_values`         | No          | Key/value map injected into template rendering.                                                               |
| `message.attachments[].type`      | No          | Determines the `content` type: `ASSET` references asset ID from the asset manager, `RAW` uses inlined base64. |
| `message.attachments[].content`   | No          | Payload or asset identifier depending on `type`.                                                              |
| `message.attachments[].file_name` | No          | Friendly download filename surfaced to recipients.                                                            |

---

### Response

**201 – Created**

Message created.

#### Headers
| Field | Description |
| --- | --- |
| `x-ratelimit-remaining` | How many requests you can still make before hitting the limit. |
| `x-ratelimit-reset` | Epoch time (or seconds until) when the current window resets. |
| `x-ratelimit-limit` | Total number of requests allowed in the current window. |
| `retry-after` | The time (in seconds) that the client should wait before retrying the request. |

```json
{
  "id": "<message id>",
  "status": "<status>",
  "updated_at": "<timestamp>",
  "cost": <number>
}
```

| Field        | Description                                           |
| ------------ | ----------------------------------------------------- |
| `id`         | Transactional message ID                              |
| `status`     | Message status                                        |
| `updated_at` | Timestamp of latest updated.                          |
| `cost`       | Credit consumed (stored as Integer, multiply by 100). |

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
curl -X POST https://api.nipamail.com/v1/messages \
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
curl -X POST https://api.nipamail.com/v1/messages \
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
curl -X POST https://api.nipamail.com/v1/messages \
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
curl -X POST https://api.nipamail.com/v1/messages \
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
GET /v1/messages/{transactional_message_id}
```

#### Path parameters

| Field                      | Required | Description                       |
| -------------------------- | -------- | --------------------------------- |
| `transactional_message_id` | Yes      | Targeted transactional message ID |

#### Headers

| Field           | Required | Description          |
| --------------- | -------- | -------------------- |
| `Authorization` | Yes      | Use `Bearer <TOKEN>` |

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

| Field                            | Type                | Description                                                               |
| -------------------------------- | ------------------- | ------------------------------------------------------------------------- |
| `id`                             | string              | Unique message log identifier returned in `MessageLogDetailResponse`.     |
| `status`                         | MessageLogStatus    | Latest delivery status recorded for the message.                          |
| `sender`                         | string              | Sender email/name originally used.                                        |
| `recipient`                      | string              | Recipient email address.                                                  |
| `opened`                         | number              | Total tracked open events.                                                |
| `clicked`                        | number              | Total tracked click events.                                               |
| `cost`                           | number              | Credit cost incurred for delivery, stored in an integer                   |
| `created_at`                     | Date                | When the log entry was created.                                           |
| `updated_at`                     | Date                | When the log entry last changed.                                          |
| `status_logs`                    | StatusLogResponse[] | Chronological list of delivery state changes.                             |
| `status_logs[].status`           | MessageLogStatus    | Status recorded for the log entry.                                        |
| `status_logs[].response.code`    | number              | Transport/MTA response code.                                              |
| `status_logs[].response.content` | string              | Raw provider response body.                                               |
| `status_logs[].timestamp`        | number              | Event timestamp from provider (epoch ms).                                 |
| `status_logs[].created`          | number              | Timestamp when the status log was persisted (epoch ms).                   |
| `status_logs[].error_message`    | string              | Optional error message describing the failure cause.                      |

#### `MessageLogStatus` meaning

| Status               | Meaning                                                               |
| -------------------- | --------------------------------------------------------------------- |
| `Created`            | Message request persisted but not yet submitted to the provider.      |
| `Accepted`           | Provider acknowledged receipt of the message.                         |
| `Delivered`          | Provider confirmed the recipient’s server accepted the message.       |
| `Deferred`           | Provider deferred the message; will retry later.                      |
| `TimedOut`           | Delivery attempts timed out before completion.                        |
| `Feedback`           | Recipient issued a feedback/complaint (feedback loop).                |
| `TechnicalError`     | Internal system error prevented sending.                              |
| `InsufficientCredit` | Sending failed because the tenant lacks credits.                      |
| `Blocked`            | The recipient is blocked by our blocklist.                            |

**Bounce statuses**

Messages may encounter issues categorized as bounces.

| Status                 | Meaning                                                                 | Bounce type | Retry allowed | Credit charged             |
| ---------------------- | ----------------------------------------------------------------------- | ----------- | ------------- | -------------------------- |
| `InvalidRecipient`     | Recipient address is invalid or the mailbox does not exist.             | Hard        | No            | Yes*                       |
| `BadDomain`            | Recipient domain is misconfigured or missing valid MX records.          | Hard        | No            | No                         |
| `InactiveMailbox`      | Mailbox exists but is disabled, suspended, or closed.                   | Hard        | No            | Yes*                       |
| `InvalidSender`        | Sender/domain is not allowed or fails sender validation.                | Hard        | No            | Yes                        |
| `QuotaIssues`          | Recipient mailbox is full or exceeds storage/attachment limits.         | Soft        | Yes           | Yes                        |
| `NoAnswerFromHost`     | Remote host did not respond during delivery attempts.                   | Hard        | No            | No                         |
| `BadConnection`        | Connection to the recipient host failed or was dropped.                 | Soft        | Yes           | No                         |
| `DNSFailure`           | DNS lookup for the recipient domain failed (e.g., no MX).               | Hard        | No            | No                         |
| `RoutingErrors`        | Provider refused to relay or could not route to the recipient.          | Soft        | Yes           | No                         |
| `MessageExpired`       | Delivery window elapsed before the provider could deliver.              | Hard        | No            | No                         |
| `ProtocolErrors`       | SMTP command sequence or syntax was rejected.                           | Soft        | Yes           | No                         |
| `AuthenticationFailed` | Authentication/DMARC/SPF checks failed for the sender.                  | Soft        | Yes           | No                         |
| `PolicyRelated`        | Rejected by provider policy or acceptable use restrictions.             | Soft        | No            | Yes                        |
| `SpamContent`          | Message content was classified as spam or virus.                        | Soft        | No            | Yes                        |
| `SpamBlock`            | Delivery blocked due to sender/IP/domain reputation or blocklists.      | Soft        | Yes           | Yes                        |
| `ProviderRejected`     | Provider actively rejected the message outside of bounce semantics.     | Hard        | No            | Yes                        |

*Credit charge occurs on the first encounter of the message recipient, next encounter will be blocked by our blocklist. 

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

## Resend the transactional message

**Tier: Enterprise**

This API ensures that the message is sent again using the same request body
as the initial send. It is useful in scenarios where the previous
message delivery failed that eligible resend

### Request

```http
PUT /v1/messages/{transactional_message_id}
```

#### Headers

| Field           | Required | Description          |
| --------------- | -------- | -------------------- |
| `Content-Type`  | Yes      | `application/json`   |
| `Authorization` | Yes      | Use `Bearer <TOKEN>` |

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
    ]
  }
}
```

| Field                             | Required    | Description                                                                                                   |
| --------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------- |
| `type`                            | Yes         | `EMAIL` for email message                                                                                     |
| `message.sender`                  | Yes         | Sender domain registered.                                                                                     |
| `message.recipient`               | Yes         | Single recipient email address.                                                                               |
| `message.subject`                 | Yes         | Must satisfy these conditions: It should not be empty, contain emojis, or start with 'Re:' or 'Fwd:'.         |
| `message.template_id`             | Conditional | Provide when not supplying `message.html`; references stored template.                                        |
| `message.html`                    | Conditional | Base64-encoded HTML content when no template is referenced.                                                   |
| `message.template_values`         | No          | Key/value map injected into template rendering.                                                               |
| `message.attachments[].type`      | No          | Determines the `content` type: `ASSET` references asset ID from the asset manager, `RAW` uses inlined base64. |
| `message.attachments[].content`   | No          | Payload or asset identifier depending on `type`.                                                              |
| `message.attachments[].file_name` | No          | Friendly download filename surfaced to recipients.                                                            |

---

### Response

**200 – OK**

Message resend.

```json
{
  "status": "<status>",
  "updated_at": "<timestamp>",
  "current_attemp": <number>
}
```

| Field        | Description                                           |
| ------------ | ----------------------------------------------------- |
| `status`     | Message status                                        |
| `updated_at` | Timestamp of latest updated.                          |
| `current_attemp`| Number of retry count                              |

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

When the specified `transactional_message_id` does not exist in the sending history, throws `MessageNotFoundError`.

```json
{
  "type": "MessageNotFoundError",
  "message": "The message specified could not be found."
}
```

**406 – Not Acceptable**

When the retry data provided does not match the original message data, throws `RetryDataMismatchError`.

```json
{
  "type": "RetryContentMismatchError",
  "message": "The content of the retried message does not match the original message."
}
```

**406 – Not Acceptable**

When the message status does not permit retrying, throws `MessageStatusNotAllowedToRetryError`.

```json
{
  "type": "RetryDeniedError",
  "message": "The specified message is not eligible for retry."
}
```

**406 – Not Acceptable**

When the maximum number of retry attempts for the message has been exceeded, throws `MaxRetryAttemptsExceededError`.

```json
{
  "type": "MaxRetryAttemptsExceededError",
  "message": "The maximum number of retry attempts on this message has been exceeded."
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

### Example: resend inline HTML with `curl`

Encode your HTML in base64 (no newlines) and place it in `message.html`.

```bash
curl -X PUT https://api.nipamail.com/v1/messages/{transactional_message_id} \
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

---