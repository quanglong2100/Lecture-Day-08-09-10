# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

> User / agent thấy gì? (VD: trả lời “14 ngày” thay vì 7 ngày)

- **Context Pollution (Ảo giác ngữ cảnh):** RAG Agent trả lời khách hàng sai về chính sách hoàn tiền (Ví dụ: "Bạn được hoàn tiền trong 7 ngày hoặc 14 ngày").
- **Lộ lọt thông tin cá nhân (PII Leak):** Agent hiển thị trực tiếp email nội bộ của công ty (`@company.internal`) ra màn hình cho người dùng ngoài xem.

---

## Detection

> Metric nào báo? (freshness, expectation fail, eval `hits_forbidden`)

- **Kiểm tra tự động (Eval):** Lệnh `eval_retrieval.py` trả về `hits_forbidden = yes` (chứng tỏ dữ liệu chứa 14 ngày đã lọt vào top-k).
- **Expectation Halt:** Pipeline tự động dừng (Halt) kèm theo log:
  - `refund_must_be_7_days: FAIL` (Thiếu chính sách chuẩn).
  - `no_duplicate_chunk_text: FAIL` (Phát hiện trùng lặp dữ liệu quá mức).

---

## Diagnosis

| Bước | Việc làm                              | Kết quả mong đợi                                                                                                                                                                                                                                        |
| ---- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Kiểm tra `artifacts/manifests/*.json` | Xác nhận cờ `no_refund_fix` và `skipped_validate` phải bằng **false** (đảm bảo bộ lọc đang bật). Kiểm tra `quarantine_records` > 0 (ví dụ: bằng 5) để chắc chắn file rác đã bị lọc. Kiểm tra `latest_exported_at` để đánh giá nguyên nhân lỗi Freshness |
| 2    | Mở `artifacts/quarantine/*.csv`       | Phải tìm thấy đúng số lượng bản ghi tương ứng với `quarantine_records` trong manifest (ví dụ: 5 bản ghi). Mỗi bản ghi phải có cột `reason` ghi rõ lý do từ chối (vd: `migration_error_detected`, `text_too_short_for_rag`).                             |
| 3    | Chạy `python eval_retrieval.py`       | Kiểm tra file CSV. Đánh giá độ nghiêm trọng của ô nhiễm qua cột `hits_forbidden`.                                                                                                                                                                       |

---

## Mitigation

> Rerun pipeline, rollback embed, tạm banner “data stale”, …

- **Rollback/Clean DB:** Xóa hoặc dọn dẹp (Prune) collection `day10_kb` trong ChromaDB để loại bỏ các vector bị nhiễm mã độc.
- **Rerun Pipeline chuẩn:** Chạy `python etl_pipeline.py run --run-id emergency-fix` (đảm bảo không sử dụng cờ `--skip-validate`).
- Nếu vấn đề PII bị lộ, kiểm tra lại Regular Expression trong `cleaning_rules.py` xem đã bọc (mask) đủ các pattern email chưa.

---

## Prevention

> Thêm expectation, alert, owner — nối sang Day 11 nếu có guardrail.

- Không bao giờ vô hiệu hóa Expectation `refund_must_be_7_days` và `no_migration_errors` trên môi trường Production.
- Theo dõi chặt chẽ file Manifest để báo cáo cho đội dữ liệu nguồn (Upstream) ngay khi `freshness_check` bị vượt ngưỡng SLA 24h.
