# Webhook

NipaMail จะส่ง HTTP webhook เมื่อสถานะการส่งข้อความธุรกรรมเปลี่ยนแปลง โดยต้องตั้งค่า webhook แยกสำหรับแต่ละแอปพลิเคชันใน NipaMail Portal

## การตั้งค่า Webhook

1. เปิดเมนู **Application** ใน NipaMail Portal แล้วเลือกแอปพลิเคชัน
2. เปิดแท็บ **Webhook Configuration**
3. ป้อน URL ของ endpoint ที่รองรับคำขอ HTTP `POST` สำหรับระบบ production ควรใช้ HTTPS
4. ตั้งค่าของเฮดเดอร์ `Authorization` ซึ่งเป็นข้อมูลที่บังคับ
5. เลือก **Verify & Save** เพื่อทดสอบและบันทึก endpoint

NipaMail ต้องสามารถเข้าถึง endpoint ดังกล่าวผ่านเครือข่ายสาธารณะได้ โปรดเก็บค่า authorization ที่ตั้งค่าไว้เป็นความลับและตรวจสอบค่าดังกล่าวก่อนประมวลผล webhook

## คำขอ Webhook

NipaMail จะส่ง JSON payload ไปยัง endpoint ที่ตั้งค่าไว้

```http
POST <your-webhook-url>
Content-Type: application/json
Authorization: <configured-authorization-value>
```

### ตัวอย่าง Payload

```json
{
  "id": "019fc698-55d4-7048-b46d-3bd4e4c0f3ae",
  "result": "Ok",
  "status": "Delivered",
  "sender": "No-Reply <noreply@example.com>",
  "recipient": "test@example.com",
  "cost": 1,
  "response": {
    "code": 250,
    "content": "OK"
  },
  "timestamp": 1785743300,
  "created": 1785743300
}
```

| ฟิลด์ | ชนิด | คำอธิบาย |
| --- | --- | --- |
| `id` | string | รหัสข้อความธุรกรรม |
| `result` | string | ผลลัพธ์ระดับสูงของการส่ง เช่น `Ok` |
| `status` | MessageLogStatus | สถานะการส่งล่าสุดที่บันทึกไว้ของข้อความ |
| `sender` | string | ชื่อและอีเมลผู้ส่งที่ใช้ส่งข้อความ |
| `recipient` | string | อีเมลผู้รับ |
| `cost` | number | จำนวนเครดิตที่ใช้สำหรับข้อความ |
| `response.code` | number | โค้ดตอบกลับจาก SMTP หรือผู้ให้บริการปลายทาง |
| `response.content` | string | ข้อความตอบกลับจาก SMTP หรือผู้ให้บริการปลายทาง |
| `timestamp` | number | เวลาที่เกิดเหตุการณ์การส่งในรูปแบบ Unix timestamp หน่วยวินาที |
| `created` | number | เวลาที่ระบบบันทึกเหตุการณ์สถานะในรูปแบบ Unix timestamp หน่วยวินาที |

สำหรับค่า `status` ที่เป็นไปได้ โปรดดูหัวข้อ [ความหมายของ `MessageLogStatus`](/th-th/transactional.md?id=ความหมายของ-messagelogstatus)

## การตอบรับ Webhook

ตอบกลับด้วย HTTP status code ในกลุ่ม `2xx` หลังจากรับ webhook ไว้ประมวลผลแล้ว ควรออกแบบการประมวลผลแต่ละเหตุการณ์ให้เป็นแบบ idempotent เนื่องจากเหตุการณ์เดิมอาจถูกส่งมามากกว่าหนึ่งครั้ง

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "received": true
}
```

ควรตอบกลับโดยเร็ว หากต้องใช้เวลาประมวลผลนาน ให้บันทึกเหตุการณ์หรือเพิ่มลงในคิวก่อนส่งการตอบรับ

## ตัวอย่าง Node.js / Express

```js
const express = require("express");

const app = express();

app.use(express.json());

app.post("/webhooks/nipamail", (req, res) => {
  const authorization = req.header("Authorization");

  if (authorization !== process.env.NIPAMAIL_WEBHOOK_AUTHORIZATION) {
    return res.status(401).json({ message: "Invalid authorization" });
  }

  const event = req.body;

  // Store or enqueue the event before acknowledging it.
  console.log(event.id, event.status);

  return res.status(200).json({ received: true });
});
```

กำหนดค่า `NIPAMAIL_WEBHOOK_AUTHORIZATION` ให้ตรงกับค่าที่ตั้งไว้ใน Portal หากค่า authorization เป็นข้อมูลลับที่มีความสำคัญต่อความปลอดภัย ควรเปรียบเทียบค่าด้วยวิธี constant-time แทนการเปรียบเทียบสตริงโดยตรง

## รายการตรวจสอบ

| รายการ | คำแนะนำ |
| --- | --- |
| Endpoint | ใช้ HTTPS URL ที่ NipaMail เข้าถึงได้ |
| Method | รองรับ HTTP `POST` |
| Content type | รองรับ `application/json` |
| Authorization | ตรวจสอบค่าให้ตรงกับค่าที่ตั้งไว้ใน Portal |
| การตอบรับ | ตอบกลับด้วย `2xx` หลังจากรับข้อมูลแล้ว |
| ข้อมูลซ้ำ | ตรวจสอบ `id`, `status` และ `timestamp` เพื่อป้องกันการประมวลผลซ้ำ |
| บันทึกระบบ | บันทึกข้อผิดพลาด แต่อย่าบันทึกค่า authorization |
