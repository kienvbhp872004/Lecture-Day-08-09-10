# Kiến trúc pipeline — Lab Day 10

**Nhóm:** AICB-P1  
**Cập nhật:** 2026-06-10

---

## 1. Sơ đồ luồng

```mermaid
flowchart LR
    A["data/raw/\npolicy_export_dirty.csv\n(247 records)"] -->|ingest| B["cleaning_rules.py\nRules 1–11"]
    B -->|quarantine_csv| C["artifacts/quarantine/\nquarantine_<run_id>.csv\n(213 records)"]
    B -->|cleaned_csv\n(34 records)| D["expectations.py\nE1–E8"]
    D -->|HALT if fail| E["❌ Pipeline stop\n(exit 1)"]
    D -->|all halt pass| F["embed_chunks()\nall-MiniLM-L6-v2\n+ ChromaDB upsert"]
    F -->|upsert by chunk_id| G["chroma_db/\ncollection: day10_kb"]
    F -->|embed_prune_removed| G
    F -->|manifest_written| H["artifacts/manifests/\nmanifest_<run_id>.json\n(run_id, counts, exported_at)"]
    H -->|freshness_check| I["monitoring/freshness_check.py\nPASS / WARN / FAIL"]
    G -->|serve| J["eval_retrieval.py\ngrading_run.py\n(Day 08/09 RAG)"]

    style C fill:#ffd700,color:#000
    style E fill:#ff6b6b,color:#fff
    style I fill:#51cf66,color:#fff
```

**Freshness** đo tại `publish` — `latest_exported_at` trong manifest so với `FRESHNESS_SLA_HOURS=24`.  
**run_id** được ghi vào tên log, manifest, cleaned CSV, quarantine CSV để trace đầy đủ.

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | Owner |
|------------|-------|--------|-------|
| **Ingest** | `data/raw/policy_export_dirty.csv` | `raw_rows` (list dict in-memory) | `etl_pipeline.py::ingest_raw()` |
| **Transform** | `raw_rows` | `cleaned_rows`, `quarantine_rows` | `transform/cleaning_rules.py::apply_rules()` |
| **Quality** | `cleaned_rows` | `ExpectationResult[]`, halt/warn decision | `quality/expectations.py::run_suite()` |
| **Embed** | `cleaned_rows` | ChromaDB upsert, prune removed IDs | `etl_pipeline.py::embed_chunks()` |
| **Monitor** | `manifest_*.json` | freshness PASS/WARN/FAIL + JSON detail | `monitoring/freshness_check.py` |

---

## 3. Idempotency & rerun

- Mỗi chunk có `chunk_id` ổn định (thường `{doc_id}_{seq}`).
- ChromaDB `collection.upsert()` dùng `chunk_id` làm ID — chạy lại 2 lần không sinh duplicate vector.
- Nếu run mới loại bỏ chunk cũ (ví dụ chunk bị quarantine lần sau), pipeline chạy `embed_prune_removed`: so sánh `chunk_id` mới vs cũ trong collection và xóa những ID không còn trong cleaned.
- Log ghi `embed_prune_removed=N` để audit.

---

## 4. Liên hệ Day 08 / Day 09

- Day 08/09 dùng `ChromaDB collection day10_kb` làm retriever trong RAG agent.
- Day 10 pipeline là **upstream producer** — mỗi khi policy thay đổi, chạy lại `python etl_pipeline.py run` để cập nhật store.
- Canonical source: `data/docs/*.txt` (5 file) — không sửa trực tiếp file này; thay vào đó cập nhật `policy_export_dirty.csv` rồi re-ingest.
- `data/test_questions.json` và `data/grading_questions.json` dùng chung giữa Day 10 eval và Day 08/09 demo.

---

## 5. Rủi ro đã biết

- `all-MiniLM-L6-v2` là model tiếng Anh — retrieval tiếng Việt kém chính xác, đặc biệt câu ngắn/từ viết tắt. Giải pháp: thay bằng `paraphrase-multilingual-MiniLM-L12-v2`.
- Freshness đo 1 boundary (`exported_at`). Nếu ingest bị lag sau export, SLA thực tế sẽ bị miss. Cần thêm `ingested_at` timestamp.
- `--skip-validate` là tính năng demo Sprint 3 — production code không nên có flag này.
- Nếu 2 agent (Day 09) query collection cùng lúc khi pipeline đang prune, có thể đọc state trung gian. ChromaDB embedded mode không có transaction isolation.
