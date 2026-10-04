# K4-Track02-Day17 — Report cá nhân

**Họ tên / MSSV:** Dau Van Thach / 2A202602592

**Repo:** https://github.com/thach-s/K4-Track02-Day17-DauVanThach-2A202602592-DataPipelineEngineering

**Commit bài nộp:** HEAD (commit chứa REPORT và `submission/checksums.txt` này)

**AI đã dùng và phạm vi hỗ trợ:** OpenAI Codex hỗ trợ đọc yêu cầu, chạy baseline, chẩn đoán và sửa ba lỗi, chạy kiểm thử, soạn REPORT; tôi đã kiểm tra diff và kết quả thực tế.

**Nguồn tham khảo khác:** Không có; sử dụng tài liệu và mã nguồn trong repo.

## 1. Ba lỗi

| | Lỗi Silver | Lỗi late data | Lỗi xoá (CDC) |
|---|---|---|---|
| **Triệu chứng** | Verify báo 24 hàng cho 12 ticket; T-91 có ba trạng thái thay vì trạng thái mới nhất. | Checksum feature `c50b8851affe` khác full recompute `8630e04a61d1`; u05 ngày 08-12 chỉ có `(2 events, 0 feedback down)`. | T-97 còn hai hàng chứa dữ liệu cá nhân, vẫn xuất hiện trong snapshot mới nhất và RAG chunks. |
| **Nguyên nhân gốc** | `upsert_silver_tickets` dùng `INSERT`, không có khóa và không bảo vệ trạng thái mới bằng LSN. | `LOOKBACK_DAYS = 0` trong khi lateness P99 đo từ Bronze là 3 ngày. | Delete Debezium có `after = null`, nhưng staging chỉ lấy `ticket_id` từ `after`, nên bản ghi delete bị loại. |
| **Cách sửa** | `pipeline/silver.py`: `MERGE` theo `ticket_id`, update chỉ khi `s._lsn > t._lsn`. | `pipeline/config.py`: đặt `LOOKBACK_DAYS = 3` để overwrite lại các partition event-time trong cửa sổ. | `pipeline/staging.py`: lấy khóa bằng `coalesce(after.ticket_id, before.ticket_id)`; Silver nhận tombstone với các cột PII null. |
| **Khái niệm trên slide** | Silver có khóa; keyed upsert/MERGE; LSN guard giúp replay batch cũ không ghi đè trạng thái mới. | Event time khác ingest time; lookback phải được đo, không đoán; overwrite-partition để nhận dữ liệu muộn. | CDC log-based; khóa delete nằm trong `before`; “xoá phải lan” xuống training snapshot mới nhất và RAG. |

## 2. Các con số

- P99 lateness đo từ Bronze: `3.00` ngày → `LOOKBACK_DAYS = 3`
- `submission/checksums.txt`: PASS — Gold checksum: `39e115c510ecdf526800eac227158a4f`
- `make parity`: PARITY

## 3. Lựa chọn công cụ / kỹ thuật

- MERGE theo khóa phù hợp với bảng thực thể `silver_tickets`; overwrite-partition phù hợp với aggregate event-time vì cần tính lại trọn partition chịu ảnh hưởng bởi late data.
- Tombstone giữ LSN của thao tác xóa để replay thay đổi cũ không làm ticket sống lại, dù đánh đổi là Silver giữ một hàng tối thiểu lâu dài.
- Snapshot training dựng lại từ Bronze “as of” ngày đó để tái lập được và không làm thay đổi dữ liệu huấn luyện đã dùng; thay đổi mới tạo version mới.
- DuckDB đủ nhẹ, zero-key và tái lập tốt cho seed nhỏ; dbt bổ sung contract, test và incremental model rõ ràng, chưa cần chi phí vận hành Spark phân tán.

## 4. Hai câu hỏi suy ngẫm

1. Bất biến không được ưu tiên hơn yêu cầu xóa hợp pháp. Tôi sẽ lưu manifest/audit không chứa PII, thu hồi và dựng lại các snapshot bị ảnh hưởng thành version thay thế, xóa vật lý bản cũ khỏi storage/cache/backup theo retention, đồng thời lưu mapping version bị thu hồi để cấm sử dụng lại. Model đã học từ dữ liệu đó cũng cần quy trình đánh giá, retrain hoặc machine unlearning tùy mức rủi ro và yêu cầu pháp lý.
2. Tôi đặt chốt PII ngay khi chuyển Bronze → Silver để Bronze raw chỉ được truy cập hạn chế, đồng thời quét lại trước khi xuất Gold. Ngoài regex, dùng NER/DLP cho tên, địa chỉ và định danh theo ngữ cảnh; đo precision/recall trên tập gán nhãn, theo dõi false negative theo từng loại PII và chặn pipeline khi tỷ lệ rò rỉ vượt ngưỡng.

## 5. Output thực tế

```text
$ make verify
RESULT: 18/18 checks — ALL PASS
re-run checksums written to submission/checksums.txt

$ make test
..................................                                       [100%]
34 passed in 5.01s

$ make rerun3
run                     gold_feature_daily    gold_training_set     gold_doc_chunks       gold (combined)
fresh build             8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #1 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #2 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #3 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
RESULT: PASS — 3 re-runs, identical checksums

$ make lateness
event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
-> lookback must be >= ceil(p99) = 3 day(s); config.LOOKBACK_DAYS = 3

$ make dbt
Finished running 3 incremental models, 13 data tests, 1 unit test, 2 view models.
Completed successfully
Done. PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19

$ make parity
=== parity: lite pipeline vs dbt ===
  [OK ] silver_tickets       lite 3c15dfd43701  dbt 3c15dfd43701
  [OK ] gold_feature_daily   lite 8630e04a61d1  dbt 8630e04a61d1
RESULT: PARITY — both implementations agree
```
