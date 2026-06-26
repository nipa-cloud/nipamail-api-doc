# รายงานข้อความธุรกรรม

## แสดงรายการรายงานข้อความธุรกรรม

### คำขอ

```http
GET /v1/report
```

#### เฮดเดอร์

| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `Authorization` | ใช่ | ใช้ `Bearer <TOKEN>` |

#### พารามิเตอร์ Query

| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `limit` | ไม่ใช่ | จำนวนรายการสูงสุดที่ต้องการให้ส่งกลับ ค่าเริ่มต้นคือ `10` และค่าสูงสุดคือ `100` |
| `start_date` | ไม่ใช่ | เวลาเริ่มต้นของตัวกรอง `created_at` ในรูปแบบ ISO 8601 ค่าเริ่มต้นคือ 30 วันก่อนเวลาที่ส่งคำขอ |
| `end_date` | ไม่ใช่ | เวลาสิ้นสุดของตัวกรอง `created_at` ในรูปแบบ ISO 8601 ค่าเริ่มต้นคือเวลาที่ส่งคำขอ |
| `cursor_id` | ไม่ใช่ | cursor ของรหัสข้อความ เมื่อระบุแล้ว หน้าถัดไปจะเริ่มหลัง ID นี้ |

### การตอบกลับ

**200 - OK**

ตอบกลับเป็นข้อมูลของรายงาน

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

| ฟิลด์ | ชนิด | คำอธิบาย |
| --- | --- | --- |
| `id` | string | รหัสข้อความธุรกรรม |
| `cost` | string | เครดิตที่ใช้ แสดงเป็นทศนิยมสองตำแหน่ง |
| `opened` | number | จำนวนการเปิดอีเมลที่ติดตามได้ทั้งหมด |
| `clicked` | number | จำนวนการคลิกลิงก์ที่ติดตามได้ทั้งหมด |
| `result` | EmailMessageResult | ผลลัพธ์ระดับสูง: `Processing`, `Ok`, `HardBounce`, `SoftBounce` หรือ `Error` |
| `status` | MessageLogStatus | สถานะการส่งล่าสุดที่บันทึกไว้ของข้อความ |
| `sender` | string | อีเมล/ชื่อผู้ส่งที่ใช้เดิม |
| `recipient` | string | อีเมลผู้รับ |
| `created_at` | Date | เวลาที่สร้างล็อกข้อความ |
| `updated_at` | Date | เวลาที่ล็อกข้อความเปลี่ยนแปลงล่าสุด |
| `status_logs` | array | รายละเอียดล็อกสถานะล่าสุดของข้อความ |
| `status_logs[].response.code` | number | โค้ดตอบกลับจาก transport/MTA |
| `status_logs[].response.content` | string | ข้อความตอบกลับจากผู้ให้บริการปลายทาง |
| `status_logs[].timestamp` | number | เวลาจากผู้ให้บริการปลายทาง (epoch ms) |
| `status_logs[].created` | number | เวลาที่บันทึกสถานะ (epoch ms) |

**401 - Unauthorized**

โทเค็นหมดอายุ หรือถูกจำกัดด้วย CIDR

```json
{
    "type": "UnauthorizedError",
    "message": "Invalid credentials or restricted by CIDR."
}
```

**429 - TooManyRequests**

เมื่อไคลเอนต์ส่งคำขอรายงานเกินจำนวนที่อนุญาตในช่วง rate limit ปัจจุบัน ระบบจะตอบกลับ `TooManyRequestsError`

```json
{
    "type": "TooManyRequestsError",
    "message": "Too Many Requests"
}
```

**500 - Internal Server Error**

เมื่อเกิดข้อผิดพลาดร้ายแรงภายในระบบ จะตอบกลับ `InternalServerError`

```json
{
    "type": "InternalServerError",
    "message": "Something went wrong, Somewhere inside."
}
```

### ตัวอย่าง: แสดงรายการรายงานข้อความธุรกรรมด้วย `curl`

```bash
curl --get https://api.nipamail.com/v1/report \
    -H "Authorization: Bearer <TOKEN>" \
    --data-urlencode "limit=100" \
    --data-urlencode "start_date=2026-01-01T00:00:00.000Z" \
    --data-urlencode "end_date=2026-01-31T23:59:59.999Z"
```

### ตัวอย่าง: อ่านหน้าถัดไปด้วย cursor

ใช้ `id` รายการสุดท้ายจากการตอบกลับก่อนหน้าเป็น `cursor_id`

```bash
curl --get https://api.nipamail.com/v1/report \
    -H "Authorization: Bearer <TOKEN>" \
    --data-urlencode "limit=100" \
    --data-urlencode "cursor_id=<message id>"
```
