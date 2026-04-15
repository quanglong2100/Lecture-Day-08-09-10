# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Công Quốc Huy
**Vai trò:** Execution / Logging / Monitoring  
**Ngày nộp:** 15/04/2026  


---

## 1. Tôi phụ trách phần nào? 

**File / module:**

- etl_pipline.py
- .log, .json, .csv

Tôi phụ trách việc **chạy toàn bộ pipeline end-to-end** và  **logging + monitor kết quả**. Cụ thể, tôi đảm bảo mỗi lần chạy pipeline đều có `run_id`, ghi lại đầy đủ thông tin, và xuất log để phục vụ debug cũng như đánh giá chất lượng dữ liệu. Việc này giúp nhóm dễ dàng truy vết lỗi và so sánh các lần chạy khác nhau.

**Kết nối với thành viên khác:**
- Nhận `raw` từ Data Raw 
- Nhận `cleaning_rule và expectations` từ cleaning và quality
- Gửi log cho các bên hoàn thành docs 
_________________

**Bằng chứng (commit / comment trong code):**

> Commit upload log lên github

---

## 2. Một quyết định kỹ thuật 

> Đọc hiểu và chạy lần lượt từ baseline (chưa update rule, expectations) đến cuối cùng.


---

## 3. Một lỗi hoặc anomaly đã xử lý

> Mặc dù chạy --no-refund nhưng kết quả cuối cùng vẫn không có kết quả 14 ngày --> Báo lại cho bên tạo rule mới kiểm tra và sửa lại thành công do có rule khác đè vào đã làm thay đổi record 14 ngày

_________________

## 4. Bằng chứng trước / sau 

| question_id     | question                                                                  | top1_doc_id      | top1_preview                                                                   | contains_expected | hits_forbidden | top1_doc_expected | top_k_used |
| --------------- | ------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------ | ----------------- | -------------- | ----------------- | ---------- |
| q_refund_window | Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn? | policy_refund_v4 | Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng. | yes               | no             |                   | 3          |

_________________

---

## 5. Cải tiến tiếp theo

> Nếu có thêm thời gian, tôi sẽ xây dựng một dashboard theo dõi metric theo thời gian (freshness, volume, error rate) dựa trên run_id. Điều này giúp phát hiện xu hướng lỗi và giám sát pipeline hiệu quả hơn thay vì chỉ kiểm tra từng lần chạy riêng lẻ