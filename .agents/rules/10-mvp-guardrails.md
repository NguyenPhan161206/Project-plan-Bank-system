---
trigger: always_on
description: Làn ranh MVP bất biến — không AI, không event bus, không tích hợp thật, không feature ngoài backlog.
---

# Làn ranh MVP (bất biến)

Mọi đề xuất thay đổi phải nằm trong `docs/00-mvp-backlog.md`. Không được làm khi chưa có yêu cầu rõ ràng:

- **No AI/ML** — không trích xuất giấy tờ tự động, không scoring ML, không agent, không MCP.
- **No event bus / saga / Kafka / schema registry** — backend trong-process, state máy trong DB.
- **No tích hợp thật** — cổng e-wallet/ngân hàng, SMS OTP đều giả lập.
- **No thêm feature ngoài backlog** — nếu đề xuất nằm ngoài `docs/00-mvp-backlog.md`, phải báo và gợi ý đưa vào v1/v2, không tự ý làm.