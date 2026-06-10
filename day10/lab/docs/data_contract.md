# Data contract — Lab Day 10

> Nguồn tham chiếu: `contracts/data_contract.yaml` — file này mở rộng và đồng bộ với YAML đó.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn (`doc_id`) | Phương thức ingest | Failure mode chính | Metric / alert |
|-----------------|-------------------|-------------------|----------------|
| `policy_refund_v4` | CSV export từ Policy Management System | Chunk chứa cửa sổ "14 ngày" (version cũ chưa update) | Expectation E3: `refund_no_stale_14d_window` (halt) |
| `sla_p1_2026` | CSV export từ ITSM ticketing | Lẫn chunk P2 vào doc P1; chunk dùng "escalate" không có prefix | Rule 9 quarantine P2 chunks; Rule 11 enrich escalation prefix |
| `it_helpdesk_faq` | CSV export từ Confluence FAQ | Chunk "FAQ bổ sung" không rõ nguồn lọt vào | Rule 10 quarantine `faq bổ sung` prefix |
| `hr_leave_policy` | CSV export từ HRIS | Version cũ "bản hr 2025" với 10 ngày phép còn tồn tại | E6: `hr_leave_no_stale_10d_annual` (halt) + user rule lọc "bản hr 2025" text |
| `access_control_sop` | CSV export từ Security GRC system | Không có trong `ALLOWED_DOC_IDS` baseline → bị drop toàn bộ | Sprint 1 fix: thêm vào `ALLOWED_DOC_IDS` + `canonical_sources` |

**Dữ liệu bị lọc (quarantine):** Tổng 213/247 record (86%) — bao gồm: doc_id không nằm trong allowlist, ngày không ISO, duplicate, empty text, chunk có prefix không rõ ràng ("Nội dung không rõ ràng:", "!!!"), stale HR, stale refund.

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Constraint | Ghi chú |
|-----|------|----------|------------|---------|
| `chunk_id` | string | Có | Unique, stable | Dùng làm ChromaDB document ID cho upsert |
| `doc_id` | string | Có | Phải nằm trong `ALLOWED_DOC_IDS` | Khóa logic tài liệu nguồn |
| `chunk_text` | string | Có | `len ≥ 8` | Text sau khi clean (refund window đã fix) |
| `effective_date` | date | Có | ISO format `YYYY-MM-DD` | Ngày hiệu lực policy; HR filter dùng `≥ 2026-01-01` |
| `exported_at` | datetime | Có | ISO datetime | Dùng cho `freshness_check`; SLA = 24h |

**Ví dụ record cleaned hợp lệ:**
```json
{
  "chunk_id": "policy_refund_v4_001",
  "doc_id": "policy_refund_v4",
  "chunk_text": "Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ xác nhận đơn hàng.",
  "effective_date": "2026-01-01",
  "exported_at": "2026-04-11T00:00:00"
}
```

---

## 3. Quy tắc quarantine vs drop

**Quarantine** (ghi vào `artifacts/quarantine/quarantine_<run_id>.csv` + cột `reason`):

| Lý do (`reason`) | Rule | Có thể khôi phục? |
|-----------------|------|-------------------|
| `not_in_allowlist` | Rule 1 | Có — nếu `doc_id` được thêm vào `ALLOWED_DOC_IDS` |
| `empty_text` | Rule 2 | Không — chunk không có nội dung |
| `stale_date` | Rule 3 (date < 2026-01-01) | Có — nếu policy được re-export với ngày mới |
| `stale_hr_content_2025_version` | Rule 3b (user rule) | Có — nếu HRIS export đúng version 2026 |
| `duplicate_text` | Rule 5 | Không — giữ lại bản đầu tiên |
| `stale_hr_10d_annual_leave` | Expectation-driven quarantine | Có — sau HR policy update |
| `unclear_content_prefix` | Rule 7 | Cần điều tra nguồn gốc chunk |
| `garbled_content_marker` | Rule 8 | Cần điều tra nguồn gốc chunk |
| `p2_content_in_p1_sla_document` | Rule 9 | Có — nên export riêng theo doc_id |
| `supplementary_faq_prefix` | Rule 10 | Cần xác nhận với Confluence owner |

**Approve merge lại:** Chỉ được merge quarantine row trở lại cleaned nếu:
1. Fix source data (re-export từ hệ thống gốc với đúng version/format)
2. Chạy lại toàn bộ pipeline — không patch trực tiếp vào cleaned CSV

---

## 4. Phiên bản & canonical source

| Tài liệu | Canonical file | Version hiện tại | Cửa sổ hợp lệ |
|----------|---------------|-----------------|---------------|
| Refund policy | `data/docs/policy_refund_v4.txt` | v4 | 7 ngày làm việc |
| SLA P1 | `data/docs/sla_p1_2026.txt` | 2026 | phản hồi 15 phút, resolution 4h |
| IT Helpdesk FAQ | `data/docs/it_helpdesk_faq.txt` | current | lockout 5 lần, pass 90 ngày |
| HR Leave Policy | `data/docs/hr_leave_policy.txt` | 2026 | under3=12d, 3-5yr=15d, over5=18d |
| Access Control SOP | `data/docs/access_control_sop.txt` | current | Level 4 cần IT Manager + CISO |

**Rule:** Nếu có conflict giữa CSV export và canonical `.txt` file, canonical `.txt` là source of truth. Pipeline phải reject chunk không match canonical.

**Versioning:** `policy_versioning.hr_leave_min_effective_date = 2026-01-01` — đặt trong `contracts/data_contract.yaml` và đọc vào pipeline qua YAML để tránh hard-code trong code.
