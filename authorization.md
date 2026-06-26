# Authentication

## Authenticate the client credential with the client ID and secret

### Request
```
POST /v1/auth/tokens
```
#### Headers
| Field | Required | Description |
| --- | --- | --- |
| `Content-Type` | Yes | `application/json` |

#### Request body

```json
{
    "grant_type": "client_credentials",
    "client_id": "<client id>",
    "client_secret": "<client secret>"
}
```

| Field | Required | Description |
| --- | --- | --- |
| `grant_type` | Yes | `client_credentials` |
| `client_id` | Yes | Client ID |
| `client_secret` | Yes | Client secret |

### Example: generate an access token with `curl`

```bash
curl -X POST https://api.nipamail.com/v1/auth/tokens \
    -H "Content-Type: application/json" \
    --data @- <<'JSON'
{
    "grant_type": "client_credentials",
    "client_id": "<CLIENT_ID>",
    "client_secret": "<CLIENT_SECRET>"
}
JSON
```

### Response

**200 – OK**

Client authorized.

#### Headers

| Field | Description |
| --- | --- |
| `x-ratelimit-remaining` | How many requests you can still make before hitting the limit. |
| `x-ratelimit-reset` | Epoch time (or seconds until) when the current window resets. |
| `x-ratelimit-limit` | Total number of requests allowed in the current window. |
| `retry-after` | The time (in seconds) that the client should wait before retrying the request. |

The `access_token` remains usable for up to 1 hour after issuance.

```json
{
    "token_type": "Bearer",
    "access_token": "<token>",
    "expires_in": "<expiration in seconds>"
}
```

**400 – Bad Request**

When a malformed request body supplied, throws `ValidationError`

```json
{
    "type": "ValidationError",
    "errors": [
        {
            "target": {
                "grant_type": "client_credential"
            },
            "property": "grant_type",
            "children": [],
            "constraints": {
                "isOneOf": "grant_type must be one of 'client_credentials'"
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

**429 – TooManyRequests**

When the client exceeds the allowed number of requests within a given timeframe, the rate limiter triggers and returns this error. Retry after the specified duration in the message.

```json
{
    "type": "TooManyRequestsError",
    "message": "Rate limit exceeded, retry in 5 seconds."
}
```
