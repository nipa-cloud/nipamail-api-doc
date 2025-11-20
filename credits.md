# Credit enquiry

## Get the credit balance of the tenant
### Request
```http
GET /v1/tenants/me/credits
```
#### Headers
| Field | Required | Description |
| --- | --- | --- |
| `Authorization` | Yes | Use `Bearer <TOKEN>` |

### Response

**200 – OK**

Returns the current available and reserved credit for the tenant.

```json
{
  "credits": {
    "available": <number>,
    "reserved": <number>,
  }
}

```
| Field | Type | Description |
| --- | --- | --- |
| `credits` | object | Credit summary for the tenant |
| `credits.available` | number | Credits that can be spent immediately |
| `credits.reserved` | number | Credits held for pending transactions |


**401 – Unauthorized**

The specified credentials is invalid, or restricted by CIDR.

```json
{
  "type": "UnauthorizedError",
  "message": "Invalid credentials or restricted by CIDR."
}
```
