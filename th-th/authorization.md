# การยืนยันตัวตน

## ยืนยันข้อมูล client ด้วย Client ID และ Secret

### คำขอ
```
POST /v1/auth/tokens
```
#### เฮดเดอร์
| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `Content-Type` | ใช่ | `application/json` |

#### บอดี้คำขอ

```json
{
    "grant_type": "client_credentials",
    "client_id": "<client id>",
    "client_secret": "<client secret>"
}
```

| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `grant_type` | ใช่ | `client_credentials` |
| `client_id` | ใช่ | Client ID |
| `client_secret` | ใช่ | Client secret |

### ตัวอย่าง: สร้าง access token ด้วย `curl`

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

### การตอบกลับ

**200 – OK**

ยืนยันสิทธิ์ client สำเร็จ

#### เฮดเดอร์

| ฟิลด์ | คำอธิบาย |
| --- | --- |
| `x-ratelimit-remaining` | จำนวนคำขอที่ยังส่งได้ก่อนถึงขีดจำกัด |
| `x-ratelimit-reset` | เวลา epoch หรือจำนวนวินาทีจนกว่าช่วง rate limit ปัจจุบันจะรีเซ็ต |
| `x-ratelimit-limit` | จำนวนคำขอทั้งหมดที่อนุญาตในช่วง rate limit ปัจจุบัน |
| `retry-after` | เวลาที่ไคลเอนต์ควรรอเป็นวินาทีก่อนลองส่งคำขอใหม่ |

`access_token` ใช้งานได้สูงสุด 1 ชั่วโมงหลังออกโทเค็น

```json
{
    "token_type": "Bearer",
    "access_token": "<token>",
    "expires_in": "<expiration in seconds>"
}
```

**400 – Bad Request**

เมื่อส่งบอดี้คำขอผิดรูปแบบ ระบบจะตอบกลับ `ValidationError`

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

โทเค็นหมดอายุ หรือถูกจำกัด IP ที่สามารถขอส่งข้อความได้

```json
{
    "type": "UnauthorizedError",
    "message": "Invalid credentials or restricted by CIDR."
}
```

**429 – TooManyRequests**

เมื่อไคลเอนต์ส่งคำขอเกินจำนวนที่อนุญาตภายในช่วงเวลาที่กำหนด ระบบจำกัดอัตราคำขอจะตอบกลับข้อผิดพลาดนี้ ให้ลองใหม่หลังเวลาที่ระบุในข้อความ

```json
{
    "type": "TooManyRequestsError",
    "message": "Rate limit exceeded, retry in 5 seconds."
}
```
