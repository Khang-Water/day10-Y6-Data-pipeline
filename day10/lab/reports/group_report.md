# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** ___________  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Hoàng Thị Thanh Tuyền | Pipeline Coordinator & Ingestion Owner | hoangthanhtuyen1412@gmail.com |
| Nguyễn Hồ Bảo Thiên | Cleaning & Quality Owner | thiennguyen3703@gmail.com |
| Võ Thanh Chung | Quality / Expectations Owner | 	vothanhchung95@gmail.com |
| Dương Khoa Điềm | Embed & Idempotency Owner | duongkhoadiemp@gmail.com |
| Đỗ Thế Anh | Eval & Inject (Before/After) Owner | anh.dothe47@gmail.com |
| Lê Minh Khang | Monitoring / Docs Owner | minhkhangle2k4@gmail.com |

**Ngày nộp:** 4/15/2026
**Repo:** https://github.com/Khang-Water/day10-Y6-Data-pipeline
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


Rule 1 (Mới): Phát hiện tàn dư lỗi từ quá trình trích xuất PDF/Word (Broken Artifacts).
    
Rule 2: Phát hiện và làm sạch mã định dạng / HTML tags thừa (Format Stripping).
Rule 3: Phát hiện rò rỉ dữ liệu nhạy cảm cá nhân (PII - Personally Identifiable Information).
    

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| … | … | … | … |

**Rule chính (baseline + mở rộng):**

- …

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**

_________________

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

Nhóm thực hiện 2 run liên tiếp để so sánh trực tiếp tác động dữ liệu lên retrieval. Run xấu dùng `run_id=inject-bad` với cờ `--no-refund-fix --skip-validate` để cố ý cho phép dữ liệu stale đi qua bước embed. Mục tiêu là giữ lại ngữ cảnh sai của policy refund (14 ngày) trong top-k retrieval. Sau đó chạy lại run sạch `run_id=sprint3-clean` (không dùng cờ inject) để pipeline áp dụng đầy đủ cleaning + validation trước khi publish vào vector store. Kết quả được ghi lần lượt vào `artifacts/eval/after_inject_bad.csv` và `artifacts/eval/before_after_eval.csv`.

**Kết quả định lượng (từ CSV / bảng):**

So sánh theo từng câu hỏi cho thấy degradation xuất hiện đúng ở câu mục tiêu `q_refund_window`:

| Question | Inject (`after_inject_bad.csv`) | Clean (`before_after_eval.csv`) | Nhận định |
|---|---|---|---|
| `q_refund_window` | `contains_expected=yes`, `hits_forbidden=yes` | `contains_expected=yes`, `hits_forbidden=no` | Inject làm bẩn context top-k (vẫn có tín hiệu đúng nhưng lẫn chunk cấm). |
| `q_p1_sla` | `yes/no` | `yes/no` | Ổn định, không bị ảnh hưởng bởi inject refund. |
| `q_lockout` | `yes/no` | `yes/no` | Ổn định, không có dấu hiệu nhiễm context sai. |
| `q_leave_version` | `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes` | `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes` | Đạt tiêu chí Merit cho câu leave version. |

Kết luận: Sprint 3 chứng minh rõ dữ liệu xấu có thể làm retrieval giảm chất lượng ngay cả khi top-1 trông vẫn đúng. Chỉ số `hits_forbidden` là tín hiệu quan trọng vì quét toàn bộ top-k, phát hiện trường hợp câu trả lời có vẻ đúng nhưng context đã nhiễm chunk stale. Sau khi chạy lại pipeline sạch, chỉ số này quay về an toàn (`no`) tại câu refund, xác nhận cleaning + validate + publish boundary hoạt động đúng mục tiêu observability.

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
