# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** ___________  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| ___ | Ingestion / Raw Owner | ___ |
| ___ | Cleaning & Quality Owner | ___ |
| ___ | Embed & Idempotency Owner | ___ |
| ___ | Monitoring / Docs Owner | ___ |

**Ngày nộp:** ___________  
**Repo:** ___________  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**

_________________

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**

_________________

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới | Trước fix | Sau fix | Chứng cứ |
|------------------------|-----------|---------|-----------|
| **Rule: `access_control_sop` vào allowlist** | `quarantine_records=203`, `access_control_sop` bị `unknown_doc_id`; E8 FAIL `missing_sources=['access_control_sop']` | `cleaned_records=34` (tăng +5 chunks access_control); E8 OK | `artifacts/logs/run_2026-06-10T07-00Z.log` |
| **Rule 7: `unclear_content_prefix`** | `cleaned_records=44` (8 chunk "Nội dung không rõ ràng:" lọt vào); E7 FAIL `violations=8` | `cleaned_records=36`, E7 OK `violations=0` | So sánh run trước/sau |
| **Rule 8: `garbled_content_marker`** | 2 chunk `!!!` lọt vào cleaned làm nhiễu retrieval | Quarantine 2 chunk, `quarantine_records` tăng | `artifacts/quarantine/` |
| **Rule 9: P2 content trong `sla_p1_2026`** | Chunk "Ticket P2: Escalation sau 90 phút" rank #1 cho query escalation P1; `gq_d10_06` FAIL | Chunk bị loại; escalation "10 phút" lên rank #4; `gq_d10_06` PASS | `artifacts/eval/grading_run.jsonl` |
| **Rule 10: `supplementary_faq_prefix`** | "FAQ bổ sung: 24 giờ" rank #4 cho escalation query → đẩy P1 escalation xuống #8 | Chunk bị loại; IT FAQ noise giảm | `artifacts/quarantine/` |
| **Rule 11: Enrich escalation chunk** | Escalation rank #8 dù relevant; `gq_d10_06` FAIL | Rank lên #4; `gq_d10_06 contains_expected=true` | `artifacts/eval/grading_run.jsonl` |
| **E7: `no_unclear_content_prefix`** (halt) | Trước Rule 7: FAIL `violations=8` | Sau Rule 7: OK `violations=0` | `artifacts/logs/run_2026-06-10T07-00Z.log` |
| **E8: `all_doc_sources_present`** (warn) | Trước thêm allowlist: WARN `missing_sources=['access_control_sop']` | Sau: OK `missing_sources=[]` | `artifacts/logs/run_2026-06-10T07-00Z.log` |

**Rule chính (baseline + mở rộng):**

Baseline (6 rules): allowlist doc_id, normalize effective_date, HR stale by date, empty text, dedup, refund window fix.

Mới thêm (5 rules + 1 user rule):
- **HR stale content**: quarantine `hr_leave_policy` chunk có text chứa `"bản HR 2025"` dù ngày mới (phát hiện version conflict không bắt được bởi date filter).
- **unclear_content_prefix** (Rule 7): quarantine chunk bắt đầu `"Nội dung không rõ ràng:"` — 8 chunk bị loại.
- **garbled_content_marker** (Rule 8): quarantine chunk bắt đầu `"!!!"` — 2 chunk bị loại.
- **P2 cross-contamination** (Rule 9): quarantine `sla_p1_2026` chunk mang nội dung Ticket P2 — fix retrieval ranking cho `gq_d10_06`.
- **supplementary_faq_prefix** (Rule 10): quarantine `"FAQ bổ sung:"` chunk — giảm noise thời gian trong IT FAQ.
- **escalation_enrich** (Rule 11): prefix `"auto-escalation P1 ticket policy:"` vào escalation chunk — P1 escalation rank từ #8 → #4.

**Ví dụ expectation fail và cách xử lý:**

Khi chạy pipeline **trước** khi thêm allowlist `access_control_sop`: expectation **E8 `all_doc_sources_present`** (warn) báo `missing_sources=['access_control_sop']`. Hành động: thêm doc_id vào `ALLOWED_DOC_IDS` trong `transform/cleaning_rules.py`. Sau fix: OK `missing_sources=[]`.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

_________________

**Kết quả định lượng (từ CSV / bảng):**

_________________

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

_________________

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

_________________

---

## 6. Rủi ro còn lại & việc chưa làm

- …
