# AI Receipt Reader

ระบบตรวจสลิปโอนเงินอัตโนมัติ ลูกค้ากรอก Google Form แล้วแนบสลิป จากนั้น AI จะอ่านสลิปและเขียนผลตรวจลง Google Sheet ให้เอง

## ทำงานยังไง

```
Google Form → Google Sheet (แท็บ form1) → n8n → Google Drive (ดาวน์โหลดสลิป) → Gemini (อ่านสลิป) → ตรวจผล → เขียนลงแถวเดิมใน Sheet
```

1. ลูกค้าส่งฟอร์ม (ชื่อผู้โอน, สลิป, จำนวนสินค้า) แล้วแถวใหม่จะเข้า Sheet แท็บ `form1`
2. n8n เช็ก Sheet ทุก 1 นาที ถ้ามีแถวใหม่จะดาวน์โหลดสลิปจาก Drive
3. Gemini อ่านสลิป ได้ยอดเงิน เวลาโอน ผู้รับ ธนาคารผู้รับ และเลขอ้างอิง
4. Code node ตัดสินว่าแถวนั้นผ่าน (TRUE) หรือไม่ผ่าน (FALSE)
5. n8n เขียนผลกลับลงแถวเดิม โดยจับคู่แถวจากลิงก์สลิป

## ใช้อะไร

| ส่วน | เครื่องมือ |
|---|---|
| รับข้อมูล | Google Form |
| เก็บข้อมูลและสรุปผล | Google Sheet |
| เก็บรูปสลิป | Google Drive (โฟลเดอร์ที่ฟอร์มสร้างให้) |
| ระบบอัตโนมัติ | n8n แบบ self-host |
| อ่านสลิป | Google Gemini (โมเดล gemini-2.5-flash) |

## ผลที่ได้

ระบบเขียนลงคอลัมน์ E–I ของแท็บ `form1`

| คอลัมน์ | ความหมาย |
|---|---|
| reference_number | เลขอ้างอิงจากสลิป |
| paid_amount | ยอดเงินที่โอน |
| receiver | ผู้รับเงิน (ชื่อหรือเบอร์) |
| check_valid | TRUE = ผ่าน, FALSE = ไม่ผ่าน |
| fail_reason | เหตุผลที่ไม่ผ่าน |

แท็บ `visualize` สรุปจำนวนออเดอร์ ผ่าน/ไม่ผ่าน/รอตรวจ ยอดเงิน และกราฟรายวัน

> ตอนนี้ระบบตรวจแค่ว่ารูปเป็นสลิปโอนเงินที่สำเร็จหรือไม่ ยังไม่ได้เช็กยอดเงิน ผู้รับ สลิปซ้ำ หรืออายุสลิป แถวที่เป็น FALSE ควรให้คนตรวจซ้ำ

## ติดตั้ง

### 1. คัดลอก Sheet และฟอร์ม
เปิดลิงก์เทมเพลตแล้วกด **Make a copy** จะได้ทั้ง Sheet และฟอร์มเป็นของตัวเอง

`<ลิงก์เทมเพลต>`

### 2. ตั้งค่า Google Cloud
1. สร้าง project ที่ [console.cloud.google.com](https://console.cloud.google.com)
2. เปิด **Google Sheets API** และ **Google Drive API**
3. ตั้ง OAuth consent screen เป็น **External** แล้วเพิ่มอีเมลตัวเองใน **Test users**
4. สร้าง OAuth client ชนิด **Web application**
   Redirect URI: `https://<โดเมน n8n ของคุณ>/rest/oauth2-credential/callback`
5. เก็บ Client ID และ Client secret ไว้

### 3. ขอ Gemini API key
สร้าง key ที่ [aistudio.google.com](https://aistudio.google.com)

### 4. ตั้งค่า n8n
1. **Import** ไฟล์ workflow `.json` (Workflows → Import from file)
2. สร้าง credential 4 ตัว แล้วเลือกใน node ให้ครบ
   - Google Sheets Trigger OAuth2 (node `Google Sheets Trigger`)
   - Google Drive OAuth2 (node `Download file`)
   - Google Gemini (node `Analyze an image`)
   - Google Sheets OAuth2 (node `Update row in sheet`)
3. ใน node `Google Sheets Trigger` และ `Update row in sheet` เปลี่ยนลิงก์ Sheet เป็นของตัวเอง แล้วเลือกแท็บ `form1`

### 5. ตั้งราคา
- แก้ราคาในหัวข้อคำถามอัปโหลดสลิปของฟอร์ม (`Price = 200`)
- แก้ช่อง **B3** ในแท็บ `visualize`
- ถ้าเปลี่ยนหัวข้อคำถาม ต้องเลือกคอลัมน์ใน **Column to Match On** ของ node `Update row in sheet` ใหม่

### 6. ทดสอบและเปิดใช้งาน
1. ส่งฟอร์มทดสอบ 1 ครั้ง แล้วกด **Execute workflow** ใน n8n
2. เช็กว่าแถวใน Sheet มีผลขึ้นในคอลัมน์ E–I
3. กด **Active / Publish** เพื่อให้รันเองตลอด ดูผลแต่ละรอบได้ที่แท็บ **Executions**

## ข้อควรระวัง

- **ห้ามเปลี่ยนชื่อหัวคอลัมน์** ใน `form1` เพราะ workflow จับคู่ข้อมูลด้วยชื่อคอลัมน์
- OAuth ในโหมด Testing จะหมดอายุทุก 7 วัน ถ้าจะใช้ยาวให้ Publish app หรือใช้ Service Account
- Gemini รุ่นฟรีจำกัดจำนวนครั้งต่อนาที ถ้าออเดอร์เข้าพร้อมกันเยอะ ให้เปิด **Retry On Fail** ใน node Gemini
- เครื่องที่รัน n8n ต้องเปิดอยู่ตลอด
- **ห้าม** อัปโหลด API key, Client secret หรือไฟล์ข้อมูลลูกค้าขึ้น GitHub
