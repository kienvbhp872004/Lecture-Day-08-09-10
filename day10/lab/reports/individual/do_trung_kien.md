# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đỗ Trung Kiên  
**Vai trò:** Cleaning & Quality Owner — Transform rules, Expectation suite, Retrieval debug  
**Ngày nộp:** 2026-06-10  
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**

- `transform/cleaning_rules.py` — Phân tích dữ liệu raw (247 records), phát hiện `access_control_sop` bị drop toàn bộ do không có trong `ALLOWED_DOC_IDS`; thêm vào allowlist. Bổ sung 5 rule mới (Rule 7–11) xử lý các loại noise chưa được baseline bắt: chunk prefix không rõ ràng, garbled markers, P2 cross-contamination trong `sla_p1_2026`, supplementary FAQ noise, và prefix enrich cho escalation chunk.
- `quality/expectations.py` — Thêm E7 (`no_unclear_content_prefix`, halt) và E8 (`all_doc_sources_present`, warn) để expectation suite phản ánh đúng reality sau khi allowlist được cập nhật.
- `docs/quality_report_template.md`, `docs/pipeline_architecture.md`, `docs/data_contract.md`, `docs/runbook.md` — Điền toàn bộ số liệu trước/sau từ artifact thực.

**Kết nối với thành viên khác:** Phần embed và idempotency (`etl_pipeline.py::embed_chunks`) chạy downstream từ cleaned rows mà tôi sản xuất. Mỗi thay đổi trong `cleaning_rules.py` ảnh hưởng trực tiếp đến `cleaned_records`, `quarantine_records` và kết quả grading.

**Bằng chứng:** Grading run `artifacts/eval/grading_run.jsonl` — 10/10 câu `contains_expected=true`, `hits_forbidden=false`, `top1_doc_matches=true`.

---

## 2. Một quyết định kỹ thuật

**Quyết định: dùng `halt` cho E7 (`no_unclear_content_prefix`) thay vì `warn`.**

Baseline có 2 mức severity: `halt` (dừng pipeline, không embed) và `warn` (ghi log, tiếp tục). Khi phát hiện 8 chunk có prefix `"Nội dung không rõ ràng:"`, tôi chọn `halt` thay vì `warn` vì lý do sau:

Chunk với prefix này không mang nội dung policy thực — chúng là artifact của quá trình export sai. Nếu embed vào store, chúng sẽ tranh ranking với chunk hợp lệ theo cơ chế cosine similarity của `all-MiniLM-L6-v2`. Thực tế quan sát thấy: trước khi có Rule 7, 8 chunk này chiếm top-k cho một số query do có từ khóa trùng, đẩy chunk chính xác xuống ngoài top-5. Dùng `warn` sẽ để pipeline tiếp tục và silently nhúng noise, khó debug sau. `halt` buộc pipeline dừng, visible trong log, và yêu cầu fix source data trước khi publish — đây là behavior đúng cho production.

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** `gq_d10_06` ("P1 auto escalate sau bao lâu?") liên tục fail `contains_expected=false` dù data cho "10 phút" đã có trong store.

**Phát hiện:** Chạy debug query trực tiếp vào ChromaDB, quan sát top-8 kết quả theo cosine similarity:
- Rank #1: chunk "Ticket P2: Escalation sau 90 phút" (từ `sla_p1_2026`) — P2 content lọt vào doc P1
- Rank #4: chunk "FAQ bổ sung: thời gian 24 giờ" (từ `it_helpdesk_faq`) — supplementary noise
- Rank #8: chunk "10 phút" (P1 escalation thực) — quá thấp để nằm trong `top_k=5`

**Fix (3 bước liên tiếp):**
1. Rule 9 quarantine P2 chunk trong `sla_p1_2026` (`reason=p2_content_in_p1_sla_document`)
2. Rule 10 quarantine "FAQ bổ sung" chunk (`reason=supplementary_faq_prefix`)
3. Rule 11 enrich escalation chunk với prefix `"auto-escalation P1 ticket policy:"` để tăng semantic weight

**Kết quả:** Escalation chunk từ rank #8 lên rank #4, `gq_d10_06` chuyển sang `contains_expected=true`. Xác nhận qua `artifacts/eval/grading_run.jsonl`.

---

## 4. Bằng chứng trước / sau

**run_id inject:** `inject-bad` | **run_id clean:** `2026-06-10T07-00Z`

```
# artifacts/eval/after_inject_bad.csv (inject — dữ liệu bẩn):
q_refund_window, top1_preview="...14 ngày làm việc...", hits_forbidden=yes ❌

# artifacts/eval/after_fix_eval.csv (clean — pipeline fix đầy đủ):
q_refund_window, top1_preview="...7 ngày làm việc...", hits_forbidden=no  ✅
```

Tổng hợp 21 câu: `hits_forbidden` từ **1/21** (inject) → **0/21** (clean). Grading chính thức (`grading_run.jsonl`): **10/10 pass**.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ thay `all-MiniLM-L6-v2` bằng `paraphrase-multilingual-MiniLM-L12-v2` và so sánh top-k ranking cho toàn bộ 21 câu. Hiện tại `q_p1_escalation` chỉ pass khi `top_k=5` — với model multilingual, escalation chunk nhiều khả năng lên top-3 do embedding tiếng Việt chính xác hơn, giảm phụ thuộc vào Rule 11 prefix workaround.
