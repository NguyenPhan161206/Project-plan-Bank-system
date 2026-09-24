---
name: mvp-check
description: Kiểm tra một đề xuất/thay đổi có nằm trong phạm vi MVP của dự án không (đối chiếu docs/00, làn ranh bất biến, quy tắc không hardcode).
---

# /mvp-check

Kiểm tra tính hợp lệ của đề xuất/thay đổi sau so với nguồn sự thật của dự án:

$ARGUMENTS

## Quy trình

1. Đọc `docs/00-mvp-backlog.md` (mục scope IN/OUT + backlog).
2. Đối chiếu đề xuất với từng ràng buộc:
   - **Phạm vi:** nằm trong IN? Không nằm trong OUT?
   - **Feature:** có trong backlog ưu tiên? Trùng/spread sang epic nào?
   - **Làn ranh MVP:** có vi phạm không (AI/ML, event bus/saga/Kafka, tích hợp thật, feature ngoài backlog)?
   - **Không hardcode:** có vi phạm quy tắc mã nguồn trong AGENTS.md không?
3. Trả kết luận rõ ràng:
   - "TRONG MVP ✅" + sprint/epic đề xuất nếu hợp lệ.
   - "TRƯỢT NGOÀI ⚠️" + lý do + gợi ý đưa vào v1/v2 nếu vi phạm.
   - Liệt kê từng ràng buộc đã kiểm và kết quả từng cái.