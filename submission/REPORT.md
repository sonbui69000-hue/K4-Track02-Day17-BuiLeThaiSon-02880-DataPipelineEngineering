# K4-Track02-Day17 — Report cá nhân

Phần phân tích tối đa một trang, không tính output ở phần 5.
Định dạng tham chiếu và phạm vi tính trang: [SUBMISSION.md](../docs/SUBMISSION.md).

**Họ tên / MSSV:** Bui Le Thai Son / 02880
**Repo:** https://github.com/sonbui69000-hue/K4-Track02-Day17-BuiLeThaiSon-02880-DataPipelineEngineering
**Commit bài nộp:**
**AI đã dùng và phạm vi hỗ trợ (hoặc không dùng):** GPT, dùng để sửa cách dùng từ khi viết report
**Nguồn tham khảo khác (nếu có):**

## 1. Ba lỗi

| Lỗi | Triệu chứng -> nguyên nhân -> cách sửa -> khái niệm |
|---|---|
| Silver | Nhiều hàng cho T-91 -> chỉ `INSERT`, không upsert giữa batch -> `MERGE` theo `ticket_id`, update khi `_lsn` mới hơn trong `pipeline/silver.py` -> Silver có khoá, LSN guard. |
| Late data | Feature không khớp full recompute, event u05 ngày 12/08 bị bỏ -> `LOOKBACK_DAYS=0` -> đặt `3=ceil(P99)` trong `pipeline/config.py`, vẫn nhóm theo `event_time` -> event time/lookback. |
| CDC delete | T-97 còn ở Silver/training/RAG -> parser lấy khoá chỉ từ `after`, nhưng delete có `after=null` -> `COALESCE(after.ticket_id,key.ticket_id,before.ticket_id)`, giữ tombstone `_lsn` và loại khỏi Gold -> CDC delete, Kafka tombstone và “xoá phải lan”. |

Seed cho thấy T-91 có các trạng thái `low/open -> high/closed/bug`, T-97 có `op="d"`
và tombstone, còn event u05 có `event_time` 12/08 nhưng ingest 15/08.

## 2. Các con số

- P99 lateness đo từ Bronze: `3.00` ngày -> `LOOKBACK_DAYS = 3`
- `submission/checksums.txt`: PASS — Gold checksum: `39e115c510ecdf526800eac227158a4f`
- `make parity`: PARITY, `silver_tickets` lite/dbt `3c15dfd43701`, `gold_feature_daily` lite/dbt `8630e04a61d1`

## 3. Lựa chọn công cụ / kỹ thuật (mỗi dòng một câu "vì sao")

- MERGE/overwrite-partition: Silver cần một trạng thái theo key, feature cần dựng lại partition bị late event ảnh hưởng.
- Tombstone: giữ key, `is_deleted` và `_lsn` để chống hồi sinh khi replay và truyền delete xuống Gold.
- Snapshot “as of”: bảo đảm point-in-time reproducibility, snapshot cũ giữ lịch sử, snapshot mới phản ánh delete.
- DuckDB/dbt: dữ liệu nhỏ, local zero-key, checksum dễ kiểm tra, dbt thêm merge, microbatch, contract và parity.

## 4. Hai câu hỏi suy ngẫm

1. Snapshot `v2026-08-12`..`v2026-08-14` vẫn chứa văn bản của T-97 (đã bị xoá ngày
08-15). "Snapshot bất biến" và "quyền được xoá dữ liệu" mâu thuẫn — bạn xử lý thế nào?
   Trong lab, giữ snapshot cũ để tái lập lịch sử nhưng loại T-97 khỏi snapshot mới và
   Gold hiện tại. Production cần legal purge có audit, xoá/ẩn PII trong snapshot/index,
   có thể huỷ khoá mã hoá và chỉ giữ metadata tối thiểu.
2. Regex che được email và số điện thoại, nhưng tên "Nguyễn Văn An" vẫn còn. Bạn sẽ
đặt chốt PII nào, ở tầng nào, và đo nó ra sao?
   Đặt quality gate sau Bronze trước Silver, dùng recognizer/NER tiếng Việt cho tên,
   địa chỉ và mã định danh, dữ liệu không chắc chắn vào quarantine. Đo precision,
   recall, F1, tỷ lệ PII lọt qua Silver/Gold và false positive trên bộ dữ liệu gán nhãn.

## 5. Output (dán nguyên văn)

```text
$ make verify

$ make test

$ make rerun3

$ make lateness

$ make dbt

$ make parity

verify: RESULT: 18/18 checks — ALL PASS
pytest: 34 passed in 3.10s
rerun3: RESULT: PASS — 3 re-runs, identical checksums
lateness: event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
           lookback must be >= ceil(p99) = 3 day(s), config.LOOKBACK_DAYS = 3
dbt build: PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19
parity: silver_tickets lite 3c15dfd43701 = dbt 3c15dfd43701,
        gold_feature_daily lite 8630e04a61d1 = dbt 8630e04a61d1,
        RESULT: PARITY
```

Nếu dùng PowerShell, ghi lệnh tương đương và output thực tế theo [SUBMISSION.md](../docs/SUBMISSION.md).
Nếu làm bonus, thêm output B1 hoặc đường dẫn bằng chứng B2 ở cuối phần này.
