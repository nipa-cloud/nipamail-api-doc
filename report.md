# Transactional report

## List transactional message reports

### Request

```http
GET /v1/report
```

#### Headers

| Field | Required | Description |
| --- | --- | --- |
| `Authorization` | Yes | Use `Bearer <TOKEN>` |

#### Query parameters

| Field | Required | Description |
| --- | --- | --- |
| `limit` | No | Maximum number of records to return. Defaults to `10`; maximum is `100`. |
| `start_date` | No | Start of the created-at filter as an ISO 8601 timestamp. Defaults to 30 days before the request time. |
| `end_date` | No | End of the created-at filter as an ISO 8601 timestamp. Defaults to the request time. |
| `cursor_id` | No | Message ID cursor. When supplied, the next page starts after this ID. |

### Response

**200 - OK**

Returns transactional messages.

```json
[
    {
        "id": "<message id>",
        "cost": "1.00",
        "opened": <number>,
        "clicked": <number>,
        "result": "<result>",
        "status": "<status>",
        "sender": "<sender name> <<sender@example.com>>",
        "recipient": "<recipient@example.com>",
        "created_at": "<timestamp>",
        "updated_at": "<timestamp>",
        "status_logs": [
            {
                "response": {
                    "code": <number>,
                    "content": "<provider response>"
                },
                "timestamp": <epoch-ms>,
                "created": <epoch-ms>
            }
        ]
    }
]
```

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Message log identifier. |
| `cost` | string | Credit cost formatted with two decimal places. |
| `opened` | number | Total tracked open events. |
| `clicked` | number | Total tracked click events. |
| `result` | EmailMessageResult | High-level result: `Processing`, `Ok`, `HardBounce`, `SoftBounce`, or `Error`. |
| `status` | MessageLogStatus | Latest delivery status recorded for the message. |
| `sender` | string | Sender email/name originally used. |
| `recipient` | string | Recipient email address. |
| `created_at` | Date | When the message log was created. |
| `updated_at` | Date | When the message log last changed. |
| `status_logs` | array | Latest status log details for the message. |
| `status_logs[].response.code` | number | Transport/MTA response code. |
| `status_logs[].response.content` | string | Raw provider response body. |
| `status_logs[].timestamp` | number | Event timestamp from provider (epoch ms). |
| `status_logs[].created` | number | Timestamp when the status log was persisted (epoch ms). |

**401 - Unauthorized**

The specified credentials is invalid, expired, or restricted by CIDR.

```json
{
    "type": "UnauthorizedError",
    "message": "Invalid credentials or restricted by CIDR."
}
```

**429 - TooManyRequests**

When the client exceeds the allowed number of report requests within the current rate-limit window, throws `TooManyRequestsError`.

```json
{
    "type": "TooManyRequestsError",
    "message": "Too Many Requests"
}
```

**500 - Internal Server Error**

When catastrophic errors occurred, throws `InternalServerError`.

```json
{
    "type": "InternalServerError",
    "message": "Something went wrong, Somewhere inside."
}
```

### Example: list transactional reports with `curl`

```bash
curl --get https://api.nipamail.com/v1/report \
    -H "Authorization: Bearer <TOKEN>" \
    --data-urlencode "limit=100" \
    --data-urlencode "start_date=2026-01-01T00:00:00.000Z" \
    --data-urlencode "end_date=2026-01-31T23:59:59.999Z"
```

### Example: continue from a cursor

Use the last `id` from the previous response as `cursor_id`.

```bash
curl --get https://api.nipamail.com/v1/report \
    -H "Authorization: Bearer <TOKEN>" \
    --data-urlencode "limit=100" \
    --data-urlencode "cursor_id=<message id>"
```
