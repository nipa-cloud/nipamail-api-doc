# การส่งข้อความธุรกรรม

## ส่งข้อความธุรกรรม

### คำขอ
```http
POST /v1/messages
```

#### เฮดเดอร์
| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `Content-Type` | Yes | `application/json` |
| `Authorization` | Yes | ใช้ `Bearer <TOKEN>` |

#### บอดี้คำขอ
```json
{
  "type": "EMAIL" | "SMS",
  "message": {
    "sender": "<sender name>",
    "recipient": "<target email>",
    "subject": "<string>",
    "template_id": "<template identifier>",     // ต้องระบุเมื่อไม่มี html
    "html": "<base64 encoded html>",            // ต้องระบุเมื่อไม่มี template_id
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

| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `type` | Yes | ใช้ `EMAIL` สำหรับอีเมล |
| `message.sender` | Yes | โดเมนผู้ส่งที่ลงทะเบียนแล้ว |
| `message.recipient` | Yes | อีเมลผู้รับ |
| `message.subject` | Yes | Subject ของอีเมล ต้องไม่เป็นค่าว่าง ไม่ใส่อีโมจิ และไม่ขึ้นต้นด้วย `Re:` หรือ `Fwd:` |
| `message.template_id` | Conditional | ระบุเมื่อไม่ส่ง `message.html`; อ้างถึงเทมเพลตก่อนหน้า |
| `message.html` | Conditional | HTML ที่เข้ารหัส base64 เมื่อไม่ใช้เทมเพลต |
| `message.template_values` | No | key/value ที่ใช้เรนเดอร์ตัวแปรในเทมเพลต |
| `message.attachments[].type` | No | กำหนดชนิด `content`: `ASSET` อ้างอิง Asset ID, `RAW` ใช้ base64 แปะมา |
| `message.attachments[].content` | No | เพย์โหลดหรือรหัสอ้างอิง ขึ้นกับ `type` |
| `message.attachments[].file_name` | No | ชื่อไฟล์ที่ผู้รับจะเห็นเวลาดาวน์โหลด |

---

### การตอบกลับ

**201 – Created**

สร้างข้อความสำเร็จ

```json
{
  "id": "<message id>",
  "status": "<status>",
  "updated_at": "<timestamp>",
  "cost": <number>
}
```
| ฟิลด์ | คำอธิบาย |
| --- | --- |
| `id` | รหัสข้อความธุรกรรม |
| `status` | สถานะข้อความ |
| `updated_at` | เวลาที่อัปเดตล่าสุด |
| `cost` | เครดิตที่ใช้ (เก็บเป็นจำนวนเต็ม คูณ 100) |

**400 – Bad Request**

บอดี้คำขอไม่ผ่านการตรวจสอบความถูกต้อง

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

ข้อมูลยืนยันตัวตนไม่ถูกต้อง หรือถูกจำกัดด้วย CIDR

```json
{
  "type": "UnauthorizedError",
  "message": "Invalid credentials or restricted by CIDR."
}
```

**404 – Not Found**

กรณี `template_id` ที่ระบุไม่พบ จะคืน `TemplateNotFoundError`

```json
{
  "type": "TemplateNotFoundError",
  "message": "The template specified could not be found."
}
```

**404 – Not Found**

กรณีไฟล์แนบแบบ `ASSET` ไม่พบใน asset manager จะคืน `AssetNotFoundError`

```json
{
  "type": "AssetNotFoundError",
  "message": "One or more of the specified assets could not be found."
}
```

**406 – Not Acceptable**

หากโดเมนผู้ส่งไม่ใช่โดเมนที่ลงทะเบียน จะคืน `InvalidSenderError`

```json
{
  "type": "InvalidSenderError",
  "message": "The specified sender domain is not registered."
}
```

**406 – Not Acceptable**

หากเครดิตคงเหลือไม่พอ จะคืน `InsufficientCreditError`

```json
{
  "type": "InsufficientCreditError",
  "message": "Your credit balance is insufficient."
}
```


**406 – Not Acceptable**

หากขนาดไฟล์แนบรวมเกินเพดาน จะคืน `TotalAttachmentSizeExceededError`

```json
{
  "type": "TotalAttachmentSizeExceededError",
  "message": "The total size of attachments exceeds the allowed limit."
}
```

**500 – Internal Server Error**

ข้อผิดพลาดภายในระบบ จะคืน `InternalServerError`

```json
{
  "type": "InternalServerError",
  "message": "Something went wrong, please try again later."
}
```

### ตัวอย่าง: ส่ง HTML แบบ inline ด้วย `curl`

เข้ารหัส HTML เป็น base64 (ไม่มีบรรทัดใหม่) แล้วใส่ใน `message.html`

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

### ตัวอย่าง: ใช้เทมเพลตและตัวแปรด้วย `curl`

ใช้ `template_id` ที่บันทึกไว้ พร้อมส่งค่าเรนเดอร์ใน `template_values`

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

### ตัวอย่าง: ใช้เทมเพลต ใส่ตัวแปร และแนบไฟล์แบบ ASSET ด้วย `curl`

อ้างอิงไฟล์แนบที่อัปโหลดแล้วด้วย Asset ID

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

### ตัวอย่าง: ใช้เทมเพลต ใส่ตัวแปร และแนบไฟล์ base64 แบบ RAW ด้วย `curl`

เข้ารหัสไฟล์เป็น base64 แล้วตั้ง `attachments[].type` เป็น `RAW`

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
## สอบถามสถานะข้อความธุรกรรม
### คำขอ
```http
GET /v1/messages/{transactional_message_id}
```
#### พารามิเตอร์ในพาธ
| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `transactional_message_id` | Yes | รหัสข้อความธุรกรรมที่ต้องการ |
#### เฮดเดอร์
| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `Authorization` | Yes | ใช้ `Bearer <TOKEN>` |

### การตอบกลับ

**200 – OK**

พบข้อมูลข้อความที่ต้องการ

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
| ฟิลด์ | ชนิด | คำอธิบาย |
| --- | --- | --- |
| `id` | string | รหัสล็อกข้อความที่ส่งกลับจาก `MessageLogDetailResponse` |
| `status` | MessageLogStatus | สถานะล่าสุดของการส่ง |
| `sender` | string | อีเมล/ชื่อผู้ส่งที่ใช้ |
| `recipient` | string | อีเมลผู้รับ |
| `opened` | number | จำนวนครั้งที่เปิดอีเมล (ที่ติดตามได้) |
| `clicked` | number | จำนวนคลิก (ที่ติดตามได้) |
| `cost` | number | เครดิตที่ใช้ คิดเป็นจำนวนเต็ม (คูณ 100) |
| `created_at` | Date | เวลาเริ่มสร้างล็อก |
| `updated_at` | Date | เวลาอัปเดตล็อกล่าสุด |
| `status_logs` | StatusLogResponse[] | ลำดับเหตุการณ์สถานะการส่ง |
| `status_logs[].status` | MessageLogStatus | สถานะในแต่ละเหตุการณ์ |
| `status_logs[].response.code` | number | โค้ดตอบกลับจากผู้ให้บริการ/MTA |
| `status_logs[].response.content` | string | ข้อความตอบกลับดิบจากผู้ให้บริการ |
| `status_logs[].timestamp` | number | เวลาที่ผู้ให้บริการกระทำเหตุการณ์ (epoch ms) |
| `status_logs[].created` | number | เวลาที่บันทึกสถานะ (epoch ms) |
| `status_logs[].error_message` | string | ข้อความผิดพลาด ถ้ามี |

#### ความหมายของ `MessageLogStatus`

| สถานะ | ความหมาย |
| --- | --- |
| `Created` | บันทึกคำขอแล้ว แต่ยังไม่ส่งไปผู้ให้บริการ |
| `Submitting` | กำลังส่งต่อไปยังผู้ให้บริการ |
| `Accepted` | ผู้ให้บริการรับคำขอแล้ว |
| `Delivered` | ผู้ให้บริการยืนยันว่าปลายทางรับข้อความแล้ว |
| `Deferred` | ผู้ให้บริการเลื่อนการส่ง จะลองใหม่ภายหลัง |
| `Delayed` | ระบบหรือผู้ให้บริการหน่วงการส่ง |
| `TimedOut` | ส่งไม่สำเร็จภายในเวลาที่กำหนด |
| `Feedback` | ผู้รับแจ้งปัญหา/ร้องเรียน (feedback loop) |
| `TechnicalError` | ข้อผิดพลาดภายในระบบ |
| `InsufficientCredit` | ส่งไม่สำเร็จเพราะเครดิตไม่พอ |

**Bounce statuses**
ข้อความอาจประสบปัญหาที่จัดอยู่ในกลุ่ม bounce ดังนี้

| สถานะ | ความหมาย | ประเภทบาวน์ซ์ | ลองส่งซ้ำได้ | ตัดเครดิต |
| --- | --- | --- | --- | --- |
| `HardBounce` | ล้มเหลวถาวร ไม่ควรลองส่งซ้ำ | Hard | No | Yes |
| `SoftBounce` | ล้มเหลวชั่วคราว สามารถลองใหม่ | Soft | Yes | No |
| `InvalidRecipient` | อีเมลผู้รับไม่ถูกต้องหรือไม่มีอยู่จริง | Hard | No | Yes |
| `BadDomain` | โดเมนผู้รับตั้งค่าไม่ถูกต้องหรือไม่มี MX | Hard | No | Yes |
| `InactiveMailbox` | กล่องจดหมายถูกปิดใช้งาน ระงับ หรือปิดบัญชี | Hard | No | Yes |
| `InvalidSender` | ผู้ส่ง/โดเมนไม่อนุญาตหรือไม่ผ่านการยืนยัน | Hard | No | Yes |
| `QuotaIssues` | กล่องจดหมายผู้รับเต็มหรือเกินขนาดไฟล์แนบ | Soft | Yes | No |
| `NoAnswerFromHost` | เซิร์ฟเวอร์ปลายทางไม่ตอบสนองระหว่างส่ง | Hard | No | Yes |
| `BadConnection` | การเชื่อมต่อกับโฮสต์ปลายทางล้มเหลวหรือหลุด | Soft | Yes | No |
| `DNSFailure` | ค้นหา DNS ของโดเมนผู้รับล้มเหลว (เช่น ไม่มี MX) | Hard | No | Yes |
| `RoutingErrors` | ผู้ให้บริการปฏิเสธรีเลย์หรือหาทางส่งไม่ได้ | Soft | Yes | No |
| `TransientFailure` | ความล้มเหลวชั่วคราวที่ไม่ระบุ สามารถลองใหม่ | Soft | Yes | No |
| `MessageExpired` | หมดเวลาส่งก่อนที่ผู้ให้บริการจะส่งสำเร็จ | Hard | No | Yes |
| `ProtocolErrors` | ลำดับ/ไวยากรณ์คำสั่ง SMTP ถูกปฏิเสธ | Soft | Yes | No |
| `AuthenticationFailed` | ตรวจสอบตัวตน/DMARC/SPF ไม่ผ่าน | Soft | Yes | No |
| `PolicyRelated` | ถูกปฏิเสธด้วยเหตุผลเชิงนโยบายหรือ AUP | Soft | No | Yes |
| `SpamContent` | เนื้อหาข้อความถูกจัดเป็นสแปมหรือไวรัส | Soft | No | Yes |
| `SpamFiltered` | ผู้ให้บริการรับไว้แต่กรอง/กักกันจาก heuristics สแปม | Soft | No | Yes |
| `SpamBlock` | ปิดกั้นเพราะชื่อเสียงของผู้ส่ง/IP/โดเมน | Soft | No | Yes |
| `ProviderRejected` | ผู้ให้บริการปฏิเสธนอกรูปแบบบาวน์ซ์ | Soft | No | Yes |

**401 – Unauthorized**

ข้อมูลยืนยันตัวตนไม่ถูกต้อง หรือถูกจำกัดด้วย CIDR

```json
{
  "type": "UnauthorizedError",
  "message": "Invalid credentials or restricted by CIDR."
}
```

**404 – Not Found**

หากไม่พบข้อความที่ระบุ จะคืน `MessageNotFoundError`

```json
{
  "type": "MessageNotFoundError",
  "message": "The message specified could not be found."
}
```
