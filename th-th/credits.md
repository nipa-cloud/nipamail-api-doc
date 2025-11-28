# ตรวจสอบเครดิต

## ตรวจสอบยอดเครดิตของผู้เช่า
### คำขอ
```http
GET /v1/credits
```
#### เฮดเดอร์
| ฟิลด์ | บังคับ | คำอธิบาย |
| --- | --- | --- |
| `Authorization` | Yes | ใช้ `Bearer <TOKEN>` |

### การตอบกลับ

**200 – OK**

ส่งกลับยอดเครดิตที่ใช้ได้และเครดิตที่กันไว้ของผู้เช่า

```json
{
  "available": <number>,
  "reserved": <number>
}

```
| ฟิลด์ | ชนิด | คำอธิบาย |
| --- | --- | --- |
| `available` | number | เครดิตที่สามารถใช้ได้ทันที |
| `reserved` | number | เครดิตที่กันไว้สำหรับธุรกรรมที่รอดำเนินการ |


**401 – Unauthorized**

ข้อมูลยืนยันตัวตนไม่ถูกต้อง หรือถูกจำกัดด้วย CIDR

```json
{
  "type": "UnauthorizedError",
  "message": "Invalid credentials or restricted by CIDR."
}
```
