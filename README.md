CleanIQ
CleanIQ is a functional, explainable data-quality workspace. It accepts CSV, XLSX, and JSON datasets, profiles them with Pandas, recommends conservative cleaning operations, previews the impact, and exports a cleaned dataset plus a cleaning report.
What is implemented
The application includes real dataset upload and parsing, a built-in messy sample dataset, automatic quality scoring, missing-value/duplicate/type/date/category/outlier/constant-column detection, issue charts, deterministic explainable recommendations, review-before-apply transformations, undo/reset history, CSV/XLSX exports, and an HTML report suitable for printing to PDF from the browser.
Uploaded datasets are held in memory under a signed session identifier. Nothing is written to a database or retained as a permanent user file. The current process is intentionally session-scoped for privacy and for this first product iteration.
Run locally
Python 3.11+ is recommended.
Bash

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python app.py
Open http://localhost:3000. The Flask server listens on 0.0.0.0 and honors the PORT environment variable.
For a production-style local process:
Bash

gunicorn --bind 0.0.0.0:3000 --workers 1 app:app
Workflow
Load the sample dataset or upload a CSV, XLSX, or JSON file. Review the quality score, chart panels, and table preview. Open Recommendations, inspect each rationale and warning, select or deselect operations, and use Preview selected. The confirmation view shows projected score, row, and column changes before applying the run. Pipeline includes additional opt-in model-prep operations, while History records each applied batch. Export results downloads the current CSV, XLSX, or HTML report.
API surface
GET /api/state — current session dataset and analysis state.
POST /api/sample — load the included sample dataset.
POST /api/upload — multipart upload with file field.
POST /api/preview — preview { "operations": ["operation_id"] } without mutation.
POST /api/apply — apply an ordered operation list and append a history entry.
POST /api/undo — rebuild the working DataFrame without the most recent run.
POST /api/reset — restore the original uploaded DataFrame.
GET /api/export/csv, /api/export/xlsx, /api/export/report — current dataset/report downloads.
GET /api/report — printable HTML report.
GET /health — unauthenticated service health response.
Supported operation IDs include remove_duplicates, fix_numeric_types, fill_missing_numeric_median, fill_missing_categorical_mode, fill_missing_forward, drop_missing_rows, standardize_text, normalize_dates, clip_outliers_iqr, remove_constant_columns, remove_identifier_columns, encode_categorical, and scale_numeric.
Test the workflow
The sample data is designed to trigger missing values, duplicate rows, mixed category casing, an invalid date, a malformed numeric value, and an extreme order count.
Bash

python -m py_compile app.py cleaning_engine.py
python - <<'PY'
from app import app

client = app.test_client()
assert client.get('/health').status_code == 200
sample = client.post('/api/sample')
assert sample.status_code == 200
state = sample.get_json()
assert state['has_dataset'] is True
assert state['analysis']['missing_cells'] > 0
assert state['analysis']['duplicate_rows'] > 0
ops = [item['id'] for item in state['recommendations'] if item['selected']]
preview = client.post('/api/preview', json={'operations': ops})
assert preview.status_code == 200
applied = client.post('/api/apply', json={'operations': ops})
assert applied.status_code == 200
assert applied.get_json()['history']
assert client.get('/api/export/csv').status_code == 200
assert client.get('/api/export/xlsx').status_code == 200
assert client.get('/api/export/report').status_code == 200
print('CleanIQ smoke workflow passed')
PY
