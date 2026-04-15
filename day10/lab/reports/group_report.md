# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** A2C401 (cập nhật theo `contracts/data_contract.yaml`)  
**Thành viên:**

| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Nguyễn Anh Tài | Embed & Idempotency Owner | (nhóm điền bổ sung) |
| (nhóm điền bổ sung) | Ingestion / Raw Owner | (nhóm điền bổ sung) |
| (nhóm điền bổ sung) | Cleaning & Quality Owner | (nhóm điền bổ sung) |
| Hoàng Bá Minh Quang | Monitoring / Docs Owner | hbminhquang.19@gmail.com |

**Ngày nộp:** 15/04/2026  
**Repo:** day10/lab

---

## 1. Pipeline tổng quan

Nhóm sử dụng nguồn raw chính là file export CSV `data/raw/policy_export_dirty.csv` (14 bản ghi), mô phỏng đúng các lỗi thường gặp trước khi đẩy vào RAG: trùng nội dung, thiếu/sai định dạng ngày hiệu lực, bản policy HR cũ, doc_id ngoài allowlist, và nhiễu do migration. Luồng end-to-end của nhóm là: ingest CSV -> cleaning + quarantine -> expectation validate -> embed Chroma (upsert theo chunk_id + prune id cũ) -> ghi manifest để theo dõi freshness/lineage. Các run được truy vết bằng `run_id` xuất hiện trong log khi chạy pipeline, đồng thời được lưu lại trong `artifacts/manifests/manifest_<run_id>.json` để phục vụ đối soát.

Run tốt dùng cho báo cáo là `sprint2_update_rule_and_expect` (raw=14, cleaned=9, quarantine=5, validate không skip). Run inject để chứng minh tác động chất lượng là `inject-bad` (bật `--no-refund-fix --skip-validate`) nhằm cố ý cho dữ liệu stale đi vào bước embed và quan sát ảnh hưởng trên retrieval.

**Lệnh chạy một dòng (theo README, nhóm dùng thực tế):**

```bash
python etl_pipeline.py run --run-id sprint2_update_rule_and_expect && python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint2_update_rule_and_expect.json && python eval_retrieval.py --out artifacts/eval/before.csv
```

---

## 2. Cleaning & expectation

Ngoài các baseline rule (allowlist doc_id, chuẩn hóa ngày ISO, quarantine HR stale, dedupe, fix refund window), nhóm mở rộng thêm các xử lý để giảm nhiễu dữ liệu đưa vào vector store. Các rule mới tập trung vào tính sử dụng thật của chunk cho retrieval và an toàn dữ liệu, thay vì chỉ “làm đẹp text”. Đồng thời nhóm bổ sung expectation để pipeline có cơ chế halt rõ ràng khi gặp sai lệch policy quan trọng.

### 2a. Bảng metric_impact

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| `too_short_chunk_text` (rule) | Dữ liệu chứa 1 dòng cực ngắn: `Hi` | Dòng này bị quarantine, đóng góp vào `quarantine_records=5` | `artifacts/quarantine/quarantine_sprint2_update_rule_and_expect.csv` |
| `pii_masking_internal_email` (rule) | Chunk IT chứa email nội bộ thật | Sau clean đổi thành `[REDACTED_EMAIL]` | `artifacts/cleaned/cleaned_sprint2_update_rule_and_expect.csv` |
| `refund 14->7` + marker stale (rule) | Khi inject (`--no-refund-fix`) top1 preview hiện “14 ngày” | Run chuẩn trả về “7 ngày”, không còn stale ở kết quả chính | `artifacts/eval/after.csv` vs `artifacts/eval/before.csv` |
| `no_duplicate_chunk_text` (expectation, halt) | Raw có bản ghi trùng | Sau clean pass (`duplicate_count=0`) | logic trong `quality/expectations.py` + cleaned CSV |
| `refund_must_be_7_days` (expectation, halt) | Kịch bản inject làm expectation fail | `manifest_inject-bad.json` có `skipped_validate=true` (chỉ xảy ra khi có halt + skip) | `artifacts/manifests/manifest_inject-bad.json` |

**Rule chính (baseline + mở rộng):**

- Allowlist `doc_id`, loại `unknown_doc_id`.
- Chuẩn hóa `effective_date` về ISO; quarantine nếu thiếu/sai format.
- Quarantine HR policy cũ (`effective_date < 2026-01-01`).
- Loại duplicate theo normalized `chunk_text`.
- Fix stale refund từ 14 ngày sang 7 ngày và gắn marker theo dõi.
- Rule mới: loại text quá ngắn (`too_short_chunk_text`), masking email nội bộ, xử lý BOM.

**Ví dụ expectation fail và cách xử lý:**

Ở run `inject-bad`, nhóm cố ý tắt refund fix và bỏ validate cứng để mô phỏng incident. Khi đó expectation `refund_must_be_7_days` fail (halt), nhưng pipeline vẫn đi tiếp do có cờ `--skip-validate` phục vụ demo. Sau đó nhóm chạy lại pipeline chuẩn (không skip validate, có refund fix) để trả hệ thống về trạng thái an toàn.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent

**Kịch bản inject:**

Nhóm chạy:

```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/after.csv
```

Mục tiêu là đẩy bản chunk refund còn “14 ngày” vào collection để xem retrieval có bị nhiễm context cũ không. Manifest inject xác nhận đúng kịch bản: `no_refund_fix=true`, `skipped_validate=true`.

**Kết quả định lượng (từ CSV):**

- Ở câu `q_refund_window`, file `after.csv` trả về top1 preview chứa “14 ngày làm việc”, `hits_forbidden=yes`.
- Sau khi chạy pipeline chuẩn và eval lại (`before.csv`), cùng câu hỏi trả về “7 ngày làm việc”, `hits_forbidden=no`.
- Các câu khác (`q_p1_sla`, `q_lockout`, `q_leave_version`) giữ ổn định, cho thấy lỗi chủ yếu nằm ở nhánh refund policy và đã được cô lập tốt sau khi áp dụng đúng rule + expectation.

Điểm quan trọng là nếu chỉ nhìn cột `contains_expected`, nhiều trường hợp vẫn là `yes`, nhưng observability qua `hits_forbidden` mới giúp phát hiện “đúng bề mặt nhưng context vẫn bẩn”. Đây là lý do nhóm giữ bước inject/before-after như một test bắt buộc trước khi publish dữ liệu cho agent.

---

## 4. Freshness & monitoring

Nhóm dùng SLA freshness 24 giờ (khai báo trong data contract). Ý nghĩa vận hành:

- PASS: tuổi dữ liệu <= 24h, an toàn để phục vụ retrieval.
- FAIL: tuổi dữ liệu > 24h, cần cảnh báo vì nguy cơ dùng policy cũ.
- WARN: manifest thiếu timestamp hoặc timestamp không parse được.

Với run tốt `sprint2_update_rule_and_expect`, manifest có `latest_exported_at=2026-04-15T08:00:00` và thời điểm chạy cùng ngày nên ở trạng thái PASS. Kiểm tra freshness cũng là checkpoint cuối trước khi cho phép hệ thống Day 09 dùng bản index mới.

---

## 5. Liên hệ Day 09

Pipeline Day 10 phục vụ trực tiếp cho Day 09 ở tầng knowledge base. Thay vì để multi-agent truy vấn dữ liệu chưa kiểm soát từ raw/doc rời rạc, nhóm publish vào collection Chroma `day10_kb` sau khi đã clean + validate + quarantine. Cách này giữ được chất lượng trả lời ổn định hơn, đặc biệt với các policy dễ thay đổi theo version như HR leave và refund window.

---

## 6. Rủi ro còn lại & việc chưa làm

- Rule độ dài tối thiểu có thể loại nhầm một số FAQ ngắn nhưng hữu ích; cần tinh chỉnh theo từng `doc_id` thay vì một ngưỡng chung.
- PII masking hiện mới bao phủ email nội bộ, chưa mở rộng cho số điện thoại hoặc mã nhân viên.
- Cần bổ sung alert tự động (chat/webhook) khi freshness FAIL, hiện mới dừng ở mức kiểm tra theo lệnh.
- Bảng thành viên trong báo cáo vẫn còn thiếu thông tin email và phân vai chi tiết của toàn nhóm, cần cập nhật trước khi nộp bản cuối.
