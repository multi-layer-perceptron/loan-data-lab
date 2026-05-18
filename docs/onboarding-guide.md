# Onboarding Guide — Contoso Bank Loan Data Modernization Lab

Welcome to the Contoso Bank Loan Data Modernization Lab. This guide walks you through setting up your local development environment and running every component of the pipeline.

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Git | 2.40+ | `git --version` |
| .NET SDK | 8.0+ | `dotnet --version` |
| Python | 3.11+ | `python --version` |
| VS Code | Latest | With GitHub Copilot extension |
| Azure CLI | 2.57+ | For infra deployment (`az --version`) |
| Snowflake account | Any | Only required for live Snowpark tests |

---

## 1. Clone the Repository

```bash
git clone https://github.com/auto-cloud-arc/loan-data-lab.git
cd loan-data-lab
```

---

## 2. Running the C# Data Cleaner

The C# cleaner normalizes and validates raw loan application CSV extracts.

### Build and test

```bash
cd src/data-cleaner-csharp
dotnet restore ContosoLoanCleaner.sln
dotnet build ContosoLoanCleaner.sln --configuration Release
dotnet test ContosoLoanCleaner.sln --configuration Release --verbosity normal
```

### Run against sample data

```bash
dotnet run --project ContosoLoanCleaner -- \
  ../../sample-data/raw/loan_applications_raw.csv \
  ../../sample-data/cleaned/loan_applications_cleaned.csv \
  ../../sample-data/expected/dq_exception_report_output.json
```

Expected output:
```
[08:30:01 INF] Contoso Bank Loan Data Cleaner starting.
[08:30:01 INF] Input:  ../../sample-data/raw/loan_applications_raw.csv
[08:30:01 INF] Output: ../../sample-data/cleaned/loan_applications_cleaned.csv
[08:30:01 WRN] Application APP-005 has 1 validation failure(s).
[08:30:01 WRN] Application APP-006 has 1 validation failure(s).
[08:30:01 WRN] Application APP-007 has 1 validation failure(s).
[08:30:01 WRN] Application APP-010 has 1 validation failure(s).
[08:30:01 INF] Cleaning complete. 6/10 records passed. Report: ...
```

### Understanding the output

- **Cleaned CSV**: Records that pass all validation rules
- **Exception JSON**: Records that failed, with redacted PII and the rule that triggered

---

## 3. Running the QA Validator (Python)

The QA Validator applies business rules to the cleaned data, performs an optional row-count reconciliation check, and writes both machine-readable and human-readable DQ reports.

### Install dependencies

```bash
pip install pandas pytest
# Or use the test requirements which include both:
pip install -r src/qa-validator/tests/requirements-test.txt
```

### Run tests

```bash
pytest src/qa-validator/tests/ -v
```

Current test layout:

- `src/qa-validator/tests/test_business_rules.py` - loan amount and secured-loan collateral checks
- `src/qa-validator/tests/test_date_validation_rules.py` - future-date validation
- `src/qa-validator/tests/test_null_check_rules.py` - required business key validation
- `src/qa-validator/tests/test_reconciliation_rules.py` - row-count reconciliation logic
- `src/qa-validator/tests/test_report_generator.py` - JSON and Markdown report output

### Run the full validation against cleaned sample data

```bash
python src/qa-validator/runners/run_validations.py \
  --input sample-data/cleaned/loan_applications_cleaned.csv \
  --report-dir src/qa-validator/reports
```

The runner loads the cleaned CSV, executes the registered rule set in `run_all_validations()`, computes reconciliation status, and then writes two timestamped files to the selected report directory:

- `qa_report_YYYYMMDD_HHMMSS.json` - structured output for automation and artifact publishing
- `qa_report_YYYYMMDD_HHMMSS.md` - Markdown summary for quick review

The Markdown and JSON reports both include:

- input file path
- total records and total failures
- pass rate
- failure details
- reconciliation summary when reconciliation arguments are provided or defaulted

By default, the runner uses the input row count as both the source and target count, which produces a passing reconciliation summary for local sample runs. To test reconciliation failures explicitly, pass different counts:

```bash
python src/qa-validator/runners/run_validations.py \
  --input sample-data/cleaned/loan_applications_cleaned.csv \
  --report-dir src/qa-validator/reports \
  --source-count 10 \
  --target-count 8 \
  --reconciliation-table loan_application_curated \
  --reconciliation-tolerance 0.01
```

Exit-code behavior:

- returns `1` when any validation failure has severity `critical`
- returns `1` when reconciliation fails
- returns `0` for warning-only failures or a fully clean run

### Adding a new validation rule

1. Create a function in `src/qa-validator/rules/` that accepts a `pd.DataFrame` and returns `list[ValidationFailure]`
2. Write or update the matching pytest coverage in `src/qa-validator/tests/`
3. Register the rule in `src/qa-validator/runners/run_validations.py` under `run_all_validations()`

---

## 4. Running Snowpark Tests Locally (No Snowflake Required)

The Snowpark transform tests test pure-Python helper functions without connecting to Snowflake.

### Install dependencies

```bash
pip install -r src/snowpark/tests/requirements-test.txt
```

### Run tests

```bash
pytest src/snowpark/tests/ -v
```

These tests cover:
- `standardize_branch_code()` — branch code normalization
- `compute_risk_tier()` — LTV-based risk tier computation
- `reconcile_counts()` — source-to-target count comparison

### Running against a live Snowflake connection

Set environment variables:
```bash
export SNOWFLAKE_ACCOUNT=your-account
export SNOWFLAKE_USER=your-user
export SNOWFLAKE_PASSWORD=your-password
export SNOWFLAKE_WAREHOUSE=CONTOSO_TRANSFORM_WH
export SNOWFLAKE_DATABASE=CONTOSO_LOAN_DB
export SNOWFLAKE_SCHEMA=CURATED_ZONE
export SNOWFLAKE_ROLE=CONTOSO_DATA_ENGINEER
```

Then run a transform directly:
```python
from snowflake.snowpark import Session
from src.snowpark.transforms.borrower_360_transform import run

session = Session.builder.configs({
    "account": os.environ["SNOWFLAKE_ACCOUNT"],
    "user": os.environ["SNOWFLAKE_USER"],
    "password": os.environ["SNOWFLAKE_PASSWORD"],
    "warehouse": os.environ["SNOWFLAKE_WAREHOUSE"],
    "database": os.environ["SNOWFLAKE_DATABASE"],
    "role": os.environ["SNOWFLAKE_ROLE"],
}).create()

run(session)
```

---

## 5. Running the Web UI for Manual Testing

The Streamlit web UI lets you run the full cleaner and QA validator pipeline from a browser without touching the command line.

### Install dependencies

```bash
pip install -r src/web-ui/requirements.txt
```

### Start the app

```bash
streamlit run src/web-ui/app.py
```

Streamlit will print a local URL (typically `http://localhost:8501`). Open it in your browser.

### What you can do in the UI

- **Sample data** — select a bundled CSV from `sample-data/raw/` to run the pipeline without uploading anything
- **Upload CSV** — upload your own raw loan data CSV for an ad-hoc run
- **Override reconciliation counts** — enable the checkbox to supply custom source and target row counts and test reconciliation failure scenarios
- **Reconciliation tolerance** — adjust the slider to change the acceptable count-mismatch threshold
- **Run pipeline** — click the button to invoke the C# cleaner and the QA validator end to end
- **Download outputs** — download the cleaned CSV, JSON QA report, and Markdown QA summary directly from the results panel

> **Note:** The C# cleaner binary must be buildable via `dotnet build` (see section 2). The web UI calls it as a subprocess at runtime.

---

## 6. Working with Azure SQL (Optional)

To run the SQL scripts against a local or cloud Azure SQL instance:

```bash
# Using sqlcmd with Microsoft Entra authentication
sqlcmd -S your-server.database.windows.net -d loan_db -G \
  -i src/sqlserver/schema/01_create_tables.sql
sqlcmd -S your-server.database.windows.net -d loan_db -G \
  -i src/sqlserver/seed/02_seed_loan_application_raw.sql
```

Or use the PowerShell helper after `az login`:

```powershell
pwsh ./src/sqlserver/scripts/apply_sql_files.ps1 `
  -ServerInstance your-server.database.windows.net `
  -Database loan_db `
  -SqlFiles 'src/sqlserver/schema/01_create_tables.sql','src/sqlserver/seed/05_seed_branch_reference.sql'
```

Or use Azure Data Studio / SSMS and sign in with Microsoft Entra authentication.

---

## 7. Deploying Infrastructure (Azure)

Infrastructure is managed with Azure Bicep.

```bash
az login
az group create --name rg-contoso-loan-dev --location centralus
az deployment group create \
  --resource-group rg-contoso-loan-dev \
  --template-file infra/bicep/main.bicep \
  --parameters infra/parameters/dev.parameters.json \
  --parameters adminObjectId=<entra-object-id> adminPrincipalName=<entra-login>
```

Or trigger the GitHub Actions workflow manually:
- Go to **Actions** → **Deploy Azure SQL & Config** → **Run workflow** → select `dev`

---

## 8. CI/CD Overview

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| `ci.yml` | Push/PR to `main` or `feature/**` | Builds C# + runs all tests |
| `validate-data-quality.yml` | Push/PR changing `sample-data/` or `src/qa-validator/` | Runs QA validator, uploads report artifact |
| `deploy-azure-sql-and-config.yml` | Manual (`workflow_dispatch`) | Deploys Bicep infra + SQL schema |

### Required GitHub Secrets

| Secret | Used by |
|--------|---------|
| `AZURE_CREDENTIALS` | `deploy-azure-sql-and-config.yml` |
| `AZURE_RG` | `deploy-azure-sql-and-config.yml` |
| `AZURE_SQL_ADMIN_OBJECT_ID` | `deploy-azure-sql-and-config.yml` |
| `AZURE_SQL_ADMIN_LOGIN` | `deploy-azure-sql-and-config.yml` |
| `AZURE_SQL_ADMIN_PRINCIPAL_TYPE` | `deploy-azure-sql-and-config.yml` |

---

## 9. Copilot Workshop Features

| Feature | Location |
|---------|---------|
| Custom prompt: Snowpark transform | `.copilot/prompts/create-snowpark-transform.prompt.md` |
| Custom prompt: Validator rule | `.copilot/prompts/generate-validator-rule.prompt.md` |
| Custom prompt: Legacy SQL explain | `.copilot/prompts/explain-legacy-sql-to-modern-sql.prompt.md` |
| Custom prompt: C# cleaner implementation | `.github/prompts/implement-csharp-cleaner.prompt.md` |
| Data Engineer Agent | `.github/agents/data-engineer.agent.md` |
| QA Validator Agent | `.github/agents/qa-validator.agent.md` |
| Secure Code Reviewer Agent | `.github/agents/secure-code-reviewer.agent.md` |
| Documentation Updater Agent | `.github/agents/documentation-updater.agent.md` |
| Product Docs Agent | `.github/agents/product-docs.agent.md` |
| Repo-level Copilot instructions | `copilot-instructions.md` |

---

## Getting Help

- See [docs/architecture.md](architecture.md) for the full system diagram
- See [docs/data-contracts.md](data-contracts.md) for table schemas
- See [docs/demo-script.md](demo-script.md) for the workshop walkthrough
- Open an issue using the [Bug Report](.github/ISSUE_TEMPLATE/bug_report.md) template
