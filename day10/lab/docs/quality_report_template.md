# Quality report — Lab Day 10 (nhóm)

**run_id (clean):** `2026-06-10T07-00Z`  
**run_id (inject):** `inject-bad`  
**Ngày:** 2026-06-10

---

## 1. Tóm tắt số liệu

| Chỉ số | Run inject-bad | Run clean (2026-06-10T07-00Z) | Ghi chú |
|--------|---------------|-------------------------------|---------|
| `raw_records` | 247 | 247 | Cùng file CSV gốc |
| `cleaned_records` | 34 | 34 | Inject không thêm rule mới nên cleaned count bằng nhau |
| `quarantine_records` | 213 | 213 | |
| `no_refund_fix` | `true` | `false` | Inject tắt rule fix "14→7 ngày" |
| `skipped_validate` | `true` | `false` | Inject bypass expectation halt |
| Expectation `refund_no_stale_14d_window` | **FAIL** `violations=1` | **OK** `violations=0` | Chunk "14 ngày làm việc" không bị fix → lọt vào store |
| Expectation `no_unclear_content_prefix` | OK | OK | |
| Expectation `all_doc_sources_present` | OK | OK | |
| `freshness_check` | FAIL | FAIL | `latest_exported_at=2026-04-11`, age≈1447h >> SLA 24h |

> **Lưu ý:** `cleaned_records` bằng nhau ở 2 run vì inject dùng cùng rules, chỉ tắt `--refund-fix`. Sự khác biệt nằm ở **nội dung** chunk được embed, không phải số lượng.

---

## 2. Before / after retrieval (bắt buộc)

File so sánh:
- **Inject (before fix):** `artifacts/eval/after_inject_bad.csv`
- **Clean (after fix):** `artifacts/eval/after_fix_eval.csv`

### Câu then chốt: `q_refund_window` — cửa sổ hoàn tiền

| | Inject (`inject-bad`) | Clean (`2026-06-10T07-00Z`) |
|--|----------------------|----------------------------|
| `top1_doc_id` | `policy_refund_v4` | `policy_refund_v4` |
| `top1_preview` | *"Yêu cầu hoàn tiền được chấp nhận trong vòng **14 ngày làm việc**..."* | *"Yêu cầu được gửi trong vòng **7 ngày làm việc**..."* |
| `contains_expected` | `yes` | `yes` |
| **`hits_forbidden`** | **`yes`** ❌ | **`no`** ✅ |
| `top1_doc_expected` | `yes` | `yes` |

**Phân tích:** Sau inject, chunk "14 ngày làm việc" được embed vào store (vì `--no-refund-fix`). Khi query refund window, top-k trả về cả chunk cũ → `hits_forbidden=yes`. Agent dùng context này sẽ trả lời sai chính sách.

### Câu HR versioning: `q_hr_annual_leave_under3`

| | Inject | Clean |
|--|--------|-------|
| `top1_preview` | *"Nhân viên dưới 3 năm... được **12 ngày** phép năm theo chính sách 2026."* | *"Nhân viên dưới 3 năm... được **12 ngày** phép năm theo chính sách 2026."* |
| `contains_expected` | `yes` | `yes` |
| `hits_forbidden` | `no` | `no` |

**Phân tích:** HR versioning không bị ảnh hưởng bởi inject (inject chỉ tắt refund fix). Rule lọc HR stale (Rule 3 + user rule) vẫn hoạt động đúng ở cả 2 run.

### Tổng hợp 21 câu

| Chỉ số | Inject | Clean | Delta |
|--------|--------|-------|-------|
| `contains_expected=yes` | 19/21 | 19/21 | 0 |
| `hits_forbidden=yes` | **1/21** ❌ | **0/21** ✅ | **+1 bị nhiễm** |
| `top1_doc_expected=yes` | 18/21 | 18/21 | 0 |

---

## 3. Freshness & monitor

**Kết quả:** `freshness_check=FAIL`

```json
{
  "latest_exported_at": "2026-04-11T00:00:00",
  "age_hours": 1447.32,
  "sla_hours": 24.0,
  "reason": "freshness_sla_exceeded"
}
```

**Giải thích SLA:** `FRESHNESS_SLA_HOURS=24` trong `.env` — dữ liệu export cũ hơn 24 giờ thì coi là stale. File CSV mẫu có `exported_at` từ 2026-04-11, đã cách ngày chạy pipeline (2026-06-10) khoảng **1447 giờ** → FAIL là đúng và có chủ đích trong lab.

**Ý nghĩa PASS/WARN/FAIL:**
- `PASS`: dữ liệu export trong vòng 24h → tin cậy
- `WARN`: manifest không có `latest_exported_at` → không thể kiểm tra (cần fix ingestion)
- `FAIL`: dữ liệu quá cũ → agent có thể trả lời theo policy lỗi thời; cần trigger re-ingest

**Hành động khi FAIL:** Ghi vào runbook, alert on-call, không dùng collection này cho production cho đến khi có export mới.

---

## 4. Corruption inject (Sprint 3)

**Lệnh:**
```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
```

**Cách inject:** Tắt 2 flags:
1. `--no-refund-fix`: chunk "Yêu cầu hoàn tiền... **14 ngày làm việc**..." không bị replace thành "7 ngày" → embed chunk stale
2. `--skip-validate`: bypass expectation `refund_no_stale_14d_window` (FAIL `violations=1`) để pipeline không bị halt

**Cách phát hiện:**
- **Expectation:** `refund_no_stale_14d_window` báo `FAIL violations=1` trong log → nếu không skip, pipeline dừng ngay tại đây
- **Eval:** `q_refund_window` chuyển từ `hits_forbidden=no` → `hits_forbidden=yes` → bằng chứng định lượng retrieval bị nhiễm
- **Log dòng quyết định:** `WARN: expectation failed but --skip-validate → tiếp tục embed` trong `run_inject-bad.log`

**Kết luận:** Expectation `refund_no_stale_14d_window` (halt) là **lớp bảo vệ đúng chỗ** — nếu không có `--skip-validate`, chunk stale sẽ không bao giờ vào được vector store.

---

## 5. Hạn chế & việc chưa làm

- `freshness_check` chỉ đo 1 boundary (publish time từ `exported_at`). Production cần đo thêm boundary ingest time để phát hiện pipeline lag.
- Eval dùng keyword matching — không phát hiện được câu trả lời sai về mặt ngữ nghĩa nhưng vẫn chứa đúng keyword.
- `all-MiniLM-L6-v2` là model tiếng Anh — retrieval tiếng Việt kém chính xác hơn so với multilingual model (biểu hiện: `q_p1_escalation` chỉ pass ở top-k=5, không pass top-k=3).
- Chưa test idempotency với collection có dữ liệu lớn hơn.
