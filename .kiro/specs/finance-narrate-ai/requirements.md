# Requirements Document

## Introduction

FinanceNarrate AI is an Executive Financial Insight Engine that enables financial analysts to upload structured financial data (CSV or Excel) and receive AI-generated executive summaries, revenue trend analyses, expense anomaly reports, and board-ready narrative recommendations. The system processes uploaded data using a Pandas pipeline, computes key financial metrics, and leverages the Google Gemini API (gemini-1.5-flash) to produce formal, structured narratives. A browser-based dashboard provides interactive charts, anomaly alerts, and report export capabilities.

## Glossary

- **System**: The FinanceNarrate AI application as a whole
- **API**: The FastAPI backend service exposing HTTP endpoints
- **Processor**: The Pandas-based data processing pipeline component
- **LLM_Client**: The Google Gemini API integration component responsible for narrative generation
- **Dashboard**: The browser-based HTML/JS frontend interface
- **File_Store**: The local filesystem component that temporarily stores uploaded files
- **Validator**: The component that checks uploaded file structure and column compliance
- **Metrics**: The structured JSON object produced by the Processor containing computed financial statistics
- **Narrative**: The structured JSON object produced by the LLM_Client containing the four-section board-ready report
- **Report**: The combined output of Metrics and Narrative for a given file_id
- **file_id**: A unique identifier assigned to each uploaded file upon successful upload
- **Anomaly**: A data point where expenses exceed 2 standard deviations from the mean, or revenue drops more than 15% compared to the previous month
- **DataFrame**: An in-memory tabular data structure produced by Pandas from the uploaded file

---

## Requirements

### Requirement 1: File Upload and Validation

**User Story:** As a financial analyst, I want to upload a CSV or Excel file containing financial data, so that the system can process and analyze it.

#### Acceptance Criteria

1. THE API SHALL expose a `POST /upload` endpoint that accepts multipart file uploads in CSV or Excel (`.xlsx`, `.xls`) format.
2. WHEN a file is uploaded, THE Validator SHALL check that the file contains the columns: `Date`, `Revenue`, `Expenses`, and `Category`.
3. IF a file is missing one or more required columns, THEN THE API SHALL return an HTTP 422 response with a descriptive error message identifying the missing columns.
4. IF a file is not in CSV or Excel format, THEN THE API SHALL return an HTTP 415 response with a message indicating the unsupported file type.
5. WHEN a valid file is uploaded, THE File_Store SHALL save the file to the local filesystem under a path derived from a newly generated `file_id`.
6. WHEN a valid file is successfully stored, THE API SHALL return a JSON response containing `file_id`, `filename`, `row_count`, and `status` fields.
7. THE API SHALL expose a `GET /health` endpoint that returns HTTP 200 with a JSON body indicating the service is operational.

---

### Requirement 2: Data Processing Pipeline

**User Story:** As a financial analyst, I want the system to compute key financial metrics from my uploaded data, so that I can understand revenue trends and expense anomalies.

#### Acceptance Criteria

1. WHEN the `POST /analyze/{file_id}` endpoint is called, THE Processor SHALL load the file associated with `file_id` from the File_Store and parse it into a DataFrame.
2. IF the file associated with `file_id` does not exist in the File_Store, THEN THE API SHALL return an HTTP 404 response with a descriptive error message.
3. THE Processor SHALL compute the total revenue grouped by calendar month from the `Date` and `Revenue` columns.
4. THE Processor SHALL calculate the month-over-month revenue growth percentage for each month relative to the immediately preceding month.
5. THE Processor SHALL identify the top 3 expense categories by summed total from the `Category` and `Expenses` columns.
6. THE Processor SHALL flag each row as an expense anomaly WHEN the `Expenses` value for that row exceeds 2 standard deviations above the mean of all `Expenses` values in the DataFrame.
7. THE Processor SHALL flag each month as a revenue dip WHEN the monthly revenue total is more than 15% lower than the monthly revenue total of the immediately preceding month.
8. WHEN processing completes successfully, THE API SHALL return a structured JSON Metrics object containing: monthly revenue totals, month-over-month growth percentages, top 3 expense categories, flagged expense anomalies, and flagged revenue dips.
9. FOR ALL valid DataFrames, the sum of all monthly revenue totals in the Metrics object SHALL equal the sum of the `Revenue` column in the original DataFrame (round-trip invariant).

---

### Requirement 3: LLM Narrative Generation

**User Story:** As a financial analyst, I want the system to generate a board-ready narrative from the computed metrics, so that I can present findings to executives without manual writing.

#### Acceptance Criteria

1. THE API SHALL expose a `POST /generate-narrative/{file_id}` endpoint that triggers narrative generation for the specified `file_id`.
2. WHEN the endpoint is called, THE LLM_Client SHALL retrieve the Metrics for the given `file_id` and send them as context to the Gemini API using the `gemini-1.5-flash` model.
3. IF the file associated with `file_id` has not been analyzed yet, THEN THE API SHALL return an HTTP 400 response indicating that analysis must be run before narrative generation.
4. THE LLM_Client SHALL instruct Gemini to produce a response containing exactly four sections: an Executive Summary of 3–4 sentences in formal tone, a Revenue Trend Analysis as bullet points, an Expense Anomaly Report explaining each flagged anomaly, and Strategic Recommendations consisting of 2–3 action items.
5. WHEN Gemini returns a response, THE LLM_Client SHALL parse the response as JSON into a Narrative object with fields: `executive_summary`, `revenue_trends`, `anomalies`, and `recommendations`.
6. IF the Gemini response cannot be parsed as valid JSON, THEN THE LLM_Client SHALL return an HTTP 502 response with an error message indicating a malformed upstream response.
7. WHEN narrative generation completes successfully, THE API SHALL return the Narrative object as a JSON response.

---

### Requirement 4: Combined Report Retrieval

**User Story:** As a financial analyst, I want to retrieve the full combined report for an uploaded file, so that I can access both metrics and narrative in a single request.

#### Acceptance Criteria

1. THE API SHALL expose a `GET /report/{file_id}` endpoint.
2. WHEN the endpoint is called for a `file_id` that has both Metrics and a Narrative available, THE API SHALL return a JSON Report object combining both the Metrics and Narrative objects.
3. IF the `file_id` does not exist, THEN THE API SHALL return an HTTP 404 response with a descriptive error message.
4. IF Metrics or Narrative have not yet been generated for the given `file_id`, THEN THE API SHALL return an HTTP 400 response indicating which components are not yet available.

---

### Requirement 5: Frontend Dashboard

**User Story:** As a financial analyst, I want a browser-based dashboard, so that I can upload files, view charts, review anomalies, and read the generated narrative without using API tools directly.

#### Acceptance Criteria

1. THE Dashboard SHALL provide a drag-and-drop file upload area that accepts CSV and Excel files and submits them to the `POST /upload` endpoint.
2. WHEN a file is successfully uploaded, THE Dashboard SHALL display a preview table showing the first 10 rows of the uploaded data.
3. WHEN analysis results are available, THE Dashboard SHALL render a revenue trend line chart using Chart.js displaying monthly revenue totals over time.
4. WHEN analysis results are available, THE Dashboard SHALL render an expense breakdown bar chart using Chart.js displaying the top expense categories by total.
5. WHEN anomalies are present in the Metrics, THE Dashboard SHALL display each anomaly as a visually distinct alert card with red highlighting.
6. WHEN the Narrative is available, THE Dashboard SHALL display all four narrative sections in a dedicated executive narrative panel.
7. WHILE an API call is in progress, THE Dashboard SHALL display a loading spinner and disable further user interaction with the triggering control.
8. THE Dashboard SHALL provide a "Copy Report" button that copies the full Narrative text to the system clipboard.
9. THE Dashboard SHALL provide a "Download as .txt" button that triggers a browser download of the full Report as a plain-text file.
10. IF an API call returns an error response, THEN THE Dashboard SHALL display a human-readable error message to the user without crashing or losing previously loaded data.

---

### Requirement 6: Cross-Cutting Concerns

**User Story:** As a developer, I want the system to be robust and maintainable, so that it can be reliably operated and extended.

#### Acceptance Criteria

1. THE API SHALL include CORS middleware configured to allow requests from the frontend origin.
2. THE System SHALL use environment variables loaded from a `.env` file for all secrets, including `GEMINI_API_KEY`.
3. IF `GEMINI_API_KEY` is not set in the environment, THEN THE LLM_Client SHALL raise a configuration error at startup before accepting any requests.
4. THE System SHALL include a `requirements.txt` file listing all Python dependencies: `fastapi`, `uvicorn`, `pandas`, `openpyxl`, `python-multipart`, `google-generativeai`, `python-dotenv`, and `pydantic`.
5. ALL Python functions and classes in the backend SHALL include docstrings describing their purpose, parameters, and return values.
