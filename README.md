# INF1103-DoNotStress

INF1103 group project — DoNotStress student academic and wellbeing risk early-warning system.

This tree is the **base four-layer framework**: four pure procedural Python modules (functions only, no classes) plus the tests that shipped with each layer’s first push. There is no web app, Flask UI, or application entry point yet. Console I/O stays inside `io_manager`; the other layers never call `print()` or `input()`.

Pipeline:

```
User → io_manager → ai_manager → logic_manager → data_manager
```

## Layers

### 1. I/O — `io_manager.py`

The only module allowed to talk to the console. Validators are pure (`validate_X(raw) -> (ok, value_or_error)`) so they can be tested without stdin.

`collect_student_record()` prompts until every field is valid and returns a snake_case dict:

| Field | Rule |
| --- | --- |
| `student_id` | 7 digits: year `23`, `24`, `25`, or `26`, then 5 digits (example `2605581`) |
| `sleep_hours` | float in `[0.0, 24.0]` |
| `stress_level` | integer in `[1, 10]` |
| `submission_rate` | float in `[0.0, 100.0]` |
| `cca_count` | integer `>= 0` |
| `financial_stress` | `yes` / `no` (stored as a bool) |
| `consecutive_absences` | integer `>= 0` |
| `free_text_concern` | optional text; blank becomes `""` |

Formatters (`format_student_record`, `format_cohort_list`, `format_summary_view`) return strings. `print_*` helpers are the console output API other layers should call.

### 2. AI processing — `ai_manager.py`

Builds a Gemini prompt from a validated student record, requests structured JSON, validates the schema, and retries (default 3 attempts, `gemini-2.0-flash`). On success, `analyse_student()` returns a shallow copy of the student record plus:

- `risk_score` — float in `[0.0, 1.0]`
- `risk_category` — `"Low"`, `"Moderate"`, or `"High"`
- `primary_stressors` — list of short strings
- `recommended_support` — non-empty string
- `confidence` — float in `[0.0, 1.0]`
- `reasoning` — non-empty string

API and schema failures return `(False, None, error)` and do not raise. A live call reads `GEMINI_API_KEY` from the environment. Tests inject `generate_fn` and do not call Gemini. This layer does not assign intervention tiers or write files.

### 3. Logic — `logic_manager.py`

Sole owner of Intervention Tier rules. `assign_intervention_tier()` returns `tier` / `intervention_tier` (1–4), `label` / `tier_label`, `colour`, `rule`, and `action` / `outcome`. `apply_intervention_tier()` merges that outcome into a copy of the record for `data_manager.save_record()`.

Rules are checked in order; the first match wins:

| Tier | Label | Colour | Rule | Condition | Action |
| --- | --- | --- | --- | --- | --- |
| 1 | Critical | Red | `critical_escalation` | `risk_score > 0.75` and `consecutive_absences >= 3` | Escalate to Dean's Office immediately. |
| 2 | Urgent | Orange | `urgent_financial` | `risk_score > 0.65`, `financial_stress` is true, and `"financial_pressure"` is in `primary_stressors` | Refer to Financial Aid Office and Student Counsellor. |
| 3 | Monitor | Yellow | `academic_watch` | `submission_rate < 50.0`, `risk_category == "High"`, and `consecutive_absences >= 1` | Flag for Academic Advisor follow-up within two weeks. |
| 3 | Monitor | Yellow | `physical_wellness` | `sleep_hours < 5.5`, `stress_level >= 7`, and `risk_score > 0.5` | Recommend Wellness Workshop and peer-support resources. |
| 4 | Clear | Green | `default_clear` | none of the above | No immediate action. Record saved for trend tracking. |

`submission_rate` may also be read as `assignment_submission_rate`, and `sleep_hours` as `average_sleep_hours`.

### 4. Data — `data_manager.py`

Sole owner of flat-file persistence. Default store: `data/student_records.json` (gitignored; the `data/` directory is kept with `.gitkeep`).

- `load_all_records()` returns `{"ok", "records", "error", "path"}`. A missing, empty, or corrupt file yields an empty list and does not raise.
- `save_record()` appends one fully evaluated record (inputs + AI fields + tier) and sets `saved_at` to a UTC ISO timestamp when it is absent. A corrupt existing file is replaced by a fresh list containing the new record.
- `filter_records()` matches `intervention_tier` / `tier`, `student_id`, `risk_category` (case-insensitive), and `cohort_year` (two-digit `student_id` prefix). Helpers `get_records_by_tier`, `get_record_by_student_id`, `get_records_by_risk_category`, and `get_records_by_cohort_year` load the file and filter.

## Tests

```bash
python3 -m pytest tests/
```

| File | Layer |
| --- | --- |
| `tests/test_io_manager.py` | validators and formatters |
| `tests/test_ai_manager.py` | schema validation, retries, `analyse_student` (stubbed Gemini) |
| `tests/test_logic_manager.py` | intervention tier rules |
| `tests/test_data_manager.py` | JSON load, save, and filters |

## Source

These modules and tests match [ethancheang/DoNotStressGrokkies](https://github.com/ethancheang/DoNotStressGrokkies) at commit `7d45b5872fcc49096fed5f0116e68e35f6a1bcd1` (2026-09-12), the merge of PR #4 and the tree after the first push of each layer:

- PR #1 — I/O (`io_manager.py`)
- PR #2 — AI (`ai_manager.py`)
- PR #3 — Logic (`logic_manager.py`)
- PR #4 — Data (`data_manager.py`)

Later Grokkies work (student schema rewrite, Flask check-in UI, mandatory Gemini) is not part of this base.
