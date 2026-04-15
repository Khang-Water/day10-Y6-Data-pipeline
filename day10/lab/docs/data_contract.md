# Data Contract - Lab Day 10

This document is the human-readable companion of `contracts/data_contract.yaml`.

---

## 1. Source map

| Source | Ingest method | Main failure modes | Metric / alert |
|-------|---------------|--------------------|----------------|
| `data/raw/policy_export_dirty.csv` (raw export snapshot) | `csv.DictReader` in `load_raw_csv` | unknown `doc_id`, missing/invalid dates, duplicate chunks | `raw_records`, `quarantine_records`, expectation fail logs |
| `data/docs/policy_refund_v4.txt` (canonical refund policy) | mapped via `doc_id=policy_refund_v4` and cleaning normalization | stale text from previous version (14-day window) | expectation `refund_no_stale_14d_window` (halt) |
| `data/docs/hr_leave_policy.txt` (canonical HR leave policy) | mapped via `doc_id=hr_leave_policy` and effective date gate | stale 2025 content mixed into 2026 serving set | quarantine reason `stale_hr_policy_effective_date`; expectation `hr_leave_no_stale_10d_annual` |

---

## 2. Cleaned schema

| Column | Type | Required | Notes |
|-----|------|----------|-------|
| `chunk_id` | string | Yes | stable ID generated from `doc_id + chunk_text + seq` |
| `doc_id` | string | Yes | must be in `allowed_doc_ids` |
| `chunk_text` | string | Yes | normalized payload for embedding, minimum length 8 |
| `effective_date` | date | Yes | normalized to `YYYY-MM-DD` |
| `exported_at` | datetime | Yes | source export timestamp used for freshness context |

---

## 3. Quarantine vs drop policy

- Invalid or suspicious records are quarantined to `artifacts/quarantine/quarantine_<run_id>.csv` with a `reason` field.
- Reasons currently include: `unknown_doc_id`, `missing_effective_date`, `invalid_effective_date_format`, `stale_hr_policy_effective_date`, `missing_chunk_text`, `duplicate_chunk_text`.
- We do not silently drop rows; quarantine is retained for audit and manual review.
- Re-admission rule: record can re-enter only after source fix and a rerun that passes expectations.

---

## 4. Versioning and canonical ownership

- Canonical refund source of truth: `data/docs/policy_refund_v4.txt`.
- Canonical HR source of truth: `data/docs/hr_leave_policy.txt` with minimum allowed effective date `2026-01-01`.
- Dataset owner: `day10-data-pipeline`.
- Freshness SLA: 24 hours at publish boundary, alert channel `#data-observability`.
