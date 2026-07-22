# การส่งข้อความธุรกรรม

## ส่งข้อความธุรกรรม

### คำขอ

```http
POST /v1/messages
```

#### เฮดเดอร์

| ฟิลด์           | บังคับ | คำอธิบาย             |
| --------------- | ------ | -------------------- |
| `Content-Type`  | ใช่   | `application/json`   |
| `Authorization` | ใช่   | ใช้ `Bearer <TOKEN>` |

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

| ฟิลด์                             | บังคับ      | คำอธิบาย                                                                             |
| --------------------------------- | ----------- | ------------------------------------------------------------------------------------ |
| `type`                            | ใช่         | ใช้ `EMAIL` สำหรับข้อความอีเมล                                                        |
| `message.sender`                  | ใช่         | โดเมนผู้ส่งที่ลงทะเบียนแล้ว                                                          |
| `message.recipient`               | ใช่         | อีเมลผู้รับหนึ่งราย                                                                  |
| `message.subject`                 | ใช่         | หัวข้ออีเมล ต้องไม่เป็นค่าว่าง ไม่มีอีโมจิ และไม่ขึ้นต้นด้วย `Re:` หรือ `Fwd:` |
| `message.template_id`             | ตามเงื่อนไข | ระบุเมื่อไม่ได้ส่ง `message.html`; อ้างอิงเทมเพลตที่บันทึกไว้                        |
| `message.html`                    | ตามเงื่อนไข | เนื้อหา HTML ที่เข้ารหัส Base64 เมื่อไม่ได้อ้างอิงเทมเพลต                            |
| `message.selector`                | ไม่ใช่      | DKIM selector ที่ต้องการใช้เมื่อโดเมนผู้ส่งมี DKIM record ที่ใช้งานอยู่มากกว่าหนึ่งรายการ |
| `message.template_values`         | ไม่ใช่      | ชุด key/value ที่ส่งเข้าไปใช้ระหว่างเรนเดอร์เทมเพลต                                  |
| `message.attachments[].type`      | ไม่ใช่      | กำหนดชนิดของ `content`: `ASSET` อ้างอิง asset ID จาก asset manager, `RAW` ใช้ Base64 แบบ inline |
| `message.attachments[].content`   | ไม่ใช่      | payload หรือรหัส asset ขึ้นอยู่กับ `type`                                            |
| `message.attachments[].file_name` | ไม่ใช่      | ชื่อไฟล์ดาวน์โหลดที่แสดงให้ผู้รับเห็น                                                |

---

### การตอบกลับ

**201 – Created**

สร้างข้อความสำเร็จ

#### เฮดเดอร์
| ฟิลด์ | คำอธิบาย |
| --- | --- |
| `x-ratelimit-remaining` | จำนวนคำขอที่ยังส่งได้ก่อนถึงขีดจำกัด |
| `x-ratelimit-reset` | เวลา epoch หรือจำนวนวินาทีจนกว่าช่วง rate limit ปัจจุบันจะรีเซ็ต |
| `x-ratelimit-limit` | จำนวนคำขอทั้งหมดที่อนุญาตในช่วง rate limit ปัจจุบัน |
| `retry-after` | เวลาที่ไคลเอนต์ควรรอเป็นวินาทีก่อนลองส่งคำขอใหม่ |

```json
{
    "id": "<message id>",
    "status": "<status>",
    "updated_at": "<timestamp>",
    "cost": <number>
}
```

| ฟิลด์        | คำอธิบาย                                 |
| ------------ | ---------------------------------------- |
| `id`         | รหัสข้อความธุรกรรม                       |
| `status`     | สถานะข้อความ                             |
| `updated_at` | เวลาที่อัปเดตล่าสุด                      |
| `cost`       | เครดิตที่ใช้                               |

#### ข้อผิดพลาดที่อาจเกิดขึ้น

| สถานะ | ประเภท | เกิดขึ้นเมื่อ |
| ------ | ------ | ------------- |
| 400    | `ValidationError`                   | บอดี้คำขอผิดรูปแบบหรือไม่ผ่านการตรวจสอบ |
| 401    | `UnauthorizedError`                 | ข้อมูลยืนยันตัวตนไม่ถูกต้อง หมดอายุ หรือถูกจำกัดด้วย CIDR |
| 404    | `TemplateNotFoundError`             | ไม่พบ `template_id` ที่ระบุ |
| 404    | `AssetNotFoundError`                | ไม่พบไฟล์แนบชนิด `ASSET` อย่างน้อยหนึ่งรายการ |
| 406    | `InvalidSenderError`                | โดเมนผู้ส่งไม่ได้ลงทะเบียนหรือไม่สามารถใช้งานได้ |
| 406    | `InvalidRecipientError`             | อีเมลหรือโดเมนของผู้รับไม่สามารถรับได้ |
| 406    | `InsufficientCreditError`           | เครดิตในบัญชีไม่เพียงพอ |
| 406    | `TotalAttachmentSizeExceededError`  | ขนาดรวมของไฟล์แนบเกินขีดจำกัด |
| 406    | `TotalAttachmentCountExceededError` | จำนวนไฟล์แนบเกินขีดจำกัด |
| 429    | `TooManyRequestsError`              | จำนวนคำขอเกินช่วง rate limit ปัจจุบัน |
| 500    | `InternalServerError`               | เกิดข้อผิดพลาดภายในระบบที่ไม่คาดคิด |

**400 – Bad Request**

เมื่อส่งบอดี้คำขอผิดรูปแบบ ระบบจะตอบกลับ `ValidationError`

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

โทเค็นหมดอายุ หรือถูกจำกัด IP ที่สามารถขอส่งข้อความได้

```json
{
    "type": "UnauthorizedError",
    "message": "Invalid credentials or restricted by CIDR."
}
```

**404 – Not Found**

กรณี `template_id` ที่ระบุไม่พบ ระบบจะตอบกลับ `TemplateNotFoundError`

```json
{
    "type": "TemplateNotFoundError",
    "message": "The template specified could not be found."
}
```

**404 – Not Found**

เมื่อไฟล์แนบหนึ่งรายการขึ้นไปที่อ้างอิงใน `attachments[].content` โดยมีชนิดเป็น `ASSET` ไม่มีอยู่ใน asset manager ระบบจะตอบกลับ `AssetNotFoundError`

```json
{
    "type": "AssetNotFoundError",
    "message": "One or more of the specified assets could not be found."
}
```

**406 – Not Acceptable**

เมื่อโดเมนผู้ส่งที่ระบุไม่ใช่หนึ่งในโดเมนผู้ส่งที่ลงทะเบียนไว้ หรือโดเมนที่ลงทะเบียนตรวจสอบ DKIM, SPF หรือ DMARC ไม่ผ่าน ระบบจะตอบกลับ `InvalidSenderError`

```json
{
    "type": "InvalidSenderError",
    "message": "The specified sender domain is not registered."
}
```

**406 – Not Acceptable**

เมื่ออีเมลผู้รับไม่ถูกต้อง เช่นผิดรูปแบบของอีเมลล์  ระบบจะตอบกลับ `InvalidRecipientError`

```json
{
    "type": "InvalidRecipientError",
    "message": "The specified recipient is invalid."
}
```

**406 – Not Acceptable**

เมื่อยอดเครดิตในบัญชีไม่เพียงพอ ระบบจะตอบกลับ `InsufficientCreditError`

```json
{
    "type": "InsufficientCreditError",
    "message": "Your credit balance is insufficient."
}
```

**406 – Not Acceptable**

เมื่อขนาดรวมของไฟล์แนบที่ระบุเกินขีดจำกัด ระบบจะตอบกลับ `TotalAttachmentSizeExceededError`

```json
{
    "type": "TotalAttachmentSizeExceededError",
    "message": "The total size of attachments exceeds the allowed limit."
}
```

**406 – Not Acceptable**

เมื่อจำนวนไฟล์แนบเกินขีดจำกัดของ tenant ระบบจะตอบกลับ `TotalAttachmentCountExceededError`

```json
{
    "type": "TotalAttachmentCountExceededError",
    "message": "The total number of attachments exceeds the allowed limit."
}
```

**429 – TooManyRequests**

เมื่อไคลเอนต์ส่งคำขอส่งข้อความเกินจำนวนที่อนุญาตในช่วง rate limit ปัจจุบัน ระบบจะตอบกลับ `TooManyRequestsError`

```json
{
    "type": "TooManyRequestsError",
    "message": "Rate limit exceeded, retry in 5 seconds."
}
```

**500 – Internal Server Error**

เมื่อเกิดข้อผิดพลาดร้ายแรงภายในระบบ ระบบจะตอบกลับ `InternalServerError`

```json
{
    "type": "InternalServerError",
    "message": "Something went wrong, please try again later."
}
```

### ตัวอย่าง: ส่ง HTML แบบ inline ด้วย `curl`

เข้ารหัส HTML เป็น Base64 (ไม่มีขึ้นบรรทัดใหม่) แล้วใส่ใน `message.html`

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

### ตัวอย่าง: ใช้เทมเพลตและตัวแปรด้วย `curl`

ใช้ `template_id` ที่บันทึกไว้ พร้อมส่งค่าเรนเดอร์ใน `template_values`

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

### ตัวอย่าง: ใช้เทมเพลต ใส่ตัวแปร และแนบไฟล์แบบ `ASSET` ด้วย `curl`

อ้างอิงไฟล์แนบที่อัปโหลดแล้วด้วย asset ID

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

### ตัวอย่าง: ใช้เทมเพลต ใส่ตัวแปร และแนบไฟล์ Base64 แบบ `RAW` ด้วย `curl`

เข้ารหัสไฟล์เป็น Base64 แล้วตั้ง `attachments[].type` เป็น `RAW`

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

## สอบถามสถานะข้อความธุรกรรม

### คำขอ

```http
GET /v1/messages/{transactional_message_id}
```

#### พารามิเตอร์ในพาธ

| ฟิลด์                      | บังคับ | คำอธิบาย                     |
| -------------------------- | ------ | ---------------------------- |
| `transactional_message_id` | ใช่   | รหัสข้อความธุรกรรมที่ต้องการ |

#### เฮดเดอร์

| ฟิลด์           | บังคับ | คำอธิบาย             |
| --------------- | ------ | -------------------- |
| `Authorization` | ใช่   | ใช้ `Bearer <TOKEN>` |

### การตอบกลับ

**200 – OK**

ส่งกลับรายละเอียดของข้อความธุรกรรมที่ระบุ

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

| ฟิลด์                            | ชนิด                | คำอธิบาย                                                |
| -------------------------------- | ------------------- | ------------------------------------------------------- |
| `id`                             | string              | รหัสข้อความธุรกรรม                                       |
| `result`                         | EmailMessageResult  | ผลลัพธ์ระดับสูง: `Processing`, `Ok`, `HardBounce`, `SoftBounce` หรือ `Error` |
| `status`                         | MessageLogStatus    | สถานะการส่งล่าสุดที่บันทึกไว้ของข้อความ                 |
| `sender`                         | string              | อีเมล/ชื่อผู้ส่งที่ใช้                                  |
| `recipient`                      | string              | อีเมลผู้รับ                                             |
| `opened`                         | number              | จำนวนการเปิดอีเมลที่ติดตามได้ทั้งหมด              |
| `clicked`                        | number              | จำนวนการคลิกลิงก์ที่ติดตามได้ทั้งหมด              |
| `cost`                           | number              | เครดิตที่เกิดขึ้นจากการส่งข้อความ                       |
| `created_at`                     | Date                | เวลาที่สร้างรายการล็อก                                  |
| `updated_at`                     | Date                | เวลาที่รายการล็อกเปลี่ยนแปลงล่าสุด                      |
| `retry_attempts`                 | number              | จำนวนครั้งที่ลองส่งซ้ำไปแล้วสำหรับข้อความนี้            |
| `status_logs`                    | StatusLogResponse[] | รายการการเปลี่ยนสถานะการส่งตามลำดับเวลา                 |
| `status_logs[].result`           | EmailMessageResult  | ผลลัพธ์ระดับสูงที่บันทึกในล็อกสถานะนี้                  |
| `status_logs[].status`           | MessageLogStatus    | สถานะที่บันทึกไว้ในรายการล็อก                           |
| `status_logs[].response`         | object      | ออบเจกต์ตอบกลับจาก transport/MTA หรือ `null` สำหรับล็อกสถานะของระบบ |
| `status_logs[].response.code`    | number      | โค้ดตอบกลับจากผู้ให้บริการปลายทาง/MTA                          |
| `status_logs[].response.content` | string      | ข้อความตอบกลับจากผู้ให้บริการปลายทาง                        |
| `status_logs[].details`          | string              | รายละเอียดระบบเพิ่มเติมสำหรับล็อกสถานะของระบบ           |
| `status_logs[].timestamp`        | number              | เวลาจากผู้ให้บริการปลายทาง (epoch ms)              |
| `status_logs[].created`          | number              | เวลาที่บันทึกสถานะ (epoch ms)                       |

#### ความหมายของ `MessageLogStatus`

| Result       | Status                 | ความหมาย                                                                                                      |
| ------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------- |
| `Processing` | `Created`              | บันทึกคำขอส่งข้อความแล้ว แต่ยังไม่ได้รับการตอบรับจาก message transport agent |
| `Processing` | `Accepted`             | message transport agent ตอบรับคำขอส่งข้อความแล้ว |
| `Processing` | `Deferred`             | message transport agent เลื่อนการส่งข้อความออกไป และจะลองส่งใหม่ภายหลัง |
| `Ok`         | `Delivered`            | ผู้ให้บริการของผู้รับยืนยันว่าเซิร์ฟเวอร์ของผู้รับได้รับข้อความแล้ว |
| `Error`      | `TimedOut`             | การดำเนินการภายในระบบหมดเวลาก่อนเสร็จสมบูรณ์ |
| `Error`      | `TechnicalError`       | ข้อผิดพลาดภายในระบบทำให้ไม่สามารถส่งข้อความได้ |
| `Error`      | `Feedback`             | ผู้รับส่ง feedback/complaint ผ่าน feedback loop |
| `Error`      | `InsufficientCredit`   | ส่งไม่สำเร็จเพราะ tenant มีเครดิตไม่เพียงพอ |
| `Error`      | `Blocked`              | ผู้รับถูกบล็อกโดย blocklist ของ NipaMail |
| `HardBounce` | `InvalidRecipient`     | อีเมลล์ของผู้รับไม่ถูกต้อง หรือ mailbox ไม่มีอยู่จริง |
| `HardBounce` | `InactiveMailbox`      | mailbox มีอยู่ แต่ถูกปิดใช้งาน ถูกระงับ หรือปิดบัญชีแล้ว |
| `HardBounce` | `BadDomain`            | โดเมนผู้รับตั้งค่าไม่ถูกต้อง หรือไม่มี MX record ที่ใช้งานได้ |
| `HardBounce` | `InvalidSender`        | ผู้ส่ง/โดเมนไม่ได้รับอนุญาต หรือไม่ผ่านการตรวจสอบ DKIM, SPF หรือ DMARC |
| `HardBounce` | `NoAnswerFromHost`     | พบที่อยู่ Host ผู้ให้บริการของผู้รับ แต่ Host ไม่ตอบสนองกับคำขอส่งข้อความ |
| `HardBounce` | `DNSFailure`           | ผู้รับค้นหา DNS ล้มเหลว เช่น ไม่มี MX |
| `HardBounce` | `MessageExpired`       | ช่วงเวลาการส่งหมดลงก่อนที่ message transport agent จะส่งข้อความสำเร็จ |
| `HardBounce` | `ProviderRejected`     | ผู้ให้บริการของผู้รับปฏิเสธการรับข้อความและมี error message ที่ไม่เข้ากับ hard bounce อื่น ๆ |
| `SoftBounce` | `QuotaIssues`          | Mailbox ของผู้รับเต็มหรือไฟล์แนบมีขนาดใหญ่เกินที่ผู้รับจะรับได้ |
| `SoftBounce` | `BadConnection`        | การเชื่อมต่อไปยังผู้ให้บริการของผู้รับล้มเหลวหรือถูกตัด |
| `SoftBounce` | `RoutingErrors`        | ผู้ให้บริการของผู้รับปฏิเสธการ relay หรือไม่สามารถ route ไปยังผู้รับได้ |
| `SoftBounce` | `ProtocolErrors`       | ลำดับคำสั่งของ SMTP ถูกปฏิเสธหรือไม่สามารถเชื่อมต่อด้วยการขอเข้ารหัส (STARTTLS) กับผู้ให้บริการของผู้รับได้ |
| `SoftBounce` | `AuthenticationFailed` | ผู้รับตรวจสอบ DKIM, DMARC หรือ SPF ของผู้ส่งไม่ผ่าน |
| `SoftBounce` | `PolicyRelated`        | ถูกปฏิเสธตามนโยบายของผู้ให้บริการของผู้รับ หรือขัดต่อข้อจำกัดการใช้งาน |
| `SoftBounce` | `SpamContent`          | เนื้อหาข้อความถูกจัดว่าเป็นสแปมหรือไวรัส |
| `SoftBounce` | `SpamBlock`            | การส่งถูกบล็อกเนื่องจากชื่อเสียงของผู้ส่ง/IP/โดเมน หรือ blocklist |
| `SoftBounce` | `GenericSoftBounce`    | ผู้ให้บริการของผู้รับปฏิเสธการรับข้อความและมี error message ที่ไม่เข้ากับ soft bounce อื่น ๆ |

#### รายละเอียดการส่งซ้ำและการตัดเครดิตของ bounce

เครื่องหมาย `*` หมายถึงมีการตัดเครดิตเมื่อไม่พบผู้รับนั้นในครั้งแรกเท่านั้น ครั้งถัดไปจะถูกบล็อกโดย blocklist ของ NipaMail

| Result       | Status                 | ส่งซ้ำ | ตัดเครดิต |
| ------------ | ---------------------- | ------ | --------- |
| `HardBounce` | `InvalidRecipient`     | ไม่ใช่ | ใช่ *     |
| `HardBounce` | `InactiveMailbox`      | ไม่ใช่ | ใช่ *     |
| `HardBounce` | `BadDomain`            | ไม่ใช่ | ไม่ใช่    |
| `HardBounce` | `InvalidSender`        | ไม่ใช่ | ใช่       |
| `HardBounce` | `NoAnswerFromHost`     | ไม่ใช่ | ไม่ใช่    |
| `HardBounce` | `DNSFailure`           | ไม่ใช่ | ไม่ใช่    |
| `HardBounce` | `MessageExpired`       | ไม่ใช่ | ไม่ใช่    |
| `HardBounce` | `ProviderRejected`     | ไม่ใช่ | ใช่       |
| `SoftBounce` | `QuotaIssues`          | ใช่    | ใช่       |
| `SoftBounce` | `BadConnection`        | ใช่    | ไม่ใช่    |
| `SoftBounce` | `RoutingErrors`        | ใช่    | ไม่ใช่    |
| `SoftBounce` | `ProtocolErrors`       | ใช่    | ไม่ใช่    |
| `SoftBounce` | `AuthenticationFailed` | ใช่    | ไม่ใช่    |
| `SoftBounce` | `PolicyRelated`        | ไม่ใช่ | ใช่       |
| `SoftBounce` | `SpamContent`          | ไม่ใช่ | ใช่       |
| `SoftBounce` | `SpamBlock`            | ใช่    | ใช่       |
| `SoftBounce` | `GenericSoftBounce`    | ใช่ | ไม่ใช่    |

#### ข้อผิดพลาดที่อาจเกิดขึ้น

| สถานะ | ประเภท | เกิดขึ้นเมื่อ |
| ------ | ------ | ------------- |
| 401    | `UnauthorizedError`    | ข้อมูลยืนยันตัวตนไม่ถูกต้อง หมดอายุ หรือถูกจำกัดด้วย CIDR |
| 404    | `MessageNotFoundError` | ไม่พบข้อความธุรกรรมที่ระบุ |

**401 – Unauthorized**

โทเค็นหมดอายุ หรือถูกจำกัด IP ที่สามารถขอส่งข้อความได้

```json
{
    "type": "UnauthorizedError",
    "message": "Invalid credentials or restricted by CIDR."
}
```

**404 – Not Found**

เมื่อไม่พบข้อความธุรกรรมที่ระบุ ระบบจะตอบกลับ `MessageNotFoundError`

```json
{
    "type": "MessageNotFoundError",
    "message": "The message specified could not be found."
}
```

## ส่งข้อความธุรกรรมซ้ำ

**แพ็กเกจ: Enterprise**

API นี้ช่วยส่งข้อความซ้ำโดยใช้บอดี้คำขอเดียวกับการส่งครั้งแรก เหมาะสำหรับกรณีที่การส่งข้อความก่อนหน้าล้มเหลวและเข้าเงื่อนไขให้ส่งซ้ำได้

### คำขอ

```http
PUT /v1/messages/{transactional_message_id}
```

#### เฮดเดอร์

| ฟิลด์           | บังคับ | คำอธิบาย             |
| --------------- | ------ | -------------------- |
| `Content-Type`  | ใช่   | `application/json`   |
| `Authorization` | ใช่   | ใช้ `Bearer <TOKEN>` |

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

| ฟิลด์                             | บังคับ      | คำอธิบาย                                                                             |
| --------------------------------- | ----------- | ------------------------------------------------------------------------------------ |
| `type`                            | ใช่         | ใช้ `EMAIL` สำหรับข้อความอีเมล                                                        |
| `message.sender`                  | ใช่         | โดเมนผู้ส่งที่ลงทะเบียนแล้ว                                                          |
| `message.recipient`               | ใช่         | อีเมลผู้รับหนึ่งราย                                                                  |
| `message.subject`                 | ใช่         | หัวข้ออีเมล ต้องไม่เป็นค่าว่าง ไม่มีอีโมจิ และไม่ขึ้นต้นด้วย `Re:` หรือ `Fwd:` |
| `message.template_id`             | ตามเงื่อนไข | ระบุเมื่อไม่ได้ส่ง `message.html`; อ้างอิงเทมเพลตที่บันทึกไว้                        |
| `message.html`                    | ตามเงื่อนไข | เนื้อหา HTML ที่เข้ารหัส Base64 เมื่อไม่ได้อ้างอิงเทมเพลต                            |
| `message.selector`                | ไม่ใช่      | DKIM selector ที่ต้องการใช้เมื่อโดเมนผู้ส่งมี DKIM record ที่ใช้งานอยู่มากกว่าหนึ่งรายการ |
| `message.template_values`         | ไม่ใช่      | ชุด key/value ที่ส่งเข้าไปใช้ระหว่างเรนเดอร์เทมเพลต                                  |
| `message.attachments[].type`      | ไม่ใช่      | กำหนดชนิดของ `content`: `ASSET` อ้างอิง asset ID จาก asset manager, `RAW` ใช้ Base64 แบบ inline |
| `message.attachments[].content`   | ไม่ใช่      | payload หรือรหัส asset ขึ้นอยู่กับ `type`                                            |
| `message.attachments[].file_name` | ไม่ใช่      | ชื่อไฟล์ดาวน์โหลดที่แสดงให้ผู้รับเห็น                                                |

---

### การตอบกลับ

**200 – OK**

ส่งข้อความซ้ำสำเร็จ

```json
{
    "status": "<status>",
    "updated_at": "<timestamp>",
    "current_attempt": <number>
}
```

| ฟิลด์ | คำอธิบาย |
| --- | --- |
| `status` | สถานะข้อความ |
| `updated_at` | เวลาที่อัปเดตล่าสุด |
| `current_attempt` | จำนวนครั้งที่ลองส่งซ้ำ |

#### ข้อผิดพลาดที่อาจเกิดขึ้น

| สถานะ | ประเภท | เกิดขึ้นเมื่อ |
| ------ | ------ | ------------- |
| 400    | `ValidationError`                     | บอดี้คำขอผิดรูปแบบหรือไม่ผ่านการตรวจสอบ |
| 401    | `UnauthorizedError`                   | ข้อมูลยืนยันตัวตนไม่ถูกต้อง หมดอายุ หรือถูกจำกัดด้วย CIDR |
| 404    | `MessageNotFoundError`                | ไม่พบ `transactional_message_id` ที่ระบุ |
| 406    | `RetryDataMismatchError`              | ข้อมูลสำหรับการส่งซ้ำไม่ตรงกับข้อมูลข้อความเดิม |
| 406    | `MessageStatusNotAllowedToRetryError` | สถานะข้อความไม่อนุญาตให้ส่งซ้ำ |
| 406    | `MaxRetryAttemptsExceededError`       | จำนวนครั้งสูงสุดในการลองส่งซ้ำถูกใช้ครบแล้ว |
| 500    | `InternalServerError`                 | เกิดข้อผิดพลาดภายในระบบที่ไม่คาดคิด |

**400 – Bad Request**

เมื่อส่งบอดี้คำขอผิดรูปแบบ ระบบจะตอบกลับ `ValidationError`

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

โทเค็นหมดอายุ หรือถูกจำกัด IP ที่สามารถขอส่งข้อความได้

```json
{
    "type": "UnauthorizedError",
    "message": "Invalid credentials or restricted by CIDR."
}
```

**404 – Not Found**

เมื่อ `transactional_message_id` ที่ระบุไม่มีอยู่ในประวัติการส่ง ระบบจะตอบกลับ `MessageNotFoundError`

```json
{
    "type": "MessageNotFoundError",
    "message": "The message specified could not be found."
}
```

**406 – Not Acceptable**

เมื่อข้อมูลสำหรับการส่งซ้ำไม่ตรงกับข้อมูลข้อความเดิม ระบบจะตอบกลับ `RetryDataMismatchError`

```json
{
    "type": "RetryContentMismatchError",
    "message": "The content of the retried message does not match the original message."
}
```

**406 – Not Acceptable**

เมื่อสถานะข้อความไม่อนุญาตให้ส่งซ้ำ ระบบจะตอบกลับ `MessageStatusNotAllowedToRetryError`

```json
{
    "type": "RetryDeniedError",
    "message": "The specified message is not eligible for retry."
}
```

**406 – Not Acceptable**

เมื่อจำนวนครั้งสูงสุดในการลองส่งซ้ำของข้อความนี้ถูกใช้ครบแล้ว ระบบจะตอบกลับ `MaxRetryAttemptsExceededError`

```json
{
    "type": "MaxRetryAttemptsExceededError",
    "message": "The maximum number of retry attempts on this message has been exceeded."
}
```

**500 – Internal Server Error**

เมื่อเกิดข้อผิดพลาดร้ายแรงภายในระบบ ระบบจะตอบกลับ `InternalServerError`

```json
{
    "type": "InternalServerError",
    "message": "Something went wrong, please try again later."
}
```

### ตัวอย่าง: ส่ง HTML แบบ inline ซ้ำด้วย `curl`

เข้ารหัส HTML เป็น Base64 (ไม่มีขึ้นบรรทัดใหม่) แล้วใส่ใน `message.html`

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
