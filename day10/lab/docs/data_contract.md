# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn                      | Phương thức ingest     | Failure mode chính                                                                            | Metric / alert                  |
| -------------------------- | ---------------------- | --------------------------------------------------------------------------------------------- | ------------------------------- |
| policy_export_dirty_v2.csv | CSV export từ hệ thống | - Missing effective_date<br>- Duplicate records<br>- Sai format ngày<br>- doc_id không hợp lệ | raw_records, quarantine_records |

---

## 2. Schema cleaned

| Cột            | Kiểu     | Bắt buộc | Ghi chú                                              |
| -------------- | -------- | -------- | ---------------------------------------------------- |
| chunk_id       | string   | Có       | ID duy nhất cho mỗi chunk (doc_id + index hoặc hash) |
| doc_id         | string   | Có       | ID tài liệu (vd: policy_refund_v4)                   |
| chunk_text     | string   | Có       | Nội dung chunk đã clean                              |
| effective_date | date     | Có       | Ngày hiệu lực (ISO format)                           |
| exported_at    | datetime | Có       | Thời điểm export từ hệ nguồn                         |

---

## 3. Quy tắc quarantine vs drop

### 3.1. Danh mục Quarantine (Lưu vào artifacts/quarantine/)
Các bản ghi vi phạm các điều kiện sau sẽ bị tách khỏi luồng xử lý chính nhưng được lưu lại kèm theo cột reason:

Lỗi định danh (Identity):

unknown_doc_id: doc_id không nằm trong danh sách ALLOWED_DOC_IDS (nhằm tránh nạp nhầm tài liệu rác hoặc chưa được phê duyệt).

Lỗi thời gian (Temporal):

missing_effective_date: Trường ngày hiệu lực bị trống.

invalid_effective_date_format: Ngày hiệu lực không đúng định dạng ISO (YYYY-MM-DD) hoặc DD/MM/YYYY.

stale_hr_policy_effective_date: Riêng với hr_leave_policy, các bản ghi có ngày hiệu lực trước 2026-01-01 bị coi là phiên bản cũ, có nguy cơ gây xung đột dữ liệu.

Lỗi nội dung (Content):

missing_chunk_text: Nội dung chunk_text bị trống hoàn toàn.

too_short_chunk_text: Nội dung sau khi làm sạch có độ dài dưới 20 ký tự (không đủ giá trị thông tin để RAG sử dụng).

duplicate_chunk_text: Nội dung bị trùng lặp (chỉ giữ lại bản ghi đầu tiên xuất hiện trong file export).

### 3.2. Quy tắc Biến đổi & Làm sạch (Cleaning Rules)
Trước khi lưu vào schema Cleaned, dữ liệu trải qua các bước chuẩn hóa tự động:

Date Normalization: Chuyển đổi mọi định dạng ngày hợp lệ về chuẩn ISO 8601 (YYYY-MM-DD).

Whitespace Stripping: Loại bỏ khoảng trắng thừa ở đầu/cuối và các khoảng trắng kép giữa văn bản.

BOM Detection: Phát hiện và gắn nhãn [BOM detect] cho các ký tự Byte Order Mark ẩn để tránh lỗi mã hóa.

Refund Policy Fix: Tự động sửa lỗi sai lệch thông tin trong policy_refund_v4. Chuyển đổi cụm từ "14 ngày làm việc" thành "7 ngày làm việc" và đánh dấu hậu tố [cleaned: stale_refund_window].

PII Masking: Tự động che giấu email nội bộ công ty (có đuôi @company.internal) bằng nhãn [REDACTED_EMAIL] để đảm bảo an toàn dữ liệu.

### 3.3. Quy trình xử lý (Workflow)
Phân loại: Mọi bản ghi lỗi sẽ được đẩy vào file CSV riêng tại artifacts/quarantine/.

Hành động: Data Engineer/Owner kiểm tra định kỳ file quarantine:

Nếu do format: Sửa file nguồn và re-ingest.

Nếu do logic (stale version): Xác nhận xóa bỏ hoặc cập nhật data contract nếu cần.

Hậu kiểm: Các bản ghi sau khi Cleaned phải đảm bảo tính duy nhất thông qua chunk_id (được tạo bằng mã băm SHA-256 từ doc_id + text + seq).
---

## 4. Phiên bản & canonical

* **Refund policy (source of truth)**:

  * File: `data/docs/policy_refund_v4.txt`
  * Version đúng: refund window = **7 ngày**

* **HR Leave policy**:

  * File: `data/docs/hr_leave_policy.txt`
  * Version hợp lệ từ: 2026-01-01

* **Nguyên tắc**:

  * Không cho phép tồn tại nhiều version conflict trong vector DB
  * Version mới sẽ override version cũ (theo effective_date)

---