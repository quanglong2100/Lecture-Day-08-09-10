# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đỗ Lê Thành Nhân  
**Vai trò:**  Cleaning / Embed 
**Ngày nộp:** 15-04-2026 
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- [`transform/cleaning_rules.py`](../../../transform/cleaning_rules.py): Xây dựng và mở rộng quy tắc làm sạch (cleaning rules) bao gồm: loại bỏ doc_id không trong danh sách cho phép, chuẩn hoá ngày hiệu lực (ISO format), phát hiện chính sách HR cũ (effective_date < 2026-01-01), loại trùng chunk_text, và sửa lỗi stale refund window (14 ngày → 7 ngày).
- [`quality/expectations.py`](../../../quality/expectations.py): Định nghĩa 4 expectation gồm kiểm tra số dòng sau cleaning, kiểm tra trường doc_id không rỗng, đảm bảo không còn "14 ngày làm việc" _sai_, và xác thực độ dài minimum của chunk_text (≥8 ký tự).

**Kết nối với thành viên khác:**

- Kết quả cleaned CSV từ tôi được team Embed sử dụng để tạo embeddings và đẩy vào Chroma (collection `day10_kb`).
- Quarantine rows được lưu vào `artifacts/quarantine/` — Monitoring/Docs owner kiểm tra tần suất từng failure mode.

**Bằng chứng (commit / comment trong code):**

- Rule thêm mới trong `cleaning_rules.py` dòng 36–78: xử lý **stale refund window** và **unknown_doc_id**.
- Expectation halt trong `expectations.py` dòng 19–47 (E1, E2, E3 = halt; E4 = warn).

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Khi phát hiện policy_refund_v4 hàng loạt chứa "14 ngày làm việc" (bản cũ sync từ refund-v3), tôi quyết định **bật flag `apply_refund_window_fix=True`** thay vì quarantine hết. Lý do: nếu quarantine sẽ mất 2–3 chunk quan trọng; nhưng lưu sửa đổi lại thành "7 ngày" sẽ:
1) Giữ ngữ cảnh tài liệu (chunk_text không bị cắt),
2) Khớp với chính sách hiện hành 2026,
3) Hỗ trợ retrieval tốt hơn cho agent multi-hop (Day 09).

Expectation `refund_no_stale_14d_window` set severity = **halt** để tránh chunk lạc lõng chứa 14 ngày qua phục vụ embedding, gây nhiễu ranking.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:** Khi chạy pipeline lần đầu, retrieval test câu "Khách hàng có bao nhiêu ngày để hoàn tiền?" trả lại top-1 = "14 ngày làm việc" (_stale_), fails expectation + hits_forbidden = yes (bắt được lỗi).

**Phát hiện:** Metric trong `before.csv` (run baseline) hiển thị `hits_forbidden=yes` ✗. Kiểm tra `artifacts/quarantine/quarantine_*` thấy chunk_id=2 (`policy_refund_v4`) chứa "14 ngày" nhưng chưa bị quarantine — nghĩa là flag fix chưa kích hoạt.

**Fix:** 
- Cập nhật `etl_pipeline.py` gọi `clean_rows(..., apply_refund_window_fix=True)`.
- Thêm rule quarantine "duplicate_chunk_text" + "stale_refund_window" vào `cleaning_rules.py`.
- Re-run pipeline → `after.csv` cho thấy `hits_forbidden=no` ✓, top-1_preview = "7 ngày" ✓.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**Before** (`before.csv` — baseline):
```
q_refund_window | policy_refund_v4 | "Yêu cầu...14 ngày làm việc..." | yes | yes ✗
```

**After** (`after.csv` — sprint2):
```
q_refund_window | policy_refund_v4 | "Yêu cầu...7 ngày làm việc..." | yes | no ✓
```

**Run ID:** `sprint2_update_rule_and_expect` (manifest: `artifacts/manifests/manifest_sprint2_update_rule_and_expect.json`)
- Raw records: 14 → Cleaned: 9, Quarantine: 5
- Quarantine_sprint2 CSV dòng 2: chunk_id=2 (lỗi `duplicate_chunk_text` từ refund), dòng 7: stale_refund_window fix applied ✓.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ: thêm expectation **"alert_on_chunk_similarity_ratio"** — kiểm tra xem cleaned output có > 20% duplicate semantic similarity không (cần thêm embedding similarity check). Hiện tại chỉ check text exact match; nếu có 2 chunk paraphrase nhau (e.g., "7 ngày làm việc" vs "7 ngày buôn bán") sẽ trượt qua. Điều này sẽ cải thiện chất lượng embedding đầu vào cho Chroma.

---
