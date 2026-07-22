# Webhook

NipaMail จะส่ง webhook เมื่อสถานะของข้อความธุรกรรมเปลี่ยนแปลง โดยจะส่งเฉพาะแอปพลิเคชันที่ตั้งค่า webhook URL และยืนยันแล้วเท่านั้น

## การส่ง Webhook

NipaMail จะส่งคำขอ HTTP `POST` ไปยัง webhook URL ที่ตั้งค่าไว้

### เฮดเดอร์

| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `Content-Type` | ใช่ | เป็น `application/json` เสมอ |
| `Authorization` | ใช่ | ค่า authorization ที่ตั้งค่าไว้ใน webhook โดย NipaMail จะส่งต่อให้ตามเดิม |
| `X-Nipa-Signature` | ตามเงื่อนไข | ลายเซ็น HMAC-SHA256 ของ JSON request body จะมีเมื่อ credential ที่ใช้ส่งข้อความมี `client_secret` |

### Request body

```json
{
    "id": "<message id>",
    "status": "<message status>",
    "result": "<message result>",
    "sender": "Sender <sender@example.com>",
    "recipient": "recipient@example.com",
    "cost": "1.00",
    "response": {
        "code": 250,
        "content": "<provider response>"
    },
    "timestamp": 1767225600000,
    "created": 1767225600000
}
```

| ฟิลด์ | ชนิด | คำอธิบาย |
| --- | --- | --- |
| `id` | string | รหัสข้อความธุรกรรม |
| `status` | MessageLogStatus | สถานะการส่งล่าสุดที่บันทึกไว้ของข้อความ |
| `result` | EmailMessageResult | ผลลัพธ์ระดับสูง: `Processing`, `Ok`, `HardBounce`, `SoftBounce` หรือ `Error` |
| `sender` | string | อีเมล/ชื่อผู้ส่งที่ใช้เดิม |
| `recipient` | string | อีเมลผู้รับ |
| `cost` | string | เครดิตที่ใช้ แสดงเป็นทศนิยมสองตำแหน่ง |
| `response.code` | number | โค้ดตอบกลับจาก transport/MTA เมื่อมีข้อมูล |
| `response.content` | string | ข้อความตอบกลับจากผู้ให้บริการปลายทางเมื่อมีข้อมูล |
| `timestamp` | number | เวลาจากผู้ให้บริการปลายทาง (epoch ms) |
| `created` | number | เวลาที่บันทึกสถานะ (epoch ms) |

สำหรับค่า `status` ที่เป็นไปได้ โปรดดูหน้า [`MessageLogStatus` meaning](/transactional.md?id=messagelogstatus-meaning)

## การตอบกลับ Webhook

ให้ตอบกลับด้วย HTTP status code ใดก็ได้ในกลุ่ม `2xx` เพื่อยืนยันว่าได้รับ webhook แล้ว

NipaMail จะตาม redirect ได้สูงสุด 5 ครั้ง และรอการตอบกลับสูงสุด 10 วินาทีต่อคำขอ หากคำขอล้มเหลว timeout หรือได้รับ response ที่ไม่ใช่ success ระบบจะ retry สูงสุด 3 ครั้ง โดยใช้ exponential backoff เริ่มต้นที่ 5 วินาที

### ตัวอย่าง response ที่แนะนำ

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
    "received": true
}
```

response body ของคุณจะถูกเก็บไว้ใน webhook delivery log เท่านั้น และจะไม่เปลี่ยนสถานะการส่งข้อความ

## ตรวจสอบ HMAC signature

เมื่อมีเฮดเดอร์ `X-Nipa-Signature` ให้ตรวจสอบว่าลายเซ็นที่ NipaMail ส่งมาถูกต้องก่อนประมวลผล webhook

NipaMail เป็นผู้สร้างค่า `X-Nipa-Signature` ด้วยรูปแบบนี้:

```text
hex(HMAC_SHA256(client_secret, raw_json_body))
```

รายละเอียดสำคัญ:

| รายการ | คำอธิบาย |
| --- | --- |
| Algorithm | HMAC-SHA256 |
| Header | `X-Nipa-Signature` |
| Secret | `client_secret` ของ application credential |
| Signed payload | raw JSON request body ตามที่ NipaMail ส่งมาแบบตรงตัว |
| Encoding | lowercase hexadecimal digest |

`client_secret` คือค่าจากหน้า application setting ของ credential ที่ใช้ส่งข้อความ โดยใช้ `client_secret` เพื่อตรวจสอบลายเซ็นที่ได้รับจาก NipaMail

### ตัวอย่าง Node.js / Express

```js
const crypto = require("crypto");
const express = require("express");

const app = express();

app.post(
    "/webhooks/nipamail",
    express.raw({ type: "application/json" }),
    (req, res) => {
        const nipaSignature = req.header("X-Nipa-Signature");
        const ClientSecret = process.env.NIPAMAIL_CLIENT_SECRET;

        if (!verifyNipaSignature(nipaSignature, req.body, ClientSecret)) {
            return res.status(401).json({ message: "Invalid signature" });
        }

        const event = JSON.parse(req.body.toString("utf8"));

        // Process the webhook event.
        res.status(200).json({ received: true });
    }
);

function verifyNipaSignature(nipaSignature, rawBody, ClientSecret) {
    const expectedSignature = crypto
        .createHmac("sha256", ClientSecret)
        .update(rawBody)
        .digest("hex");

    const signatureBuffer = Buffer.from(nipaSignature || "", "hex");
    const expectedBuffer = Buffer.from(expectedSignature, "hex");

    return signatureBuffer.length === expectedBuffer.length &&
        crypto.timingSafeEqual(signatureBuffer, expectedBuffer);
}
```

### ตัวอย่าง Python / Flask

```python
import hashlib
import hmac
import json
import os
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.post("/webhooks/nipamail")
def nipamail_webhook():
    nipa_signature = request.headers.get("X-Nipa-Signature", "")
    ClientSecret = os.environ["NIPAMAIL_CLIENT_SECRET"].encode("utf-8")
    raw_body = request.get_data()

    if not verify_nipa_signature(nipa_signature, raw_body, ClientSecret):
        return jsonify({"message": "Invalid signature"}), 401

    event = json.loads(raw_body)

    # Process the webhook event.
    return jsonify({"received": True}), 200

def verify_nipa_signature(nipa_signature, raw_body, ClientSecret):
    expected_signature = hmac.new(ClientSecret, raw_body, hashlib.sha256).hexdigest()

    return hmac.compare_digest(nipa_signature, expected_signature)
```

### ตัวอย่าง Java / Spring Boot

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class NipaMailWebhookController {
    private final ObjectMapper objectMapper = new ObjectMapper();
    private final String ClientSecret = System.getenv("NIPAMAIL_CLIENT_SECRET");

    @PostMapping("/webhooks/nipamail")
    public ResponseEntity<?> receiveWebhook(
            @RequestHeader(value = "X-Nipa-Signature", required = false) String nipaSignature,
            HttpServletRequest request
    ) throws Exception {
        byte[] rawBody = request.getInputStream().readAllBytes();

        if (!verifyNipaSignature(nipaSignature, rawBody, ClientSecret)) {
            return ResponseEntity.status(401).body("{\"message\":\"Invalid signature\"}");
        }

        JsonNode event = objectMapper.readTree(rawBody);

        // Process the webhook event.
        return ResponseEntity.ok("{\"received\":true}");
    }

    private static boolean verifyNipaSignature(
            String nipaSignature,
            byte[] rawBody,
            String ClientSecret
    ) throws NoSuchAlgorithmException, InvalidKeyException {
        String expectedSignature = hmacSha256Hex(ClientSecret, rawBody);

        return timingSafeEquals(nipaSignature, expectedSignature);
    }

    private static String hmacSha256Hex(String secret, byte[] payload)
            throws NoSuchAlgorithmException, InvalidKeyException {
        Mac mac = Mac.getInstance("HmacSHA256");
        SecretKeySpec keySpec = new SecretKeySpec(
                secret.getBytes(StandardCharsets.UTF_8),
                "HmacSHA256"
        );
        mac.init(keySpec);
        return HexFormat.of().formatHex(mac.doFinal(payload));
    }

    private static boolean timingSafeEquals(String signature, String expected) {
        if (signature == null) {
            return false;
        }

        byte[] signatureBytes;
        byte[] expectedBytes;

        try {
            signatureBytes = HexFormat.of().parseHex(signature);
            expectedBytes = HexFormat.of().parseHex(expected);
        } catch (IllegalArgumentException ex) {
            return false;
        }

        return signatureBytes.length == expectedBytes.length
                && MessageDigest.isEqual(signatureBytes, expectedBytes);
    }
}
```

หากใช้ Spring Boot 2 ให้เปลี่ยน import จาก `jakarta.servlet.http.HttpServletRequest` เป็น `javax.servlet.http.HttpServletRequest`

### Checklist สำหรับการตรวจสอบ

| ตรวจสอบ | เหตุผล |
| --- | --- |
| อ่าน raw body ก่อน JSON parsing | whitespace และลำดับ key มีผลต่อ HMAC digest |
| เปรียบเทียบ signature ด้วย timing-safe comparison | ลดความเสี่ยงที่ข้อมูล signature จะรั่วผ่าน timing difference |
| ตอบกลับ `401` เมื่อ signature ไม่ถูกต้อง | ป้องกันการประมวลผล webhook ปลอม |
| ตอบกลับ `2xx` หลังจากประมวลผลหรือบันทึกข้อมูลสำเร็จแล้วเท่านั้น | NipaMail จะถือว่า `2xx` คือได้รับแล้วและจะไม่ retry |
