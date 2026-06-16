# B2B Lead Intake and Qualification Pipeline

A production-style **B2B lead intake and qualification pipeline** built in **n8n**. The workflow accepts inbound leads from two sources — a website webhook and CSV import — then normalises messy lead data, validates required fields, checks for duplicate or related CRM organisations, performs mock company enrichment with response validation, scores each lead, routes it into the right operational destination, and exports structured JSON outputs for CRM actions, human review, and audit logging.

---

## What This Project Demonstrates

This project is designed as a portfolio-ready automation workflow showing how a real lead operations pipeline can be built with n8n.

It demonstrates:

- Multi-source lead intake
- CSV parsing and webhook intake
- Lead field extraction and normalisation
- Data validation and rejection handling
- CRM-style entity resolution and deduplication
- Mock enrichment API integration
- Enrichment response validation
- Lead scoring from 0–100
- Internal lead tiering from A–D
- Client-facing score from 1–5
- Three-way business routing
- Structured JSON export
- Full audit logging
- Separate workflow-level error logging

---

## Architecture

```text
01A - Manual CSV Import Trigger
        |
        v
Read/Write Files from Disk
        |
        v
01C - Parse Lead CSV Rows
        |
        v
01E - Standardize Intake Stream
        ^
        |
01D - Webhook Lead Intake
        |
        v
02A - Extract and Normalize Lead Fields
        |
        v
03A - Validate Lead Fields
        |
        v
04A - Dedup Check Against Mock CRM
        |
        v
05A - Mock Enrichment API
        |
        v
06A - Validate Enrichment Response
        |
        v
07A - Score Lead
        |
        v
08A - Route Lead
        |
        v
09A - Prepare CRM and Audit Outputs
        |
        v
10A - Split Output Records
        |
        v
11A - Group Output Records by Destination
        |
        v
12A - Build Export Files
        |
        v
13A - Convert JSON Content to Binary
        |
        v
13B - Write Export Files
```

A separate error workflow receives unexpected n8n node failures and writes structured error logs to disk.

---

## Key Technical Decisions

### 1. Deduplication happens before enrichment

The workflow checks the mock CRM before enrichment. This avoids spending enrichment calls on leads that are already known, duplicated, or likely connected to an existing account.

In a real production setup, this reduces cost, avoids duplicate CRM records, and prevents sales teams from creating parallel deals for the same organisation.

### 2. Enrichment responses are validated before scoring

The workflow does not blindly trust enrichment data. Even if the mock enrichment API returns a successful response, the response is checked for domain and company consistency before being used for scoring.

This handles a common production issue: a third-party API can return HTTP 200 while still returning data for the wrong company.

### 3. The workflow does not invent missing data

If company identity, domain, or enrichment data is missing, the workflow preserves the uncertainty instead of guessing. Unknown values stay unknown, and weak records are routed to human review or audit.

This is important for CRM and sales operations because invented data creates false confidence and pollutes downstream systems.

### 4. Count reconciliation is built in

After routing, the workflow groups all output records and checks whether the audit log count matches the number of input leads. This ensures no records silently disappear during processing.

### 5. Error handling is separated

Unexpected workflow failures are handled by a separate n8n error workflow. This keeps the main workflow focused on business processing while still preserving failure visibility.

---

## Production Challenges Solved

### Entity resolution under semantic mismatch

The workflow handles cases where the same organisation appears in different forms:

```text
MetroSolar Solutions Sdn Bhd
Metro Solar Solutions
MS
MetroSolar Group
```

It uses domains, normalised company names, and fuzzy matching to detect duplicates, possible duplicates, and related subsidiaries.

### Successful response does not mean correct response

The mock enrichment API includes validation logic because a successful API response may still be wrong. The workflow checks whether returned enrichment data matches the expected company/domain before it is trusted.

### No invented data

The pipeline avoids creating fake company details when the input is weak. Generic Gmail leads, missing companies, and unclear company/person fields are routed to review instead of being treated as fully qualified CRM records.

---

## Routing Logic

Each lead is routed into one of three broad outcomes.

### 1. Qualified CRM output

Strong leads with good fit, valid data, and no duplicate conflict are prepared as mock CRM deal and activity payloads.

Output files:

```text
sample_outputs/mock_crm_deals.json
sample_outputs/mock_crm_activities.json
```

### 2. Human review queue

Borderline, ambiguous, or risky records are sent to human review. Examples include:

- Missing company identity
- Generic email with weak company context
- Possible duplicate or subsidiary relationship
- Non-target market lead
- Supplier/vendor-style inquiry
- Enrichment mismatch

Output file:

```text
sample_outputs/human_review_queue.json
```

### 3. Audit log

Every lead produces an audit record, including qualified, duplicate, review, rejected, and empty records.

Output file:

```text
sample_outputs/audit_log.json
```

---

## Output Files

The workflow writes four JSON files:

```text
sample_outputs/
├── mock_crm_deals.json
├── mock_crm_activities.json
├── human_review_queue.json
└── audit_log.json
```

Each file contains:

- file type
- generation timestamp
- record count
- scoring note
- quality checks
- source counts
- structured records

The audit log is the source of truth for reconciliation.

---

## Test Dataset

The test CSV intentionally contains messy lead data to simulate real-world inbound lead quality issues.

Included failure modes:

- Clean qualified lead
- Duplicate lead
- Same parent domain with different subsidiary
- Abbreviation vs full company name
- Generic Gmail address
- Missing company name
- Company field containing a person name
- Supplier/vendor-style inquiry
- Non-target market lead
- Malay-language message
- Empty row
- Domain-based enrichment match
- Enrichment missing / unknown case
- Human review case
- Audit-only case

Dataset location:

```text
data/messy_leads_test_dataset.csv
```

---

## Running Locally

### Requirements

- Docker Desktop
- n8n running through Docker Compose
- Git

### Start n8n

```bash
docker compose up -d
```

Then open:

```text
http://localhost:5678
```

### Docker Compose

The project uses local volume mounts so n8n can read the CSV input and write JSON outputs.

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_SECURE_COOKIE=false
      - GENERIC_TIMEZONE=Asia/Kuala_Lumpur
      - TZ=Asia/Kuala_Lumpur
    volumes:
      - ./n8n_data:/home/node/.n8n
      - ./data:/home/node/.n8n-files/data
      - ./sample_outputs:/home/node/.n8n-files/sample_outputs
      - ./error_logs:/home/node/.n8n-files/error_logs
```

The local runtime folders are ignored by Git:

```text
n8n_data/
error_logs/
```

---

## Importing the Workflows

Import these workflow JSON files into n8n:

```text
workflow/b2b-lead-intake-and-qualification-pipeline.json
workflow/lead-pipeline-error-handler.json
```

After importing, set the main workflow error workflow to:

```text
Lead Pipeline Error Handler
```

---

## Running the CSV Flow

1. Place the test CSV in:

```text
data/messy_leads_test_dataset.csv
```

2. Execute the workflow from:

```text
01A - Manual CSV Import Trigger
```

3. Check generated outputs in:

```text
sample_outputs/
```

---

## Testing the Webhook Flow

The webhook path is:

```text
/webhook/b2b-lead-intake
```

Example PowerShell request:

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:5678/webhook/b2b-lead-intake" `
  -Method POST `
  -ContentType "application/json" `
  -Body '{
    "timestamp": "2026-06-16T14:00:00+08:00",
    "first_name": "Test",
    "last_name": "Lead",
    "company_name": "GridNusa Energy Sdn Bhd",
    "email": "test.lead@gridnusa.example",
    "phone": "+60 12 3456789",
    "country": "Malaysia",
    "interest": "Energy storage",
    "message": "We are looking for battery storage and grid integration options.",
    "source": "web_form",
    "language": "en"
  }'
```

The webhook response only confirms the workflow started. Final outputs are written to JSON files.

---

## Repository Structure

```text
b2b-lead-intake-pipeline/
├── data/
│   └── messy_leads_test_dataset.csv
├── sample_outputs/
│   ├── mock_crm_deals.json
│   ├── mock_crm_activities.json
│   ├── human_review_queue.json
│   └── audit_log.json
├── workflow/
│   ├── b2b-lead-intake-and-qualification-pipeline.json
│   └── lead-pipeline-error-handler.json
├── docker-compose.yml
├── .gitignore
└── README.md
```

---

## Notes

This project uses mock CRM and mock enrichment data for local portfolio demonstration. In a production deployment, these mock nodes can be replaced with live CRM, enrichment, notification, or database integrations while keeping the same validation, scoring, routing, and audit structure.
