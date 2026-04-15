# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Hoàng Bá Minh Quang  
**Vai trò:** Monitoring / Docs Owner, hỗ trợ Cleaning test data  
**Ngày nộp:** 15/04/2026  
**Độ dài yêu cầu:** **400–650 từ**

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong Lab Day 10, tôi phụ trách phần monitoring và tài liệu vận hành, đồng thời hỗ trợ nhóm ở nhánh tạo dữ liệu bẩn để kiểm thử pipeline. Các file tôi làm chính là `reports/group_report.md`, `docs/quality_report.md`, `docs/runbook.md` và cập nhật `data/raw/policy_export_dirty_v2.csv` để bổ sung failure cases có chủ đích. Cụ thể, tôi thêm các tình huống lỗi như stale refund 14 ngày (chunk_id=3), ngày sai format `01/02/2026` (chunk_id=10), text quá ngắn `Hi` (chunk_id=13), và email nội bộ cần masking (chunk_id=12). Những case này giúp đội cleaning/expectation đo được tác động thật thay vì chỉ thêm rule “trang trí”.

**Bằng chứng:**

- Dòng CSV thật trong `policy_export_dirty_v2.csv`: `13,sla_p1_2026,Hi,2026-02-01,2026-04-10T08:00:00`.
- Dòng stale case: `3,policy_refund_v4,...14 ngày...lỗi migration...`.
- Tài liệu metric_impact ở `reports/group_report.md` mục 2a.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định kỹ thuật quan trọng của tôi là tách rõ hai mode đánh giá chất lượng: mode vận hành thật và mode diễn tập sự cố. Ở mode vận hành thật, expectation phải `halt` khi gặp lỗi policy nghiêm trọng (ví dụ refund khác 7 ngày) để chặn dữ liệu bẩn trước khi embed. Ở mode diễn tập, nhóm cần quan sát toàn bộ chuỗi lỗi nên cho phép chạy `--skip-validate` có kiểm soát. Vì vậy tôi thống nhất ghi nhận rõ trạng thái này trong manifest để không nhầm với run chuẩn. Bằng chứng là `manifest_inject-bad.json` có `no_refund_fix=true`, `skipped_validate=true`, còn `manifest_sprint2_update_rule_and_expect.json` là `false/false`. Cách làm này giúp observability minh bạch: nhìn manifest là biết run nào dùng để chứng minh incident, run nào đủ điều kiện publish index.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Anomaly tôi theo dõi và xử lý cùng nhóm là “trả lời đúng bề mặt nhưng context vẫn bẩn” ở câu hoàn tiền. Triệu chứng: khi chạy run `inject-bad`, file eval cho thấy top1 preview chứa “14 ngày làm việc”, trong khi câu đúng theo policy hiện hành là 7 ngày. Check phát hiện chính là cột `hits_forbidden`: dù `contains_expected=yes`, hệ thống vẫn báo `hits_forbidden=yes`, nghĩa là trong top-k còn chứa chunk cấm/stale. Fix được áp dụng theo hai lớp: (1) cleaning rule xử lý migration/stale refund và đưa bản ghi lỗi vào quarantine; (2) expectation `refund_must_be_7_days` đặt mức halt cho run chuẩn. Sau fix và chạy lại run `sprint2_update_rule_and_expect`, cùng câu hỏi trả về “7 ngày”, `hits_forbidden=no`, xác nhận dữ liệu retrieval đã sạch hơn.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**run_id trước fix:** `inject-bad` (eval file: `artifacts/eval/after.csv`)  
`q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 14 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,yes,,3`

**run_id sau fix:** `sprint2_update_rule_and_expect` (eval file: `artifacts/eval/before.csv`)  
`q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,no,,3`

Ngoài ra, manifest run chuẩn ghi `quarantine_records=5`, phản ánh các failure cases bổ sung đã được hệ thống bắt và cách ly đúng như thiết kế.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ bổ sung một file tổng hợp `before_after_eval.csv` theo đúng tên rubric và tự động ghép từ `before.csv` + `after.csv` sau mỗi run. Việc này giúp truy vết nhanh theo `run_id`, giảm thao tác tay khi viết quality report, và tránh sai sót khi đối chiếu evidence giữa manifest, eval và báo cáo cá nhân.

---
