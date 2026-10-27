**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:**
**agent ที่ใช้:** เช่น GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: …
**โมเดล:** …
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: …

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Antigravitiy Gemini 3.8 Flash | `prompts/1-…` run A | `1-recover-architecture.md`, `demo.proto` | อยู่ | 4 | |
| 2 | 1-B | Antigravitiy Gemini 3.8 Flash | `prompts/1-…` run B | `1-recover-architecture.md`, `demo.proto`, `adservice.yaml`, `cartservice.yaml`, `checkoutservice.yaml`, `currencyservice.yaml`, `emailservice.yaml`, `frontend.yaml`, `kustomization.yaml`, `loadgenerator.yaml`, `paymentservice.yaml`, `productcatalogservice.yaml`, `recommendationservice.md`, `shippingservice.yaml`, `README.md` | อยู่ | 4 | |
| 3 | 2-a | Antigravitiy Gemini 3.8 Flash | `prompts/2-…` run a | `2-write-adr.md` and `template.md` | | | |
| 4 | 2-b | Antigravitiy Gemini 3.8 Flash | `prompts/2-…` run b | `2-write-adr.md`, `template.md`, `scenario.md`, `0001-use-redis-for-cart-storage.md`, `main.go` | | | |
| 5 | 3 | Antigravitiy Gemini 3.8 Flash | `prompts/3-…` | | | | |
| 6 | | | | | | | |
| 7 | | | | | | | |
