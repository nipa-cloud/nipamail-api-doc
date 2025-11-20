# การยืนยันตัวตน

## ขอรับโทเค็นด้วย Application ID และ Secret

### คำขอ
```
POST /v1/auth/tokens
```
#### เฮดเดอร์
| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `Content-Type` | Yes | `application/x-www-form-urlencoded` |

#### พารามิเตอร์
| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `grant_type` | Yes | `client_credentials` |
| `client_id` | Yes | Application ID |
| `client_secret` | Yes | Application secret |

### การตอบกลับ

**200 – OK**

ยืนยันสิทธิ์แอปพลิเคชันสำเร็จ

```json
{
  "token_type": "Bearer",
  "access_token": "<token>",
  "expires_in": "<expiration in seconds>"
}
```

**400 – Bad Request**

หากบอดี้คำขอไม่ถูกต้อง จะตอบกลับด้วย `ValidationError`
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

ข้อมูลยืนยันตัวตนไม่ถูกต้อง หรือถูกจำกัดด้วย CIDR

```json
{
  "type": "UnauthorizedError",
  "message": "Invalid credentials or restricted by CIDR."
}
```
