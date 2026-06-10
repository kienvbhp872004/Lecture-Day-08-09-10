# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

**S1 — Sai cửa sổ hoàn tiền:** Agent / user nhận câu trả lời *"14 ngày làm việc"* thay vì *"7 ngày làm việc"* khi hỏi về refund policy.

**S2 — Data stale:** Agent trả lời theo chính sách cũ (HR 10 ngày phép, escalation P1 không có prefix) — thường xảy ra sau khi pipeline chạy với `--no-refund-fix` hoặc sau khi nguồn export không được refresh.

**S3 — Freshness FAIL:** Monitor báo `freshness_check=FAIL` — dữ liệu trong store cũ hơn SLA 24h.

---

## Detection

| Metric | Tool | Ngưỡng cảnh báo |
|--------|------|----------------|
| `hits_forbidden=yes` trong eval CSV | `eval_retrieval.py` | > 0 câu hỏi → nghiêm trọng |
| `expectation[refund_no_stale_14d_window] FAIL` | `artifacts/logs/run_*.log` | violations ≥ 1 → halt (hoặc bypass nếu `--skip-validate`) |
| `freshness_check=FAIL` | `monitoring/freshness_check.py` | `age_hours > sla_hours` |
| `embed_prune_removed` bất thường | Log | Số lớn bất thường → nhiều chunk bị xóa |
| Grading `contains_expected=no` | `grading_run.jsonl` | Bất kỳ câu nào fail |

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | `cat artifacts/manifests/manifest_<run_id>.json` | Xác nhận `no_refund_fix`, `skipped_validate` có là `true` không; kiểm tra `latest_exported_at` |
| 2 | `cat artifacts/logs/run_<run_id>.log` | Tìm `FAIL (halt)` hoặc `WARN:` để xác định expectation nào vi phạm |
| 3 | Mở `artifacts/quarantine/quarantine_<run_id>.csv` | Kiểm tra cột `reason` — xem có `stale_refund`, `stale_hr_*`, hay pattern lạ không |
| 4 | `python eval_retrieval.py --out /tmp/diag_eval.csv` | Cột `hits_forbidden` và `top1_preview` cho biết chunk nào đang được serve |
| 5 | `python grading_run.py --out artifacts/eval/grading_run.jsonl` | Câu nào fail `contains_expected` hoặc `top1_doc_expected` → xác định doc_id bị ảnh hưởng |

**Query nhanh để xem chunk trong store:**
```python
import chromadb
client = chromadb.PersistentClient(path="./chroma_db")
col = client.get_collection("day10_kb")
results = col.query(query_texts=["hoàn tiền bao nhiêu ngày"], n_results=5)
for doc in results["documents"][0]:
    print(doc[:100])
```

---

## Mitigation

**Trường hợp S1/S2 — Chunk bẩn đã vào store:**

```bash
# Bước 1: Xác nhận manifest của run bẩn
cat artifacts/manifests/manifest_inject-bad.json

# Bước 2: Chạy lại pipeline sạch để prune chunk bẩn
python etl_pipeline.py run

# Bước 3: Xác nhận log có embed_prune_removed >= 1
grep "embed_prune_removed" artifacts/logs/run_*.log | tail -1

# Bước 4: Verify eval
python eval_retrieval.py --out /tmp/verify_clean.csv
grep "hits_forbidden" /tmp/verify_clean.csv
```

**Trường hợp S3 — Freshness FAIL:**
- Trigger re-export từ nguồn (Policy System, HRIS, v.v.)
- Đặt file export mới vào `data/raw/` với `exported_at` cập nhật
- Chạy lại `python etl_pipeline.py run`
- Nếu chưa có export mới: dùng banner "⚠️ Dữ liệu đang được cập nhật, vui lòng liên hệ CS" trong agent response

**Rollback về run tốt nhất gần nhất:**
```bash
# Xem các run đã có manifest
ls artifacts/manifests/

# Rebuild store từ cleaned CSV của run cụ thể (nếu cần)
# Lưu ý: etl_pipeline upsert idempotent — chạy lại với cleaned đúng sẽ overwrite
```

---

## Prevention

1. **Không dùng `--skip-validate` ngoài demo** — xóa flag này khỏi production entrypoint.
2. **Thêm expectation E9** cho `exported_at` phải trong 24h — halt nếu FAIL (không chỉ log warn).
3. **Alert tự động:** Sau mỗi pipeline run, gửi Slack message với `freshness_check` result và `expectation_fail_count`.
4. **Owner per doc_id:** Mỗi nguồn cần 1 người chịu trách nhiệm re-export — ghi trong `data_contract.yaml::owner`.
5. **Nối Day 11 guardrail:** Nếu có guardrail layer (Day 11), thêm check `hits_forbidden` như 1 pre-serve gate — agent không trả lời nếu top-k chứa forbidden chunk.
