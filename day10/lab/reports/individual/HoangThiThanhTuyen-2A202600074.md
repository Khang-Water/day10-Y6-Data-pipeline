# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Hoàng Thị Thanh Tuyền
**Vai trò:** Ingestion / Pipeline Coordinator  
**Ngày nộp:** 2026-04-15  
**Độ dài:** khoảng 520 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong Day 10, tôi phụ trách vai trò Ingestion Owner và điều phối luồng chạy pipeline đầu-cuối. Trách nhiệm chính của tôi là bảo đảm mỗi lần chạy có khả năng truy vết rõ bằng run_id và các chỉ số record-level ngay ở log đầu ra để cả nhóm dùng chung baseline trước khi mở rộng cleaning, expectation, embed và monitoring.

Tôi tập trung vào entrypoint, phần ghi log và kiểm soát nhánh xử lý khi expectation fail. Ở run baseline sprint1, log thể hiện rõ run_id=sprint1, raw_records=10, cleaned_records=6, quarantine_records=4. Tôi phối hợp trực tiếp với Dương Khoa Diễm ở phần tổng hợp báo cáo nhóm và thống nhất cách đối chiếu run_id giữa log, manifest và artifacts để đảm bảo số liệu không bị lệch giữa các thành viên.

**File / module:**

- `etl_pipeline.py`
- `artifacts/logs/run_sprint1.log`
- `artifacts/manifests/manifest_sprint1.json`

**Kết nối với thành viên khác:**
Tôi bàn giao baseline run cho bạn phụ trách cleaning và expectation để test rule mới, đồng thời phối hợp với bạn phụ trách embed để kiểm tra nhánh halt và skip-validate trong Sprint 3.

**Bằng chứng (commit / comment trong code):**
Đã commit, đoạn cmd_run ghi đủ run_id, raw_records, cleaned_records, quarantine_records và branch theo cờ chạy đã có trong `etl_pipeline.py`; log thực tế nằm ở `run_sprint1.log` và `run_inject-bad.log`.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định kỹ thuật quan trọng nhất tôi chốt là tách hai chế độ kiểm soát chất lượng: chế độ mặc định fail-fast và chế độ bypass có kiểm soát cho demo corruption. Ở chế độ mặc định, khi expectation mức halt fail thì pipeline dừng để không publish dữ liệu bẩn vào collection. Tuy nhiên Sprint 3 cần mô phỏng dữ liệu xấu có chủ đích, nên tôi giữ cờ skip-validate để hệ thống ghi cảnh báo rồi vẫn tiếp tục embed. Cách này giúp vừa an toàn khi chạy thật, vừa linh hoạt cho việc tạo bằng chứng before/after. Song song đó, tôi giữ cờ no-refund-fix độc lập để bật hoặc tắt rule sửa stale refund window 14 ngày về 7 ngày, vì expectation refund phụ thuộc trực tiếp vào rule này. Tôi cũng lưu trạng thái các cờ vào manifest để audit theo run_id về sau.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Anomaly tôi xử lý là stale refund window trong kịch bản inject. Triệu chứng xuất hiện khi chạy run_id inject-bad: expectation refund_no_stale_14d_window báo FAIL với violations=1. Ở lần chạy strict mode, pipeline dừng với trạng thái PIPELINE_HALT, đây là hành vi đúng để ngăn dữ liệu sai policy đi vào publish. Sau đó tôi kiểm tra nhánh phục vụ demo bằng cách bật skip-validate. Kết quả log thể hiện rõ cảnh báo expectation failed but --skip-validate rồi pipeline tiếp tục embed thành công với embed_upsert count=6. Nhờ đó, lỗi không bị che giấu mà vẫn được ghi nhận minh bạch, đồng thời nhóm vẫn có dữ liệu để chứng minh tác động chất lượng trong phần before/after. Việc tách rõ hai nhánh xử lý giúp team phân biệt được production safety và demo scenario, tránh hiểu nhầm rằng pipeline đã bỏ qua chất lượng mặc định.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Tôi dùng run_id sprint1 làm baseline sạch và run_id inject-bad làm kịch bản xấu có chủ đích.

Hai dòng bằng chứng trước/sau:

- run_id=sprint1 | raw_records=10 | cleaned_records=6 | quarantine_records=4
- run_id=inject-bad | expectation refund_no_stale_14d_window FAIL (violations=1)

Bổ sung chứng cứ dữ liệu:

- Trong cleaned_sprint1.csv có dòng policy_refund_v4 đã được chuẩn hóa thành 7 ngày và có marker cleaned: stale_refund_window.
- Trong quarantine_sprint1.csv có các lý do cách ly rõ ràng: duplicate_chunk_text, missing_effective_date, stale_hr_policy_effective_date, unknown_doc_id.

Số liệu thay đổi chính tôi dùng để đối chiếu với metric_impact của nhóm là trạng thái expectation refund từ OK ở sprint1 sang FAIL ở inject-bad, trong khi số lượng cleaned_records và quarantine_records giữ nguyên để làm rõ đây là lỗi chất lượng nội dung, không phải lỗi volume.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ thêm bước tự động tổng hợp metric theo run_id sau mỗi lần chạy, liên kết trực tiếp log, manifest và eval thành một bảng duy nhất để giảm đối chiếu thủ công. Mục tiêu là nhóm chỉ cần đọc một nguồn chuẩn khi điền group report, hạn chế sai lệch số liệu giữa các artifact và rút ngắn thời gian review chéo.
