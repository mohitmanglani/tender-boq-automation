# Tender BOQ Automation Pipeline

> Automated extraction of Bill of Quantities from Indian Railways tender PDFs into structured Excel reports — deployed on a self-hosted VPS, triggered by Google Drive, zero manual intervention.

---

## What This Does

Indian Railways tenders arrive as dense, multi-schedule PDF documents. Manually copying BOQ line items into Excel takes hours per tender and introduces errors. This pipeline eliminates that.

**Drop a PDF into a Google Drive folder → get a formatted Excel BOQ in ~30 seconds.**

---

## Demo

| Input | Output |
|-------|--------|
| `BCT-25-26-240.pdf` (72-page tender) | `BCT-25-26-240_BOQ.xlsx` — 15 schedules, 423 line items, all formulas intact |

---

## Architecture

```
Google Drive (Tenders/Inbox)
        │
        ▼ [n8n Trigger — polls every 15 min]
        │
        ▼ [Download PDF Node]
        │
        ▼ [HTTP Request → Anthropic Claude API]
          PDF sent as base64 document block
          Claude extracts structured JSON
        │
        ▼ [Code Node — JavaScript + ExcelJS]
          JSON → formatted Excel with formulas
        │
      ┌─┴─────────────────────┐
      ▼ (success)             ▼ (failure)
Upload Excel            Move PDF to
to Outputs/             Tenders/Failed/
Move PDF to
Tenders/Processed/
```

**Stack:**
- **n8n** — workflow orchestration (self-hosted on Hetzner CX23, Helsinki)
- **Anthropic Claude API** — PDF parsing (Haiku 4.5 default, Sonnet 4.6 fallback)
- **ExcelJS** — Excel generation with formatting, formulas, and dropdowns (host-mounted via NODE_PATH)
- **Google Drive** — input/output storage + trigger
- **Docker** — containerized n8n deployment

---

## What Gets Extracted

For each tender, the pipeline outputs a structured Excel with:

**Tender-level metadata**
- Tender ID, Name of Work, Advertised Value

**Per-schedule data** (Schedule A, B, C … all schedules present)
- Schedule name, Escalation %

**Per-line-item data**
- Item No., Description, Unit, Quantity, Rate, Amount
- Cross-referenced formulas (`=D{n}*E{n}`) preserved in output

---

## Key Technical Decisions

**Why Claude API instead of a rule-based parser?**

Indian Railways tenders have no consistent schema. Table formatting, schedule headers, and item numbering vary across divisions and tender types. A regex/pdfplumber approach works on one format and breaks on the next. Claude reads the semantic structure regardless of layout variation.

**Why n8n instead of a pure Python script?**

Visual pipeline = auditable, modifiable by non-developers. The client can see exactly what happens at each step without reading code. Failure routing (Processed vs Failed folders) is clear and self-documenting. Excel generation happens inside n8n's Code node — no external service needed.

**Why self-hosted (Hetzner) instead of cloud functions?**

Cost control for a B2B client deployment. ~€4/month for the VPS vs variable serverless pricing at volume. Full control over runtime, no cold starts, predictable latency.

---

## Repository Structure

```
tender-boq-automation/
├── n8n/
│   └── workflow_export.json      # Full n8n workflow (importable)
├── docs/
│   ├── architecture.md           # Detailed architecture notes
│   ├── deployment.md             # VPS + Docker setup steps
│   └── prompt_design.md          # Claude API prompt + schema
├── sample/
│   ├── sample_input.pdf          # Anonymized tender PDF
│   └── sample_output.xlsx        # Generated BOQ Excel
└── README.md
```

---

## Deployment

**Prerequisites:** Hetzner (or any Ubuntu VPS), Docker, Google Cloud OAuth credentials, Anthropic API key.

```bash
# SSH into VPS
ssh root@<server-ip>

# Clone and configure
git clone https://github.com/<you>/tender-boq-automation
cd tender-boq-automation

# Set environment variables
cp .env.example .env
# Fill in: ANTHROPIC_API_KEY, N8N_BASIC_AUTH_USER, N8N_BASIC_AUTH_PASSWORD

# Start n8n
docker compose up -d

# Import workflow
# n8n UI → Settings → Import from File → n8n/workflow_export.json
```



---

## Results

Tested against 3 real BCT division tenders:

| Tender | Schedules | Line Items | Processing Time | Accuracy |
|--------|-----------|------------|-----------------|----------|
| BCT-25-26-240 | 15 | 423 | 28s | 100% |
| BCT-25-26-241 | 8 | 187 | 19s | 97%* |
| BCT-25-26-312 | 11 | 294 | 24s | 100% |

*One schedule had a non-standard header format. Escalation % missed; item data correct.

---

## What's Next (Phase 2)

Automate **Variation Statement** generation from Measurement Bank PDFs.

A variation statement records how actual quantities deviated from the work order — per item, per chapter, across all schedules. Currently produced manually in Excel. Target output mirrors the standard format used by Indian Railways divisions:

**Per line item:** W.O. Quantity, Actual Quantity, Excess/Saving Quantity, W.O. Amount, Actual Amount, Upto-date Amount, Excess/Saving Amount — with escalation % deduction and contingency rate deduction applied per chapter.

**Pipeline:** Measurement Bank PDF → Claude API extracts recording & billing quantities → variation statement Excel generated automatically, ready for submission.

- [ ] Measurement Bank PDF parsing (recording quantity + billing quantity)
- [ ] Variation statement Excel generation per chapter/schedule
- [ ] Billing vs recording quantity reconciliation
- [ ] Multi-tender batch processing

---

## About

Built as a client project for an Indian Railways contractor. Reduced per-tender processing time from ~3 hours to ~30 seconds.

**Mohit Manglani** — [LinkedIn](https://linkedin.com/in/mohitmanglani-data) · (mailto: manglanimohit01@gmail.com)
