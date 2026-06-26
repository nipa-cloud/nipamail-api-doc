# ตรวจสอบเครดิต

## ตรวจสอบยอดเครดิตของผู้เช่า
### คำขอ
```http
GET /v1/credits
```
#### เฮดเดอร์
| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `Authorization` | ใช่ | ใช้ `Bearer <TOKEN>` |

### การตอบกลับ

**200 – OK**

ส่งกลับยอดเครดิตที่พร้อมใช้งานและเครดิตที่กันไว้ของ tenant

```json
{
    "available": <number>,
    "reserved": <number>
}

```
| ฟิลด์ | ชนิด | คำอธิบาย |
| --- | --- | --- |
| `available` | number | เครดิตที่พร้อมใช้ได้ทันที |
| `reserved` | number | เครดิตที่กันไว้สำหรับธุรกรรมที่รอดำเนินการ |


**401 – Unauthorized**

โทเค็นหมดอายุ หรือถูกจำกัด IP ที่สามารถขอส่งข้อความได้

```json
{
    "type": "UnauthorizedError",
    "message": "Invalid credentials or restricted by CIDR."
}
```

**406 – Not Acceptable**

เมื่อ tenant ที่ยืนยันตัวตนถูกเรียกเก็บเงินผ่าน parent tenant จะไม่อนุญาตให้ตรวจสอบเครดิตโดยตรง

```json
{
    "type": "GetAvailableCreditNotAllowedError",
    "message": "Get available credit is not allowed."
}
```
