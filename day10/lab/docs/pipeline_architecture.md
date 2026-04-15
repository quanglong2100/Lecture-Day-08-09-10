# Kiến trúc pipeline — Lab Day 10

**Nhóm:** C401 - A2  
**Cập nhật:** 15/04/2026

---

## 1. Sơ đồ luồng (bắt buộc có 1 diagram: Mermaid / ASCII)

````
raw export (CSV/API/…)  →  clean  →  validate (expectations)  →  embed (Chroma)  →  serving (Day 08/09)
```
graph TD
    A[Raw Export CSV] --> B(Clean & Transform)
    B -->|Lỗi/Rác| C[Quarantine CSV]
    B -->|Dữ liệu Sạch| D{Validate: Expectations}
    D -->|Halt| E[Pipeline Stopped]
    D -->|Pass| F(Embed - ChromaDB)
    F --> G[(Vector Store: day10_kb)]
    G --> H[Serving: RAG Agent Day 09]

    Logs & Monitoring
    D -.-> I[Manifest JSON]
    I -.-> J((Freshness Check))
````

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input                    | Output                          | Owner nhóm         |
| ---------- | ------------------------ | ------------------------------- | ------------------ |
| Ingest     | policy_export_dirty.csv  | Danh sách Dictionary thô        | Ingest Owner       |
| Transform  | Danh sách Dictionary thô | cleaned_rows và quarantine_rows | Transform Owner    |
| Quality    | cleaned_rows             | ExpectationResult, cờ halt      | Quality Owner      |
| Embed      | cleaned_rows hợp lệ      | Vector trong day10_kb ChromaDB  | Embed Owner        |
| Monitor    | manifest\_\*.json        | Log trạng thái PASS/FAIL        | Docs/Monitor Owner |

---

## 3. Idempotency & rerun

> Mô tả: upsert theo `chunk_id` hay strategy khác? Rerun 2 lần có duplicate vector không?

> Cơ chế Idempotent: Hệ thống tạo chunk_id cố định (stable) bằng cách băm (hash SHA-256) sự kết hợp giữa doc_id, chunk_text, và seq. Khi chạy lại, hàm col.upsert của ChromaDB sẽ ghi đè lên ID cũ thay vì tạo mới. Việc Rerun 2 lần liên tiếp sẽ không làm phình to (duplicate) số lượng vector.

> Cơ chế Pruning (Dọn dẹp): Pipeline lấy danh sách ID hiện có trong collection trừ đi danh sách ID mới của lần chạy này (drop = sorted(prev_ids - set(ids))). Các ID lạc hậu (không còn xuất hiện trong bản Clean mới nhất) sẽ bị xóa thông qua lệnh col.delete, đảm bảo không có vector "mồi cũ" bám lại trong kết quả tìm kiếm.

---

## 4. Liên hệ Day 09

> Pipeline này cung cấp / làm mới corpus cho retrieval trong `day09/lab` như thế nào? (cùng `data/docs/` hay export riêng?)

> Pipeline này đóng vai trò là "nhà máy lọc nước" cung cấp nguồn tri thức sạch (Corpus) cho hệ thống RAG xây dựng từ Day 09. Thay vì đọc trực tiếp từ file tĩnh data/docs/, Agent ở Day 09 giờ đây sẽ truy vấn vào collection day10_kb trong ChromaDB — nơi chỉ chứa các quy định đã được chuẩn hóa và loại bỏ hoàn toàn các lỗi phiên bản

---

## 5. Rủi ro đã biết

> Rủi ro False Positive: Các quy tắc làm sạch quá nghiêm ngặt (như độ dài tối thiểu) có thể vô tình loại bỏ các câu FAQ ngắn hợp lệ (vd: FAQ khóa tài khoản 5 lần). Đã khắc phục bằng cách hạ ngưỡng độ dài xuống 40 ký tự.

> Dữ liệu nguồn (Upstream) chậm cập nhật, dẫn đến freshness_check bị cảnh báo SLA vượt quá 24h.
