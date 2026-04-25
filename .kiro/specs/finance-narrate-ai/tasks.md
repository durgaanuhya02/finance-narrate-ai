# Implementation Plan: FinanceNarrate AI

## Overview

Implement the FinanceNarrate AI Executive Financial Insight Engine incrementally: project scaffolding and data models first, then the Pandas processing pipeline, then the Gemini LLM client, then the FastAPI endpoints, and finally the frontend dashboard. Property-based tests (Hypothesis) are placed immediately after the code they validate to catch regressions early.

## Tasks

- [x] 1. Project scaffolding and dependencies
  - Create the directory structure: `finance_narrate/backend/`, `finance_narrate/frontend/`, `finance_narrate/sample_data/`
  - Create `backend/requirements.txt` listing: `fastapi`, `uvicorn[standard]`, `pandas`, `openpyxl`, `python-multipart`, `google-generativeai`, `python-dotenv`, `pydantic`, `hypothesis`, `pytest`, `httpx`
  - Create `backend/.env` with a placeholder `GEMINI_API_KEY=your_key_here`
  - Create `sample_data/sample_finance.csv` with at least 24 rows covering multiple months, categories, and at least one expense anomaly and one revenue dip
  - _Requirements: 6.2, 6.4_

- [x] 2. Pydantic data models (`backend/models.py`)
  - [x] 2.1 Implement all Pydantic v2 models
    - Define `UploadResponse`, `MonthlyRevenue`, `MoMGrowth`, `TopCategory`, `ExpenseAnomaly`, `RevenueDip`, `MetricsResult`, `NarrativeResult`, `ReportResult` exactly as specified in the design
    - Add docstrings to every class describing its purpose and fields
    - _Requirements: 1.6, 2.8, 3.7, 4.2, 6.5_

- [x] 3. Pandas processing pipeline (`backend/processor.py`)
  - [x] 3.1 Implement `load_dataframe`
    - Load CSV or Excel file from a `Path`; parse `Date` column as datetime; return a `pd.DataFrame`
    - Add docstring
    - _Requirements: 2.1_

  - [x] 3.2 Implement `compute_monthly_revenue`
    - Group by `YYYY-MM` month string; sum `Revenue`; return `dict[str, float]` sorted chronologically
    - Add docstring
    - _Requirements: 2.3_

  - [ ]* 3.3 Write property test for `compute_monthly_revenue` (Property 4)
    - **Property 4: Monthly Revenue Aggregation Is Lossless**
    - **Validates: Requirements 2.3, 2.9**
    - Use `@given` with a Hypothesis strategy that generates DataFrames with random dates and positive revenue values
    - Assert `sum(monthly_revenue.values()) == pytest.approx(df["Revenue"].sum())`
    - Tag: `# Feature: finance-narrate-ai, Property 4: Monthly Revenue Aggregation Is Lossless`

  - [x] 3.4 Implement `compute_mom_growth`
    - Iterate over sorted monthly dict; compute `(C - P) / P * 100` for each consecutive pair; return `dict[str, float]`
    - Add docstring
    - _Requirements: 2.4_

  - [ ]* 3.5 Write property test for `compute_mom_growth` (Property 5)
    - **Property 5: Month-over-Month Growth Is Mathematically Correct**
    - **Validates: Requirements 2.4**
    - Use `@given` with a strategy generating ordered lists of `(month_str, revenue)` pairs with non-zero revenues
    - Assert each `growth_pct == pytest.approx((C - P) / P * 100)`
    - Tag: `# Feature: finance-narrate-ai, Property 5: Month-over-Month Growth Is Mathematically Correct`

  - [x] 3.6 Implement `compute_top_categories`
    - Group by `Category`; sum `Expenses`; return top `n` (default 3) as list of dicts sorted descending by total
    - Add docstring
    - _Requirements: 2.5_

  - [ ]* 3.7 Write property test for `compute_top_categories` (Property 6)
    - **Property 6: Top Categories Are the Highest-Total Categories**
    - **Validates: Requirements 2.5**
    - Use `@given` with a strategy generating DataFrames with random category names and positive expense values
    - Assert every returned category total >= every non-returned category total
    - Tag: `# Feature: finance-narrate-ai, Property 6: Top Categories Are the Highest-Total Categories`

  - [x] 3.8 Implement `detect_expense_anomalies`
    - Compute `mean` and `std` of `Expenses`; flag rows where `Expenses > mean + 2 * std`; return list of dicts with `row_index`, `date`, `category`, `expenses`, `z_score`
    - Add docstring
    - _Requirements: 2.6_

  - [ ]* 3.9 Write property test for `detect_expense_anomalies` (Property 7)
    - **Property 7: Expense Anomaly Detection Respects the 2-Std Threshold**
    - **Validates: Requirements 2.6**
    - Use `@given` with a strategy generating DataFrames with at least 3 rows and varied expense values
    - Assert every flagged row has `expenses > mean + 2 * std` and every non-flagged row has `expenses <= mean + 2 * std`
    - Tag: `# Feature: finance-narrate-ai, Property 7: Expense Anomaly Detection Respects the 2-Std Threshold`

  - [x] 3.10 Implement `detect_revenue_dips`
    - Iterate sorted monthly revenue; flag months where `(C - P) / P * 100 < -15.0`; return list of dicts with `month`, `revenue`, `previous_month`, `previous_revenue`, `drop_pct`
    - Add docstring
    - _Requirements: 2.7_

  - [ ]* 3.11 Write property test for `detect_revenue_dips` (Property 8)
    - **Property 8: Revenue Dip Detection Respects the 15% Threshold**
    - **Validates: Requirements 2.7**
    - Use `@given` with a strategy generating ordered lists of positive monthly revenue values
    - Assert every flagged month has `drop_pct < -15.0` and every non-flagged month has `drop_pct >= -15.0`
    - Tag: `# Feature: finance-narrate-ai, Property 8: Revenue Dip Detection Respects the 15% Threshold`

  - [x] 3.12 Implement `build_metrics`
    - Orchestrate all sub-functions; return a fully populated `MetricsResult`
    - Add docstring
    - _Requirements: 2.1, 2.8_

- [x] 4. Checkpoint — processor tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [x] 5. Gemini LLM client (`backend/llm_client.py`)
  - [x] 5.1 Implement startup key check and Gemini SDK initialisation
    - Load `GEMINI_API_KEY` from environment via `python-dotenv`; raise `RuntimeError` at module load if absent
    - Initialise `google.generativeai` with the key
    - Add docstring
    - _Requirements: 6.2, 6.3_

  - [x] 5.2 Implement `build_prompt`
    - Serialise the `MetricsResult` to a JSON string and embed it in a prompt that instructs Gemini to return a JSON object with exactly the four keys: `executive_summary`, `revenue_trends`, `anomalies`, `recommendations`
    - Specify tone (formal), length constraints (3–4 sentence summary, 2–3 recommendations), and output format
    - Add docstring
    - _Requirements: 3.2, 3.4_

  - [x] 5.3 Implement `call_gemini`
    - Call `gemini-1.5-flash` via the SDK with the prompt string; return the raw text response
    - Add docstring
    - _Requirements: 3.2_

  - [x] 5.4 Implement `parse_narrative`
    - Call `json.loads` on the raw response; construct and return a `NarrativeResult`; raise `HTTPException(502)` if JSON parsing fails
    - Add docstring
    - _Requirements: 3.5, 3.6_

  - [ ]* 5.5 Write property test for `parse_narrative` (Property 9)
    - **Property 9: Narrative Parse Round-Trip**
    - **Validates: Requirements 3.5**
    - Use `@given` with a strategy generating dicts with `executive_summary` (str), `revenue_trends` (list[str]), `anomalies` (list[str]), `recommendations` (list[str])
    - Serialise to JSON string; call `parse_narrative`; assert all fields equal the originals
    - Tag: `# Feature: finance-narrate-ai, Property 9: Narrative Parse Round-Trip`

  - [x] 5.6 Implement `generate_narrative`
    - Orchestrate `build_prompt` → `call_gemini` → `parse_narrative`; return `NarrativeResult`
    - Add docstring
    - _Requirements: 3.1, 3.7_

- [x] 6. FastAPI application (`backend/main.py`)
  - [x] 6.1 Set up FastAPI app with CORS middleware
    - Initialise `FastAPI` app; add `CORSMiddleware` allowing all origins (or configured frontend origin)
    - Declare module-level in-process stores: `metrics_store`, `narrative_store`, `upload_store`
    - Add docstring to the module
    - _Requirements: 6.1_

  - [x] 6.2 Implement `GET /health`
    - Return `{"status": "ok"}` with HTTP 200
    - _Requirements: 1.7_

  - [x] 6.3 Implement file Validator helper
    - Check extension is `.csv`, `.xlsx`, or `.xls`; raise `HTTPException(415)` otherwise
    - Read column headers; check for `Date`, `Revenue`, `Expenses`, `Category`; raise `HTTPException(422)` with missing column list otherwise
    - Add docstring
    - _Requirements: 1.2, 1.3, 1.4_

  - [ ]* 6.4 Write property test for Validator — missing columns (Property 1)
    - **Property 1: Column Validation Identifies All Missing Columns**
    - **Validates: Requirements 1.2, 1.3**
    - Use `@given` with a strategy generating random subsets of the four required columns as DataFrame headers
    - Post a CSV built from those headers to `POST /upload` via `httpx.AsyncClient`; assert the 422 response `detail` lists exactly the absent columns
    - Tag: `# Feature: finance-narrate-ai, Property 1: Column Validation Identifies All Missing Columns`

  - [ ]* 6.5 Write property test for Validator — invalid file type (Property 2)
    - **Property 2: Invalid File Type Always Returns 415**
    - **Validates: Requirements 1.4**
    - Use `@given` with a strategy generating random file extensions that are not `.csv`, `.xlsx`, or `.xls`
    - Post a file with that extension to `POST /upload`; assert HTTP 415 is returned
    - Tag: `# Feature: finance-narrate-ai, Property 2: Invalid File Type Always Returns 415`

  - [x] 6.6 Implement `POST /upload`
    - Validate file type and columns using the Validator helper
    - Generate a UUID `file_id`; save file bytes to `uploads/{file_id}/{filename}`
    - Count data rows; populate and store `UploadInfo`; return `UploadResponse`
    - Add docstring
    - _Requirements: 1.1, 1.5, 1.6_

  - [ ]* 6.7 Write property test for `POST /upload` — row count (Property 3)
    - **Property 3: Upload Response Row Count Matches Actual Rows**
    - **Validates: Requirements 1.6**
    - Use `@given` with a strategy generating valid CSVs with a random number of data rows (1–200)
    - Post to `POST /upload`; assert `response.json()["row_count"] == generated_row_count`
    - Tag: `# Feature: finance-narrate-ai, Property 3: Upload Response Row Count Matches Actual Rows`

  - [x] 6.8 Implement `POST /analyze/{file_id}`
    - Look up `file_id` in `upload_store`; raise 404 if absent
    - Call `build_metrics(path)`; store result in `metrics_store`; return `MetricsResult`
    - Add docstring
    - _Requirements: 2.1, 2.2, 2.8_

  - [x] 6.9 Implement `POST /generate-narrative/{file_id}`
    - Look up `file_id` in `metrics_store`; raise 400 if absent (analysis not run)
    - Call `generate_narrative(metrics)`; store result in `narrative_store`; return `NarrativeResult`
    - Add docstring
    - _Requirements: 3.1, 3.3_

  - [x] 6.10 Implement `GET /report/{file_id}`
    - Look up `file_id` in `upload_store`; raise 404 if absent
    - Check both `metrics_store` and `narrative_store`; raise 400 listing missing components if either absent
    - Return `ReportResult` combining both
    - Add docstring
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ]* 6.11 Write property test for `GET /report/{file_id}` (Property 10)
    - **Property 10: Report Combines Metrics and Narrative Unchanged**
    - **Validates: Requirements 4.2**
    - Use `@given` with strategies generating random `MetricsResult` and `NarrativeResult` objects
    - Inject them directly into `metrics_store` and `narrative_store`; call `GET /report/{file_id}`; assert `response.json()["metrics"] == metrics.model_dump()` and `response.json()["narrative"] == narrative.model_dump()`
    - Tag: `# Feature: finance-narrate-ai, Property 10: Report Combines Metrics and Narrative Unchanged`

- [x] 7. Checkpoint — all backend tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [x] 8. Frontend dashboard (`frontend/`)
  - [x] 8.1 Create `frontend/index.html` layout
    - Add drag-and-drop upload zone, file input, status area, preview table container, two `<canvas>` elements for Chart.js charts, anomaly cards container, executive narrative panel, "Copy Report" button, "Download as .txt" button, and loading spinner
    - Link `styles.css` and `dashboard.js`; include Chart.js via CDN
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9_

  - [x] 8.2 Create `frontend/styles.css`
    - Style the upload zone (dashed border, hover highlight), preview table, chart containers, anomaly alert cards (red left border / background tint), narrative panel, loading spinner overlay, and error message banner
    - _Requirements: 5.5, 5.7_

  - [x] 8.3 Implement file upload flow in `dashboard.js`
    - Handle drag-and-drop and `<input type="file">` change events
    - `POST /upload` with `FormData`; on success display preview table (first 10 rows parsed client-side with FileReader + PapaParse or manual CSV split); store `file_id`
    - Show spinner during request; display error banner on non-2xx response
    - _Requirements: 5.1, 5.2, 5.7, 5.10_

  - [x] 8.4 Implement analysis and narrative trigger in `dashboard.js`
    - After upload success, automatically call `POST /analyze/{file_id}` then `POST /generate-narrative/{file_id}` in sequence
    - Show spinner between calls; handle errors at each step
    - _Requirements: 5.7, 5.10_

  - [x] 8.5 Implement Chart.js revenue trend line chart in `dashboard.js`
    - On analysis success, render a line chart on the first canvas using `monthly_revenue` data (x = month labels, y = totals)
    - _Requirements: 5.3_

  - [x] 8.6 Implement Chart.js expense breakdown bar chart in `dashboard.js`
    - On analysis success, render a bar chart on the second canvas using `top_categories` data (x = category names, y = totals)
    - _Requirements: 5.4_

  - [x] 8.7 Implement anomaly alert cards in `dashboard.js`
    - For each entry in `expense_anomalies` and `revenue_dips`, create a card element with red highlight and inject into the anomaly container
    - _Requirements: 5.5_

  - [x] 8.8 Implement executive narrative panel in `dashboard.js`
    - On narrative success, populate the narrative panel with all four sections: executive summary, revenue trends (bulleted list), anomalies list, and recommendations list
    - _Requirements: 5.6_

  - [x] 8.9 Implement "Copy Report" and "Download as .txt" buttons in `dashboard.js`
    - "Copy Report": call `navigator.clipboard.writeText` with the full narrative text
    - "Download as .txt": create a `Blob`, generate an object URL, trigger an `<a>` click to download `report_{file_id}.txt`
    - _Requirements: 5.8, 5.9_

- [x] 9. Final checkpoint — full pipeline verified
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Property tests use the Hypothesis library; run with `pytest backend/tests/` from the project root
- Each property test is tagged with `# Feature: finance-narrate-ai, Property N: <title>` for traceability
- The Gemini API call in `call_gemini` should be mocked in all tests that exercise endpoints beyond `POST /generate-narrative`
- The frontend has no build step — open `frontend/index.html` directly or serve with `python -m http.server` from the `frontend/` directory
- Start the backend with: `uvicorn main:app --reload` from the `backend/` directory
