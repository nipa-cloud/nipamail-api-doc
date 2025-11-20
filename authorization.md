# Authentication

## Authenticate the application credential with the application ID and secret

### Request
```
POST /v1/auth/tokens
```
#### Headers
| Field | Required | Description |
| --- | --- | --- |
| `Content-Type` | Yes | `application/x-www-form-urlencoded` |

#### Parameter
| Field | Required | Description |
| --- | --- | --- |
| `grant_type` | Yes | `client_credentials` |
| `client_id` | Yes | Application ID |
| `client_secret` | Yes | Application secret |

### Response

**200 – OK**

Application authorized.

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
