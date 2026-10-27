**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย | ใช้ |
| ทางเลือกที่พิจารณา (ระบุ) | เป็นรูปแบบเชิงสถาปัตยกรรมทั่วไป (Generic Patterns): 1. Synchronous Direct RPC/HTTP Orchestration, 2. Asynchronous Publish-Subscribe Event Broker (ไม่ระบุเทคโนโลยี), 3. Asynchronous Centralized Task Queue / Job Worker | เป็นเทคโนโลยีและกลไกที่เจาะจงกับระบบจริง: 1. Direct synchronous / parallel gRPC calls from checkoutservice (อิงโค้ด gRPC เดิม), 2. Asynchronous event-driven publish-subscribe with RabbitMQ (อิงทักษะทีมและโจทย์ K8s), 3. Distributed event streaming with Apache Kafka (นำมาเปรียบเทียบข้อดีข้อเสียกับขนาดทีม) |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | ระบุเพียงหลักการกว้างๆ ในระดับ Mitigation: แนะนำให้ทำ idempotent consumer โดยใช้ order_id แต่ไม่มีรายละเอียดระดับ implementation/database, แนะนำ rate-limited consumers ร่วมกับ exponential backoff และ Dead-Letter Queue (DLQ) แต่ไม่ทราบตัวเลขขีดจำกัดจริง | จัดการอย่างเป็นรูปธรรมและคำนวณเชิงปริมาณ: แต้มสะสม: ออกแบบให้ loyaltyservice เก็บ order_id ลง database table ด้วย unique constraint เพื่อการันตี exactly-once ภายใต้กรอบเวลา 1 นาที, Rate limit: กำหนด QoS prefetch limit ของคิวให้ดึงไม่เกิน 50 emails/s, คำนวณ queue backlog ช่วงพีก (เข้า 200/s ดึงออก 50/s เกิดค้าง 150/s รวม 120,000 ข้อความใน 10 นาที ใช้เวลา drain 40 นาที) |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | มีข้อสมมติฐานในเชิง structure เพิ่มเติม | ไม่มี |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | ระบุเทคโนโลยีฐานข้อมูลที่ชัดเจนสำหรับ loyaltyservice แทนการทิ้งไว้เป็น Open Question | ร่าง 2b ทิ้งคำถามเรื่อง datastore ของ loyaltyservice ไว้ใน Open Questions ทั้งที่ระบบจำเป็นต้องใช้ Transactional Database (เช่น PostgreSQL หรือ Cloud Spanner) เพื่อทำ ACID transaction และ enforce order_id UNIQUE constraint สำหรับการการันตี Exactly-once loyalty points |
