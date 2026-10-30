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
| `Authorization` | Yes      | `Bearer <TOKEN>`     |

#### Request body

```json
{
    "type": "EMAIL" | "SMS",
    "message": {
        "sender": "<sender name>",
        "recipient": "<target email>",
        "cc": ["copy@example.com"],
        "bcc": ["archive@example.com"],
        "subject": "<string>",
        "template_id": "<template identifier>",     // required when html is absent
        "html": "<base64 encoded html>",            // required when template_id is absent
        "selector": "<dkim selector>",
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
| `message.cc`                      | No          | Array of CC email addresses, up to 4 entries. Each entry must be a valid email address. |
| `message.bcc`                     | No          | Array of BCC email addresses, up to 4 entries. Each entry must be a valid email address. |
| `message.subject`                 | Yes         | Must satisfy these conditions: It should not be empty, contain emojis, or start with 'Re:' or 'Fwd:'.         |
| `message.template_id`             | Conditional | Provide when not supplying `message.html`; references stored template.                                        |
| `message.html`                    | Conditional | Base64-encoded HTML content when no template is referenced.                                                   |
| `message.selector`                | No          | DKIM selector to use when the sender domain has more than one active DKIM record.                             |
| `message.template_values`         | No          | Key/value map injected into template rendering.                                                               |
| `message.attachments[].type`      | No          | Determines the `content` type: `ASSET` references asset ID from the asset manager, `RAW` uses inlined base64. |
| `message.attachments[].content`   | No          | Payload or asset identifier depending on `type`.                                                              |
| `message.attachments[].file_name` | No          | Friendly download filename surfaced to recipients.                                                            |

#### CC and BCC rules

- Keep the primary recipient in `message.recipient`; CC and BCC do not replace it.
- Supply `message.cc` and `message.bcc` as arrays of email strings. Omit either field or use `[]` when there are no recipients of that type.
- Each array accepts at most 4 addresses, allowing up to 9 recipients in total including the primary recipient.
- The primary recipient must not appear in CC or BCC. Addresses must also be unique within each array and across both arrays.
- Invalid email addresses, non-array values, or more than 4 entries in either array return HTTP 400 `ValidationError`.

#### Example request body with CC and BCC

Send this JSON body to `POST /v1/messages`. Replace the sender with an address on your registered sender domain. The `html` value is the Base64 encoding of `<p>Thank you.</p>`.

```json
{
    "type": "EMAIL",
    "message": {
        "sender": "NipaMail <sender@yourdomain.com>",
        "recipient": "recipient@example.com",
        "cc": ["copy@example.com", "team@example.com"],
        "bcc": ["archive@example.com"],
        "subject": "Order confirmation",
        "html": "PHA+VGhhbmsgeW91LjwvcD4="
    }
}
```

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
    "status": "Queued",
    "updated_at": "<timestamp>",
    "cost": 1,
    "recipient": "recipient@example.com",
    "cc": [
        {
            "id": "<cc message id>",
            "status": "Queued",
            "updated_at": "<timestamp>",
            "cost": 1,
            "recipient": "copy@example.com"
        }
    ],
    "bcc": [
        {
            "id": "<bcc message id>",
            "status": "Queued",
            "updated_at": "<timestamp>",
            "cost": 1,
            "recipient": "archive@example.com"
        }
    ]
}
```

| Field        | Description                                           |
| ------------ | ----------------------------------------------------- |
| `id`         | Transactional message ID                              |
| `status`     | Message status                                        |
| `updated_at` | Timestamp of latest updated.                          |
| `cost`       | Credit cost for the primary recipient, returned in credit units (internal value divided by 100). The example value is illustrative. |
| `recipient`  | Primary recipient email address. |
| `cc`, `bcc`  | Arrays of results for CC and BCC recipients. Each field is omitted when its request array is empty or absent. |
| `cc[].id`, `bcc[].id` | Message ID for that individual recipient. |
| `cc[].recipient`, `bcc[].recipient` | Email address for that individual recipient. |
| `cc[].status`, `bcc[].status` | Initial message status (`Queued`). |
| `cc[].updated_at`, `bcc[].updated_at` | Timestamp of the latest update for that recipient. |
| `cc[].cost`, `bcc[].cost` | Credit cost for that individual recipient, in the same units as the top-level `cost`. |

The top-level result describes the primary recipient. Each CC and BCC recipient has its own message ID and cost; the top-level `cost` is not the sum for all recipients.

Use the relevant `id`, `cc[].id`, or `bcc[].id` with `GET /v1/messages/{transactional_message_id}` to query that recipient's message status.

#### Possible exceptions

| Status | Type                                | When it happens |
| ------ | ----------------------------------- | --------------- |
| 400    | `ValidationError`                   | Request body is malformed or fails validation. |
| 401    | `UnauthorizedError`                 | Credentials are invalid, expired, or restricted by CIDR. |
| 404    | `TemplateNotFoundError`             | The specified `template_id` does not exist. |
| 404    | `AssetNotFoundError`                | One or more `ASSET` attachments cannot be found. |
| 406    | `InvalidSenderError`                | Sender domain is not registered or accepted. |
| 406    | `InvalidRecipientError`             | Recipient email address or domain cannot be accepted. |
| 406    | `RecipientConflictError`            | The primary recipient also appears in CC or BCC. |
| 406    | `DuplicateEmailError`               | An address is repeated within CC or BCC, or appears in both arrays. |
| 406    | `InsufficientCreditError`           | Account credit balance is insufficient. |
| 406    | `TotalAttachmentSizeExceededError`  | Total attachment size exceeds the allowed limit. |
| 406    | `TotalAttachmentCountExceededError` | Attachment count exceeds the allowed limit. |
| 429    | `TooManyRequestsError`              | Request volume exceeds the current rate-limit window. |
| 500    | `InternalServerError`               | An unexpected internal error occurs. |

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

When the recipient email address or recipient domain cannot be accepted, throws `InvalidRecipientError`.

```json
{
    "type": "InvalidRecipientError",
    "message": "The specified recipient is invalid."
}
```

**406 – Not Acceptable**

When `message.recipient` also appears in `message.cc` or `message.bcc`, throws `RecipientConflictError`.

```json
{
    "type": "RecipientConflictError",
    "message": "The recipient cannot be the same as the cc or bcc."
}
```

**406 – Not Acceptable**

When an email address is repeated within either array or across CC and BCC, throws `DuplicateEmailError`.

```json
{
    "type": "DuplicateEmailError",
    "message": "Duplicate email addresses are not allowed in cc or bcc."
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

**406 – Not Acceptable**

When the number of attachments exceeds the tenant attachment count limit, throws `TotalAttachmentCountExceededError`.

```json
{
    "type": "TotalAttachmentCountExceededError",
    "message": "The total number of attachments exceeds the allowed limit."
}
```

**429 – TooManyRequests**

When the client exceeds the allowed number of send requests within the current rate-limit window, throws `TooManyRequestsError`.

```json
{
    "type": "TooManyRequestsError",
    "message": "Rate limit exceeded, retry in 5 seconds."
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

### Example: Send Base64 encoded HTML with `curl`

Encode your HTML in base64 (no newlines) and place it in `message.html`.

```bash
curl -X POST https://api.nipamail.com/v1/messages \
    -H "Authorization: Bearer <TOKEN>" \
    -H "Content-Type: application/json" \
    --data @- <<'JSON'
{
    "type": "EMAIL",
    "message": {
        "sender": "<sender display name> <<sender@yourdomain.com>>",
        "recipient": "<recipient@example.com>",
        "subject": "<subject line>",
        "html": "<base64-encoded-html>"
    }
}
JSON
```

### Example: send with CC and BCC using `curl`

Use distinct addresses for the primary recipient, CC, and BCC. The same `cc` and `bcc` fields can also be used with `template_id` and `template_values` instead of `html`.

```bash
curl -X POST https://api.nipamail.com/v1/messages \
    -H "Authorization: Bearer <TOKEN>" \
    -H "Content-Type: application/json" \
    --data @- <<'JSON'
{
    "type": "EMAIL",
    "message": {
        "sender": "NipaMail <sender@yourdomain.com>",
        "recipient": "recipient@example.com",
        "cc": ["copy@example.com", "team@example.com"],
        "bcc": ["archive@example.com"],
        "subject": "Order confirmation",
        "html": "PHA+VGhhbmsgeW91LjwvcD4="
    }
}
JSON
```

### Example: template with variables using `curl`

Provide a stored `template_id` and pass values to render inside the template.

```bash
curl -X POST https://api.nipamail.com/v1/messages \
    -H "Authorization: Bearer <TOKEN>" \
    -H "Content-Type: application/json" \
    --data @- <<'JSON'
{
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
}
JSON
```

### Example: template with variables and asset attachment using `curl`

Reference an uploaded asset by ID for attachments.

```bash
curl -X POST https://api.nipamail.com/v1/messages \
    -H "Authorization: Bearer <TOKEN>" \
    -H "Content-Type: application/json" \
    --data @- <<'JSON'
{
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
}
JSON
```

### Example: template with variables and raw base64 attachment using `curl`

Inline a file by base64-encoding it and setting `attachments[].type` to `RAW`.

```bash
curl -X POST https://api.nipamail.com/v1/messages \
    -H "Authorization: Bearer <TOKEN>" \
    -H "Content-Type: application/json" \
    --data @- <<'JSON'
{
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
}
JSON
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
    "result": "<result>",
    "status": "<status>",
    "sender": "<sender name> <<sender@example.com>>",
    "recipient": "<recipient@example.com>",
    "opened": <number>,
    "clicked": <number>,
    "cost": <number>,
    "created_at": "<timestamp>",
    "updated_at": "<timestamp>",
    "retry_attempts": <number>,
    "status_logs": [
        {
            "result": "<result>",
            "status": "<status>",
            "response": {
                "code": <number>,
                "content": "<provider response>"
            },
            "timestamp": <epoch-ms>,
            "created": <epoch-ms>
        },
        {
            "result": "<result>",
            "status": "<status>",
            "details": "<system detail>",
            "response": null,
            "timestamp": <epoch-ms>,
            "created": <epoch-ms>
        }
    ]
}

```

| Field                            | Type                | Description                                                               |
| -------------------------------- | ------------------- | ------------------------------------------------------------------------- |
| `id`                             | string              | Unique message id.     |
| `result`                         | EmailMessageResult  | High-level result: `Processing`, `Ok`, `HardBounce`, `SoftBounce`, or `Error`. |
| `status`                         | MessageLogStatus    | Latest delivery status recorded for the message.                          |
| `sender`                         | string              | Sender email/name originally used.                                        |
| `recipient`                      | string              | Recipient email address.                                                  |
| `opened`                         | number              | Total tracked open events.                                                |
| `clicked`                        | number              | Total tracked click events.                                               |
| `cost`                           | number              | Credit cost incurred for delivery.                                        |
| `created_at`                     | Date                | When the log entry was created.                                           |
| `updated_at`                     | Date                | When the log entry last changed.                                          |
| `retry_attempts`                 | number              | Number of retry attempts already made for the message.                    |
| `status_logs`                    | StatusLogResponse[] | Chronological list of delivery state changes.                             |
| `status_logs[].result`           | EmailMessageResult  | High-level result recorded for this status log.                           |
| `status_logs[].status`           | MessageLogStatus    | Status recorded for the log entry.                                        |
| `status_logs[].response`         | object \| null      | Transport/MTA response object, or `null` for system status logs.          |
| `status_logs[].response.code`    | number \| null      | Transport/MTA response code.                                              |
| `status_logs[].response.content` | string \| null      | Raw provider response body.                                               |
| `status_logs[].details`          | string              | Optional system detail for system status logs.                            |
| `status_logs[].timestamp`        | number              | Event timestamp from provider (epoch ms).                                 |
| `status_logs[].created`          | number              | Timestamp when the status log was persisted (epoch ms).                   |

#### `MessageLogStatus` meaning

| Result       | Status                 | Meaning                                                                                                      |
| ------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------ |
| `Processing` | `Created`              | Message request was saved but not yet accepted by the message transport agent.                               |
| `Processing` | `Accepted`             | Message transport agent accepted the message request.                                                        |
| `Processing` | `Deferred`             | Message transport agent deferred the message and will retry later.                                           |
| `Ok`         | `Delivered`            | Recipient provider confirmed that the recipient server received the message.                                 |
| `Error`      | `TimedOut`             | Internal system attempts timed out before completion.                                                        |
| `Error`      | `TechnicalError`       | Internal system error prevented sending.                                                                     |
| `Error`      | `Feedback`             | Recipient submitted feedback or a complaint through a feedback loop.                                         |
| `Error`      | `InsufficientCredit`   | Sending failed because the tenant lacks credits.                                                             |
| `Error`      | `Blocked`              | The recipient is blocked by NipaMail's blocklist.                                                            |
| `HardBounce` | `InvalidRecipient`     | Recipient address is invalid or the mailbox does not exist.                                                  |
| `HardBounce` | `InactiveMailbox`      | Mailbox exists but is disabled, suspended, or closed.                                                        |
| `HardBounce` | `BadDomain`            | Recipient domain is misconfigured or missing valid MX records.                                               |
| `HardBounce` | `InvalidSender`        | Sender/domain is not allowed or fails to validate DKIM, SPF or DMARC.                                        |
| `HardBounce` | `NoAnswerFromHost`     | Recipient provider host was found, but the host did not respond to the message request.                      |
| `HardBounce` | `DNSFailure`           | Recipient DNS lookup failed, for example no MX record was found.                                             |
| `HardBounce` | `MessageExpired`       | Delivery window elapsed before the message transport agent could deliver the message successfully.           |
| `HardBounce` | `ProviderRejected`     | Recipient provider rejected the message with an error that does not match another hard-bounce status.        |
| `SoftBounce` | `QuotaIssues`          | Recipient mailbox is full or the attachment is too large for the recipient to receive.                       |
| `SoftBounce` | `BadConnection`        | Connection to the recipient provider failed or was dropped.                                                  |
| `SoftBounce` | `RoutingErrors`        | Recipient provider refused to relay or could not route to the recipient.                                     |
| `SoftBounce` | `ProtocolErrors`       | SMTP command sequence was rejected or STARTTLS encryption could not be established with the recipient provider. |
| `SoftBounce` | `AuthenticationFailed` | Recipient provider failed DKIM, DMARC or SPF checks for the sender.                                         |
| `SoftBounce` | `PolicyRelated`        | Rejected by recipient provider policy or acceptable use restrictions.                                        |
| `SoftBounce` | `SpamContent`          | Message content was classified as spam or virus.                                                             |
| `SoftBounce` | `SpamBlock`            | Delivery blocked due to sender/IP/domain reputation or blocklists.                                           |
| `SoftBounce` | `GenericSoftBounce`    | Recipient provider rejected the message with an error that does not match another soft-bounce status.        |

#### Bounce retry and credit charged

"*" Credit charge occurs on the first encounter of the message recipient, next encounter will be blocked by our blocklist.

| Result       | Status                 | Retry | Credit charged |
| ------------ | ---------------------- | ----- | -------------- |
| `HardBounce` | `InvalidRecipient`     | No    | Yes *          |
| `HardBounce` | `InactiveMailbox`      | No    | Yes *          |
| `HardBounce` | `BadDomain`            | No    | No             |
| `HardBounce` | `InvalidSender`        | No    | Yes            |
| `HardBounce` | `NoAnswerFromHost`     | No    | No             |
| `HardBounce` | `DNSFailure`           | No    | No             |
| `HardBounce` | `MessageExpired`       | No    | No             |
| `HardBounce` | `ProviderRejected`     | No    | Yes            |
| `SoftBounce` | `QuotaIssues`          | Yes   | Yes            |
| `SoftBounce` | `BadConnection`        | Yes   | No             |
| `SoftBounce` | `RoutingErrors`        | Yes   | No             |
| `SoftBounce` | `ProtocolErrors`       | Yes   | No             |
| `SoftBounce` | `AuthenticationFailed` | Yes   | No             |
| `SoftBounce` | `PolicyRelated`        | No    | Yes            |
| `SoftBounce` | `SpamContent`          | No    | Yes            |
| `SoftBounce` | `SpamBlock`            | Yes   | Yes            |
| `SoftBounce` | `GenericSoftBounce`    | Yes    | No             |

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

For requests originally sent with CC or BCC, preserve the exact original request body, including those arrays. The API compares a hash of the raw body, so changes to whitespace, field order, or recipient lists cause `RetryContentMismatchError`. The current resend implementation queues only the primary recipient; it does not resend CC or BCC copies.

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
        "selector": "<dkim selector>",
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
| `message.selector`                | No          | DKIM selector to use when the sender domain has more than one active DKIM record.                             |
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
    "current_attempt": <number>
}
```

| Field        | Description                                           |
| ------------ | ----------------------------------------------------- |
| `status`     | Message status                                        |
| `updated_at` | Timestamp of latest updated.                          |
| `current_attempt` | Number of retry count                            |

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

### Example: resend Base64 encoded HTML with `curl`

Encode your HTML in base64 (no newlines) and place it in `message.html`.

```bash
curl -X PUT https://api.nipamail.com/v1/messages/{transactional_message_id} \
    -H "Authorization: Bearer <TOKEN>" \
    -H "Content-Type: application/json" \
    --data @- <<'JSON'
{
    "type": "EMAIL",
    "message": {
        "sender": "<sender display name> <<sender@yourdomain.com>>",
        "recipient": "<recipient@example.com>",
        "subject": "<subject line>",
        "html": "<base64-encoded-html>"
    }
}
JSON
```

---
