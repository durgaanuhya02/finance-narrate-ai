# Design Document

## Overview

FinanceNarrate AI is a web application that accepts structured financial data uploads and produces AI-generated executive reports. A financial analyst uploads a CSV or Excel file through a browser dashboard; the backend validates the file, runs a Pandas-based metrics pipeline, calls the Google Gemini API to generate a four-section board-ready narrative, and returns the combined report. The frontend renders interactive Chart.js visualisations, anomaly alert cards, and the narrative panel, and lets the analyst copy or download the report.

The system is intentionally simple: no database, no authentication, no message queue. State is held on the local filesystem (uploaded files) and in-process Python dicts (computed metrics and narratives keyed by `file_id`). This keeps the architecture easy to run locally while still demonstrating the full AI-augmented analytics workflow.

---

## Architecture

```mermaid
graph TD
    A[Browser Dashboard<br/>HTML / JS / Chart.js] -->|POST /upload| B[FastAPI Backend]
    A -->|POST /analyze/{file_id}| B
    A -->|POST /generate-narrative/{file_id}| B
    A -->|GET /report/{file_id}| B
    A -->|GET /health| B

    B --> C[Validator<br/>column & format check]
    B --> D[File_Store<br/>local filesystem]
    B --> E[Processor<br/>Pandas pipeline]
    B --> F[LLM_Client<br/>Gemini API]

    E -->|Metrics JSON| B
    F -->|Narrative JSON| B

    F -->|HTTPS| G[Google Gemini API<br/>gemini-1.5-flash]
```

### Request Flow

1. **Upload** — `POST /upload` → Validator checks format & columns → File_Store saves file → returns `file_id`.
2. **Analyze** — `POST /analyze/{file_id}` → Processor loads file, computes Metrics → stored in-process → returns Metrics JSON.
3. **Narrate** — `POST /generate-narrative/{file_id}` → LLM_Client fetches stored Metrics, calls Gemini → parses Narrative JSON → stored in-process → returns Narrative JSON.
4. **Report** — `GET /report/{file_id}` → merges stored Metrics + Narrative → returns combined Report JSON.

---

## Components and Interfaces

### FastAPI Application (`main.py`)

Wires together all components and exposes the HTTP API.

| Endpoint | Method | Description |
|---|---|---|
| `/health` | GET | Liveness check |
| `/upload` | POST | Accept multipart file, validate, store |
| `/analyze/{file_id}` | POST | Run Processor pipeline |
| `/generate-narrative/{file_id}` | POST | Run LLM_Client narrative generation |
| `/report/{file_id}` | GET | Return combined Metrics + Narrative |

In-process state stores (Python dicts, module-level):
- `metrics_store: dict[str, MetricsResult]` — keyed by `file_id`
- `narrative_store: dict[str, NarrativeResult]` — keyed by `file_id`
- `upload_store: dict[str, UploadInfo]` — keyed by `file_id`

### Validator (inline in `main.py` / `processor.py`)

- Checks file extension is `.csv`, `.xlsx`, or `.xls`; raises HTTP 415 otherwise.
- Reads column headers; checks for `Date`, `Revenue`, `Expenses`, `Category`; raises HTTP 422 with missing column list otherwise.

### File_Store (inline in `main.py`)

- Saves uploaded bytes to `uploads/{file_id}/{filename}` on the local filesystem.
- Provides a `get_file_path(file_id) -> Path` helper used by the Processor.

### Processor (`processor.py`)

Pure-function pipeline — takes a `Path`, returns a `MetricsResult`.

Key functions:

```
load_dataframe(path: Path) -> pd.DataFrame
compute_monthly_revenue(df: pd.DataFrame) -> dict[str, float]
compute_mom_growth(monthly: dict[str, float]) -> dict[str, float]
compute_top_categories(df: pd.DataFrame, n: int = 3) -> list[dict]
detect_expense_anomalies(df: pd.DataFrame) -> list[dict]
detect_revenue_dips(monthly: dict[str, float]) -> list[dict]
build_metrics(path: Path) -> MetricsResult
```

### LLM_Client (`llm_client.py`)

Wraps the `google-generativeai` SDK.

Key functions:

```
build_prompt(metrics: MetricsResult) -> str
call_gemini(prompt: str) -> str          # raw text response
parse_narrative(raw: str) -> NarrativeResult
generate_narrative(metrics: MetricsResult) -> NarrativeResult
```

The prompt instructs Gemini to respond with a JSON object containing exactly the four required keys. `parse_narrative` calls `json.loads` on the response; if that raises, the function raises an HTTP 502.

### Frontend (`frontend/`)

Single-page application with no build step.

| File | Responsibility |
|---|---|
| `index.html` | Layout, drag-and-drop zone, chart canvases, narrative panel |
| `dashboard.js` | API calls (`fetch`), Chart.js rendering, anomaly cards, clipboard/download |
| `styles.css` | Visual styling, anomaly red highlight, loading spinner |

---

## Data Models

All models are defined in `models.py` using Pydantic v2.

```python
class UploadResponse(BaseModel):
    file_id: str
    filename: str
    row_count: int
    status: str  # "uploaded"

class MonthlyRevenue(BaseModel):
    month: str   # "YYYY-MM"
    total: float

class MoMGrowth(BaseModel):
    month: str
    growth_pct: float  # e.g. 5.2 means +5.2%

class TopCategory(BaseModel):
    category: str
    total_expenses: float

class ExpenseAnomaly(BaseModel):
    row_index: int
    date: str
    category: str
    expenses: float
    z_score: float

class RevenueDip(BaseModel):
    month: str
    revenue: float
    previous_month: str
    previous_revenue: float
    drop_pct: float  # negative value, e.g. -18.3

class MetricsResult(BaseModel):
    file_id: str
    monthly_revenue: list[MonthlyRevenue]
    mom_growth: list[MoMGrowth]
    top_categories: list[TopCategory]
    expense_anomalies: list[ExpenseAnomaly]
    revenue_dips: list[RevenueDip]

class NarrativeResult(BaseModel):
    file_id: str
    executive_summary: str
    revenue_trends: list[str]   # bullet points
    anomalies: list[str]        # one entry per flagged anomaly
    recommendations: list[str]  # 2–3 action items

class ReportResult(BaseModel):
    file_id: str
    metrics: MetricsResult
    narrative: NarrativeResult
```

### Prompt Schema Contract

The Gemini prompt instructs the model to return JSON matching:

```json
{
  "executive_summary": "<3-4 sentence formal summary>",
  "revenue_trends": ["<bullet 1>", "<bullet 2>", "..."],
  "anomalies": ["<anomaly description 1>", "..."],
  "recommendations": ["<action 1>", "<action 2>"]
}
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Column Validation Identifies All Missing Columns

*For any* uploaded file whose column headers are a subset of the required columns (`Date`, `Revenue`, `Expenses`, `Category`), the Validator SHALL return an error response that identifies exactly the set of columns that are absent — no more, no fewer.

**Validates: Requirements 1.2, 1.3**

---

### Property 2: Invalid File Type Always Returns 415

*For any* uploaded file whose extension is not `.csv`, `.xlsx`, or `.xls`, the API SHALL return an HTTP 415 response regardless of the file's content.

**Validates: Requirements 1.4**

---

### Property 3: Upload Response Row Count Matches Actual Rows

*For any* valid CSV or Excel file with N data rows, the `row_count` field in the upload response SHALL equal N.

**Validates: Requirements 1.6**

---

### Property 4: Monthly Revenue Aggregation Is Lossless

*For any* valid DataFrame, the sum of all `total` values in the `monthly_revenue` list of the Metrics object SHALL equal the sum of the `Revenue` column in the original DataFrame (within floating-point tolerance).

**Validates: Requirements 2.3, 2.9**

---

### Property 5: Month-over-Month Growth Is Mathematically Correct

*For any* consecutive pair of months (previous month P, current month C) in the Metrics object, the `growth_pct` value for month C SHALL equal `(C.total - P.total) / P.total * 100` (within floating-point tolerance).

**Validates: Requirements 2.4**

---

### Property 6: Top Categories Are the Highest-Total Categories

*For any* valid DataFrame, every category returned in `top_categories` SHALL have a summed expense total greater than or equal to the summed expense total of every category NOT in `top_categories`.

**Validates: Requirements 2.5**

---

### Property 7: Expense Anomaly Detection Respects the 2-Std Threshold

*For any* valid DataFrame, every row flagged as an expense anomaly SHALL have an `Expenses` value strictly greater than `mean(Expenses) + 2 * std(Expenses)`, and every non-flagged row SHALL have an `Expenses` value less than or equal to that threshold.

**Validates: Requirements 2.6**

---

### Property 8: Revenue Dip Detection Respects the 15% Threshold

*For any* sequence of monthly revenue totals, every month flagged as a revenue dip SHALL have a `drop_pct` strictly less than `-15.0`, and every non-flagged month SHALL have a `drop_pct` greater than or equal to `-15.0`.

**Validates: Requirements 2.7**

---

### Property 9: Narrative Parse Round-Trip

*For any* JSON string that is a valid serialisation of a `NarrativeResult`-shaped object (containing `executive_summary`, `revenue_trends`, `anomalies`, and `recommendations`), calling `parse_narrative` SHALL return a `NarrativeResult` whose fields are equal to the original object's fields.

**Validates: Requirements 3.5**

---

### Property 10: Report Combines Metrics and Narrative Unchanged

*For any* `file_id` for which both a `MetricsResult` and a `NarrativeResult` have been stored, the `GET /report/{file_id}` response SHALL contain a `ReportResult` whose `metrics` field equals the stored `MetricsResult` and whose `narrative` field equals the stored `NarrativeResult`.

**Validates: Requirements 4.2**

---

## Error Handling

| Scenario | Component | HTTP Status | Response |
|---|---|---|---|
| Unsupported file type | Validator | 415 | `{"detail": "Unsupported file type. Upload CSV or Excel (.xlsx/.xls)."}` |
| Missing required columns | Validator | 422 | `{"detail": "Missing required columns: [Date, Revenue]"}` |
| `file_id` not found (analyze) | main.py | 404 | `{"detail": "File not found for file_id: <id>"}` |
| `file_id` not found (report) | main.py | 404 | `{"detail": "File not found for file_id: <id>"}` |
| Analysis not run before narrative | main.py | 400 | `{"detail": "Analysis must be run before narrative generation."}` |
| Metrics or Narrative missing (report) | main.py | 400 | `{"detail": "Report incomplete. Missing: metrics"}` |
| Gemini response not valid JSON | LLM_Client | 502 | `{"detail": "Malformed response from Gemini API."}` |
| `GEMINI_API_KEY` not set | LLM_Client | startup error | `RuntimeError: GEMINI_API_KEY is not set.` |

All FastAPI HTTP errors are raised using `HTTPException` with the appropriate `status_code` and `detail` string. The frontend catches non-2xx responses and displays the `detail` field as a human-readable error message.

---

## Testing Strategy

### Unit Tests (pytest)

Focus on specific examples, edge cases, and error conditions:

- **Validator**: test each missing-column combination, unsupported extensions, valid files.
- **Processor**: test `compute_monthly_revenue`, `compute_mom_growth`, `compute_top_categories`, `detect_expense_anomalies`, `detect_revenue_dips` with known DataFrames.
- **LLM_Client**: test `build_prompt` contains all four section instructions; test `parse_narrative` with valid and invalid JSON; test startup error when `GEMINI_API_KEY` is absent.
- **API endpoints**: test 404/400/415/422 error paths; test happy-path responses with mocked Processor and LLM_Client.

### Property-Based Tests (Hypothesis)

Use the [Hypothesis](https://hypothesis.readthedocs.io/) library for Python. Each property test runs a minimum of 100 iterations.

Each test is tagged with a comment in the format:
`# Feature: finance-narrate-ai, Property <N>: <property_text>`

| Property | Test Description |
|---|---|
| Property 1 | Generate DataFrames with random column subsets; verify validator error lists exactly the missing columns |
| Property 2 | Generate random non-CSV/Excel extensions; verify 415 response |
| Property 3 | Generate CSVs with random row counts; verify `row_count` in response matches |
| Property 4 | Generate DataFrames with random dates/revenues; verify `sum(monthly_revenue) == sum(Revenue)` |
| Property 5 | Generate monthly revenue sequences; verify each MoM growth value equals `(C - P) / P * 100` |
| Property 6 | Generate DataFrames with random categories/expenses; verify top-N categories have highest totals |
| Property 7 | Generate DataFrames with random expense distributions; verify anomaly flags match 2-std threshold |
| Property 8 | Generate monthly revenue sequences; verify dip flags match 15% threshold |
| Property 9 | Generate random NarrativeResult dicts; serialize to JSON; verify `parse_narrative` round-trip |
| Property 10 | Generate random MetricsResult + NarrativeResult pairs; store them; verify report combines both unchanged |

### Integration Tests

- Upload a real sample CSV → analyze → generate-narrative (with mocked Gemini) → GET /report: verify full pipeline end-to-end.
- Verify CORS headers are present on responses.

### Frontend Testing

Manual testing and/or Playwright browser automation for:
- Drag-and-drop upload, preview table, chart rendering, anomaly cards, narrative panel, copy/download buttons, loading spinner, error display.
