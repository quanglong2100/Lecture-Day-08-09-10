# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Anh Tài  
**Vai trò:** Embed Owner  
**Ngày nộp:** 15/04/2025  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- etl_pipeline.py: Chịu trách nhiệm thiết lập cơ chế Idempotency (chống trùng lặp) và Pruning (dọn dẹp vector lạc hậu) tại ChromaDB
- docs/: Biên soạn tài liệu pipeline_architecture.md (Kiến trúc hệ thống) và runbook.md (Quy trình xử lý sự cố).

**Kết nối với thành viên khác:**

- transform/cleaning_rules.py: Tham gia cài đặt các luật lọc dữ liệu (ngưỡng độ dài, lỗi migration) và biến đổi dữ liệu (PII Masking, chuẩn hóa khoảng trắng).

---

**Bằng chứng (commit / comment trong code):**

- **commit:** add runbook and architecture - 62d5aaa689dbcce0de212535ae7bf7bcd9369dd5

---

---

## 2. Một quyết định kỹ thuật (100–150 từ)

> VD: chọn halt vs warn, chiến lược idempotency, cách đo freshness, format quarantine.

- Tại pha Transform, thay vì tạo các biến tạm, tôi đề xuất mọi thao tác làm sạch (xóa ký tự tàng hình BOM \ufeff, chuẩn hóa khoảng trắng, sửa lỗi 14 ngày và ẩn PII email) đều phải tác động cộng dồn lên một biến duy nhất là fixed_text. Cấu trúc này giúp đảm bảo dữ liệu trước khi được đưa vào ChromaDB được làm sạch và đồng nhất.

---

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

> Mô tả triệu chứng → metric/check nào phát hiện → fix.

- Trong Sprint 3, tôi xử lý thành công lỗi "Ô nhiễm ngữ cảnh" (Context Pollution).
- Ở kịch bản Inject Bad, mặc dù Agent trả lời đúng 7 ngày ở top1_preview, nhưng script đánh giá báo hits_forbidden = yes. Nguyên nhân là bản ghi chứa rác "14 ngày do lỗi migration" lọt vào top 3, gây nguy cơ ảo giác (Hallucination) cho LLM
- Cách fix: Kích hoạt luật migration_error_detected để ném bản ghi này vào Quarantine. Đồng thời, cấu hình Expectation refund_must_be_7_days với mức severity="halt"

---

---

## 4. Bằng chứng trước / sau (80–120 từ)

> Dán ngắn 2 dòng từ `before_after_eval.csv` hoặc tương đương; ghi rõ `run_id`.

- Trước fix (inject-bad): question_id,question,top1_doc_id,top1_preview,contains_expected,hits_forbidden,top1_doc_expected,top_k_used
  q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 14 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,yes,,3

- Sau fix (clean-fix): question_id,question,top1_doc_id,top1_preview,contains_expected,hits_forbidden,top1_doc_expected,top_k_used
  q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,no,,3

---

---

## 5. Cải tiến tiếp theo (40–80 từ)

> Nếu có thêm 2 giờ — một việc cụ thể (không chung chung).

> Nếu có thêm 2 giờ, thay vì viết logic kiểm tra bằng Python thuần trong expectations.py, tôi sẽ tích hợp thư viện Pydantic để thiết lập Data Schema chặt chẽ hơn. Việc này giúp tự động hóa khâu bắt lỗi sai kiểu dữ liệu ngay trước khi Embed.

---
