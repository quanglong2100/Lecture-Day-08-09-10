# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Trần Quang Long
**Vai trò:** Cleaning
**Ngày nộp:** 15/04/2026
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong đợt Lab này, tôi trực tiếp đảm nhiệm việc nâng cấp hệ thống xử lý dữ liệu và kiểm định chất lượng cho Pipeline. Cụ thể, tôi đã bổ sung vào file transform/cleaning_rules.py ba quy tắc làm sạch mới: xử lý ký tự ẩn BOM, lọc bỏ các đoạn văn bản quá ngắn và mã hóa email nội bộ (PII). Song song đó, tôi thiết lập hai chốt chặn chất lượng quan trọng tại quality/expectations.py để kiểm soát dữ liệu trùng lặp và xác thực nội dung chính sách hoàn tiền. Công việc của tôi đóng vai trò là "màng lọc" cuối cùng, đảm bảo dữ liệu khi đi vào ChromaDB không chỉ sạch về mặt định dạng mà còn phải tuyệt đối chính xác về nội dung nghiệp vụ và an toàn về bảo mật.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định kỹ thuật quan trọng nhất của tôi là triển khai Expectation kiểm tra dương tính (Positive Check) mang tên refund_must_be_7_days với mức độ nghiêm trọng là Halt. Khác với các kiểm tra thông thường chỉ tìm lỗi (negative check), quy tắc này yêu cầu tất cả các bản ghi thuộc policy_refund_v4 bắt buộc phải chứa cụm từ "7 ngày làm việc". Tôi quyết định chọn mức Halt vì đây là thông tin cốt lõi của doanh nghiệp; nếu thiếu cụm từ này, Agent có thể đưa ra câu trả lời mơ hồ hoặc thiếu căn cứ pháp lý. Việc ép buộc pipeline phải dừng lại nếu không tìm thấy từ khóa "7 ngày" giúp chúng ta ngăn chặn triệt để tình trạng dữ liệu bị thiếu sót hoặc bị thay đổi cấu trúc nội dung ngoài ý muốn trong quá trình transform, đảm bảo rằng tri thức mà AI cung cấp luôn đạt độ tin cậy cao nhất.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Tôi đã xử lý một anomaly đặc thù liên quan đến ký tự ẩn BOM (\ufeff) thường xuất hiện khi export dữ liệu từ các hệ thống cũ sang định dạng CSV. Ký tự này dù không hiển thị trên màn hình nhưng lại làm sai lệch kết quả tính toán vector similarity, khiến AI khó tìm thấy đoạn văn bản đó. Tôi đã thêm rule text.replace('\ufeff', "[BOM detect]") để chủ động nhận diện và làm sạch. Ngoài ra, tôi phát hiện nhiều chunk văn bản chỉ có vài từ vô nghĩa làm nhiễu hệ thống, nên đã triển khai thêm rule lọc too_short_chunk_text để đẩy các bản ghi dưới 20 ký tự vào khu vực Quarantine. Nhờ việc xử lý này, số lượng quarantine_records trong các lần chạy thử nghiệm đã phản ánh chính xác các "rác" dữ liệu thực tế, giúp dữ liệu sạch trở nên tinh gọn và chất lượng hơn.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Before: q_refund_window	Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?	policy_refund_v4	Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.	yes	no		3
After: q_refund_window	Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?	policy_refund_v4	Yêu cầu được gửi trong vòng 14 ngày làm việc kể từ thời điểm xác nhận đơn hàng.	yes	yes		3

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ làm việc, tôi sẽ nâng cấp quy tắc PII Masking để không chỉ lọc email @company.internal mà còn tự động nhận diện các mẫu số điện thoại và mã số nhân viên bằng Regex. Hiện tại hệ thống mới chỉ xử lý được email đơn giản. Việc mở rộng này sẽ giúp tăng cường khả năng bảo mật dữ liệu, đảm bảo pipeline có thể xử lý an toàn các tài liệu nhạy cảm hơn từ phòng nhân sự hoặc tài chính mà không lo vi phạm các quy định về quyền riêng tư.

---
