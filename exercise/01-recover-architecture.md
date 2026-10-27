**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| `frontend -> productcatalogservice` | `system/online-boutique/kubernetes-manifests/frontend.yaml` (lines 69–70, env `PRODUCT_CATALOG_SERVICE_ADDR: "productcatalogservice:3550"`); `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml` (lines 80–82, Service port `3550` name `grpc`); `system/online-boutique/protos/demo.proto` (lines 71–75, `service ProductCatalogService`) | ยืนยันแล้ว | ใน `frontend.yaml` มี URL ของ `productcatalogservice` ใน `env` และใน `src/frontend` มีการเรียก gRPC method `ListProducts` / `GetProduct` |
| `frontend -> checkoutservice` | `system/online-boutique/kubernetes-manifests/frontend.yaml` (lines 79–80, env `CHECKOUT_SERVICE_ADDR: "checkoutservice:5050"`); `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (lines 88–90, Service port `5050` name `grpc`); `system/online-boutique/protos/demo.proto` (lines 224–226, `service CheckoutService`) | ยืนยันแล้ว | ใน `frontend.yaml` มี URL ของ `checkoutservice` ใน `env` และใน `src/frontend` มีการเรียก `PlaceOrder` |
| `checkoutservice -> shippingservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (lines 59–60, env `SHIPPING_SERVICE_ADDR: "shippingservice:50051"`); `system/online-boutique/kubernetes-manifests/shippingservice.yaml` (lines 79–81, Service port `50051` name `grpc`); `system/online-boutique/protos/demo.proto` (lines 107–110, `service ShippingService`) | ยืนยันแล้ว | ใน `checkoutservice.yaml` มี URL ของ `shippingservice` ใน `env` และใน `src/checkoutservice` มีัการเรียก `GetQuote` และ `ShipOrder` |
| `checkoutservice -> cartservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (lines 67–68, env `CART_SERVICE_ADDR: "cartservice:7070"`); `system/online-boutique/kubernetes-manifests/cartservice.yaml` (lines 80–82, Service port `7070` name `grpc`); `system/online-boutique/protos/demo.proto` (lines 23–27, `service CartService`) | ยืนยันแล้ว | ใน `checkoutservice.yaml` มี URL ของ `carservice` ใน `env` และใน `src/checkoutservice` มีการเรียก `GetCart` และ `EmptyCart` |
| `recommendationservice -> productcatalogservice` | `system/online-boutique/kubernetes-manifests/recommendationservice.yaml` (lines 60–61, env `PRODUCT_CATALOG_SERVICE_ADDR: "productcatalogservice:3550"`); `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml` (lines 80–82, Service port `3550` name `grpc`); `system/online-boutique/protos/demo.proto` (lines 71–75, `service ProductCatalogService`) | ยืนยันแล้ว | ใน `recommendationservice.yaml` มี URL ของ `productcatalogservice` ใน `env` และใน `src/recommendationservice` มีการเรียก `ListProducts` |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): มี `shoppingassistantservice` ที่มีพูดถึงใน `frontend` แต่ในตารางกับแผนภาพของ agent ไม่ได้เขียนไว้

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
- A ไม่สามารถรู้เรื่อง dependency ได้ เพราะ proto file เป็นการกำหนดแค่ RPC methods
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
- การใส่ `loadgenerator` เป็น microservice ใน system boundary อาจทำให้มีคนเข้าใจผิดว่ามี service ที่คอยส่ง HTTP request ตลอดเวลาใน production
