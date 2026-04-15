# Quality report — Lab Day 10 (nhóm)

**run_id:**  sprint1 & sprint2_update_rule_and_expect
**Ngày:** 15/4/2026

---

## 1. Tóm tắt số liệu

| Chỉ số             | Trước  | Sau    | Ghi chú                                                   |
| ------------------ | ------ | ------ | --------------------------------------------------------- |
| raw_records        | 14     | 14     | Tổng số bản ghi thô từ nguồn export.                      |
| cleaned_records    | 10     | 9      | Đã lọc thêm 1 dòng rác (migration error) so với baseline. |
| quarantine_records | 4      | 5      | Bản ghi lỗi bị cách ly để bảo vệ chất lượng RAG.          |
| Expectation halt?  | All OK | All OK | Cả 8 bộ quy tắc kiểm định đều đạt trạng thái OK.          |

---

## 2. Before / after retrieval (bắt buộc)

> Đính kèm hoặc dẫn link tới `artifacts/eval/before_after_eval.csv` (hoặc 2 file before/after).

**Câu hỏi then chốt:** refund window (`q_refund_window`)  
**Trước:**  
| question_id     | question                                                                  | top1_doc_id      | top1_preview                                                                   | contains_expected | hits_forbidden | top1_doc_expected | top_k_used |
| --------------- | ------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------ | ----------------- | -------------- | ----------------- | ---------- |
| q_refund_window | Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn? | policy_refund_v4 | Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng. | yes               | no             |                   | 3          |

**Sau:**
| question_id     | question                                                                  | top1_doc_id      | top1_preview                                                                    | contains_expected | hits_forbidden | top1_doc_expected | top_k_used |
| --------------- | ------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------- | ----------------- | -------------- | ----------------- | ---------- |
| q_refund_window | Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn? | policy_refund_v4 | Yêu cầu được gửi trong vòng 14 ngày làm việc kể từ thời điểm xác nhận đơn hàng. | yes               | yes            |                   | 3          |

---

**Merit (khuyến nghị):** versioning HR — `q_leave_version` (`contains_expected`, `hits_forbidden`, cột `top1_doc_expected`)

**Trước:**  
| question_id     | question                                                                                                   | top1_doc_id     | top1_preview                                                                 | contains_expected | hits_forbidden | top1_doc_expected | top_k_used |
| --------------- | ---------------------------------------------------------------------------------------------------------- | --------------- | ---------------------------------------------------------------------------- | ----------------- | -------------- | ----------------- | ---------- |
| q_leave_version | Theo chính sách nghỉ phép hiện hành (2026), nhân viên dưới 3 năm kinh nghiệm được bao nhiêu ngày phép năm? | hr_leave_policy | Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026. | yes               | no             | yes               | 3          |

**Sau:**
| question_id     | question                                                                                                   | top1_doc_id     | top1_preview                                                                 | contains_expected | hits_forbidden | top1_doc_expected | top_k_used |
| --------------- | ---------------------------------------------------------------------------------------------------------- | --------------- | ---------------------------------------------------------------------------- | ----------------- | -------------- | ----------------- | ---------- |
| q_leave_version | Theo chính sách nghỉ phép hiện hành (2026), nhân viên dưới 3 năm kinh nghiệm được bao nhiêu ngày phép năm? | hr_leave_policy | Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026. | yes               | no             | yes               | 3          |

---

## 3. Freshness & monitor

> PASS {"latest_exported_at": "2026-04-15T08:00:00", "age_hours": 2.518, "sla_hours": 24.0}
Nhóm chọn mức 24h vì dữ liệu chính sách (HR, IT, Hoàn tiền) thường được hệ thống nguồn (Upstream) đồng bộ mỗi ngày 1 lần . Mức 24h là "phòng tuyến" vừa đủ để phát hiện ngay sự cố nếu hôm nay hệ thống không xuất file mới, ngăn chặn kịp thời rủi ro AI dùng chính sách cũ để tư vấn sai cho người dùng.
PASS là do mới nhất (được thêm vào) là từ 8h ngày 15/4 chưa quá 24h


--- 

## 4. Corruption inject (Sprint 3)

> Nhóm duplicate, tạo thêm các record ngắn
Việc phát hiện được thực hiện qua các expectation và metric cụ thể: duplicate được phát hiện bằng đếm số chunk_text trùng; stale data kiểm tra bằng rule-based (không chứa “14 ngày”, phải có “7 ngày”); sai format dùng regex ISO cho effective_date; dữ liệu thiếu hoặc rỗng kiểm tra bằng null/empty check; BOM phát hiện qua ký tự \ufeff. Mỗi lỗi đều gắn với ngưỡng cảnh báo hoặc dừng pipeline (halt), giúp đảm bảo dữ liệu đầu vào cho hệ thống RAG luôn sạch, nhất quán và đáng tin cậy.

---

## 5. Hạn chế & việc chưa làm
**1. Quy tắc độ dài**  
Ngưỡng lọc văn bản hiện tại là 20 ký tự. Điều này có thể gây mất các mẩu tin FAQ rất ngắn nhưng có ích.

**2. Validate nâng cao**  
Hiện tại đang sử dụng logic Python thuần. Hướng cải tiến là tích hợp **Pydantic** để kiểm soát schema chặt chẽ hơn hoặc **Great Expectations** cho báo cáo trực quan.

**3. PII Masking**  
Hiện mới chỉ che giấu email nội bộ. Cần mở rộng thêm cho số điện thoại hoặc mã nhân viên trong tương lai.