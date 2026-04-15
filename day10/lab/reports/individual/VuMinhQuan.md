# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Vu minh quan
**Vai trò:** Ingestion
**Ngày nộp:** 15/4/2026
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**: etl_pipeline.py

Trong Sprint 1, tôi đảm nhận vai trò Ingestion Owner, chịu trách nhiệm chính cho việc đọc dữ liệu đầu vào và ghi log pipeline. Tôi làm việc chủ yếu trên file etl_pipeline.py, nơi triển khai luồng ingest từ file CSV raw (policy_export_dirty.csv) vào hệ thống.

Cụ thể, tôi thực hiện:

Đọc dữ liệu bằng hàm load_raw_csv()
Ghi log các chỉ số quan trọng như:
run_id
raw_records
cleaned_records
quarantine_records
Tạo thư mục artifacts (logs, manifests, quarantine)

Tôi phối hợp với thành viên phụ trách cleaning để đảm bảo dữ liệu sau ingest có thể chuyển sang bước xử lý tiếp theo mà không lỗi format.
**Bằng chứng (commit / comment trong code):**
chunk_id,doc_id,chunk_text,effective_date,exported_at,reason,effective_date_normalized
2,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,2026-02-01,2026-04-10T08:00:00,duplicate_chunk_text,
5,policy_refund_v4,,,2026-04-10T08:00:00,missing_effective_date,
7,hr_leave_policy,Nhân viên dưới 3 năm kinh nghiệm được 10 ngày phép năm (bản HR 2025).,2025-01-01,2026-04-10T08:00:00,stale_hr_policy_effective_date,2025-01-01
9,legacy_catalog_xyz_zzz,Chunk nội dung đủ dài để vượt ngưỡng expectation độ dài tối thiểu trong cleaned export.,2026-02-01,2026-04-10T08:00:00,unknown_doc_id,


---

## 2. Một quyết định kỹ thuật (100–150 từ)

> VD: chọn halt vs warn, chiến lược idempotency, cách đo freshness, format quarantine.
Một quyết định quan trọng trong Sprint 1 là sử dụng run_id theo timestamp UTC thay vì đặt thủ công.

Lý do:

Đảm bảo mỗi lần chạy pipeline là duy nhất
Dễ truy vết log và manifest
Hỗ trợ debugging khi có nhiều lần chạy
Quyết định này giúp:

Đồng bộ giữa log file và manifest
Dễ kiểm tra freshness ở Sprint 4

Ngoài ra, tôi chọn ghi log ra file (artifacts/logs) thay vì chỉ print, giúp nhóm có thể:

So sánh giữa các lần chạy
Làm bằng chứng cho report
_________________

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

> Mô tả triệu chứng → metric/check nào phát hiện → fix.
File raw không tồn tại hoặc sai path

Triệu chứng:

Pipeline dừng đột ngột
Log báo FileNotFoundError
Không có dữ liệu nào được xử lý

Metric/check phát hiện:

load_raw_csv() raise FileNotFoundError

Fix:

Thêm try-except block
Log lỗi và exit gracefully
Không làm hỏng pipeline
_________________

---

## 4. Bằng chứng trước / sau (80–120 từ)

> Dán ngắn 2 dòng từ `before_after_eval.csv` hoặc tương đương; ghi rõ `run_id`.
Trước khi fix: ERROR: raw file not found
Sau khi fix:
run_id=sprint1
raw_records=120
cleaned_records=95
quarantine_records=25
cleaned_csv=artifacts/cleaned/cleaned_sprint1.csv
_________________

---

## 5. Cải tiến tiếp theo (40–80 từ)

> Nếu có thêm 2 giờ — một việc cụ thể (không chung chung).
Nếu có thêm 2 giờ, tôi sẽ:

Thêm validate path đầu vào (check tồn tại + format CSV)
Log chi tiết hơn (ví dụ: số dòng lỗi theo từng loại)
Tích hợp cảnh báo khi raw_records = 0

Điều này giúp pipeline robust hơn và giảm lỗi khi deploy thực tế.
_________________
