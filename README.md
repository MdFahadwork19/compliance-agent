# Documentation Compliance Agent — WaiverPro

An automated system that verifies whether the live **WaiverPro** web application conforms to its official PDF guidelines. Built as a four-stage agentic pipeline.

---

## Architecture Overview

```
PDF Guidelines ──► Stage 1: Ingest & Parse   ──► rules.json + ChromaDB
Live Web App   ──► Stage 2: Extract & Crawl  ──► ui_elements.json + screenshots/
                   Stage 3: AI Agent (RAG)    ──► comparison_results.json + discrepancies.json
                   Stage 4: Report Generator  ──► report.html
```

### Stage 1 — Ingest & Parse (`stage1_ingest/`)
Reads the official WaiverPro PDF, extracts text by section, breaks it into checkable rule-sentences, and embeds them into a ChromaDB vector store for semantic retrieval.

### Stage 2 — Extract & Crawl (`stage2_extract/`)
Uses **Playwright** to authenticate and crawl the live application. Captures full-page screenshots and extracts UI elements (buttons, nav items, text blocks, inputs) into structured JSON matching the canonical schema.

### Stage 3 — AI Compliance Agent (`stage3_agent/`)
For each page, retrieves the most relevant guideline rules via RAG (ChromaDB), then sends both the live UI snapshot and the rules to **Gemini 2.5 Flash** which returns a structured JSON verdict with match status, confidence, guideline citation, and discrepancy explanation.

### Stage 4 — Report (`stage4_report/`)
Generates an HTML compliance report from the comparison results, with summary statistics, pass/fail cards, guideline citations, and embedded screenshots.

---

## Setup Instructions

### Prerequisites
- Python 3.10+
- A Gemini API key (free at [aistudio.google.com](https://aistudio.google.com/app/apikey))

### 1. Clone & create virtual environment
```bash
git clone <your-repo-url>
cd compliance_agent
python -m venv venv

# Activate (Windows)
venv\Scripts\activate

# Activate (Mac/Linux)
source venv/bin/activate
```

### 2. Install dependencies
```bash
pip install -r requirements.txt
playwright install chromium
```

### 3. Configure environment
```bash
cp .env.example .env
# Edit .env and add your GEMINI_API_KEY
```

---

## How to Run Each Stage

All stages must be run from the **project root** directory.

### Stage 1 — Parse the PDF and build the vector store
```bash
python stage1_ingest/parse_pdf.py
python stage1_ingest/embed_rules.py
```
**Output:** `data/rules.json`, `data/chroma_store/`

### Stage 2 — Crawl the live application
```bash
cd stage2_extract
python crawler.py
cd ..
```
**Output:** `data/ui_elements.json`, `data/screenshots/*.png`

### Stage 3 — Run the compliance agent
```bash
cd stage3_agent
python agent.py
cd ..
```
**Output:** `data/comparison_results.json`, `data/discrepancies.json`

### Stage 4 — Generate the HTML report
```bash
python stage4_report/report_generator.py
```
**Output:** `stage4_report/report.html` — open in any browser.

---

## Canonical Data Schema

Every extracted element and comparison result uses this schema (as required by the assignment):

| Field | Description |
|---|---|
| `page_url` | Full URL of the page (e.g. `/dashboard`) |
| `component_type` | `button`, `text_block`, `navigation_item`, `input_field`, `table`, `image` |
| `component_selector` | CSS selector or structural path for reproducibility |
| `actual_text_content` | Text found on the live site (`null` for images/icons) |
| `expected_text_content` | What the PDF guidelines specify |
| `guideline_reference` | Section citation (e.g. "2. Accessing WaiverPro") |
| `discrepancy_flag` | Boolean — `true` if mismatch detected |
| `discrepancy_reason` | LLM-generated explanation of the mismatch |
| `screenshot_path` | File path to the captured screenshot |
| `retrieved_at` | ISO 8601 UTC timestamp |

---

## Tool Selection Justification

| Component | Tool Chosen | Alternatives Considered | Reason |
|---|---|---|---|
| Browser automation | **Playwright** | Selenium, Puppeteer | Async-native, reliable auth state serialization, better JS SPA support |
| HTML parsing | **BeautifulSoup4** | lxml, selectolax | Simple API, sufficient for structured extraction |
| PDF parsing | **pdfplumber** | PyMuPDF, pdfminer | Best text extraction quality with layout awareness |
| Vector store | **ChromaDB** | Pinecone, FAISS, Weaviate | Local, zero-config, good Python SDK |
| LLM | **Gemini 2.5 Flash** | GPT-4o, Claude Sonnet | Fast, cost-effective, strong JSON instruction-following |
| Retry logic | **tenacity** | Manual retry loops | Clean decorator API, exponential backoff |
| Logging | **loguru** | stdlib logging | Zero-config, colored output, structured |

---

## Project Structure

```
compliance_agent/
├── .env.example              # Environment variable template
├── .gitignore
├── README.md
├── requirements.txt
├── test_gemini.py            # Quick Gemini API connectivity check
├── test_key.py               # Quick API key validation
│
├── data/
│   ├── WaiverPro-User-Guidelines.pdf   # Source PDF (guideline document)
│   ├── rules.json                       # Parsed rules (Stage 1 output)
│   ├── chroma_store/                    # Vector embeddings (Stage 1 output)
│   ├── auth_state.json                  # Saved browser session (Stage 2)
│   ├── ui_elements.json                 # Extracted UI elements (Stage 2 output)
│   ├── screenshots/                     # Page screenshots (Stage 2 output)
│   ├── comparison_results.json          # All comparison results (Stage 3 output)
│   └── discrepancies.json               # Flagged mismatches only (Stage 3 output)
│
├── stage1_ingest/
│   ├── parse_pdf.py          # Extracts sections and rules from PDF
│   └── embed_rules.py        # Embeds rules into ChromaDB
│
├── stage2_extract/
│   ├── crawler.py            # Playwright crawler — auth + route discovery + screenshots
│   ├── dom_parser.py         # BeautifulSoup HTML → structured element list
│   ├── extractor.py          # Saves elements to JSON
│   └── test_login.py         # Standalone login test script
│
├── stage3_agent/
│   ├── agent.py              # Orchestrates comparison loop across all pages
│   ├── compare.py            # Gemini LLM comparison — RAG + structured verdict
│   ├── retrieve.py           # ChromaDB semantic rule retrieval
│   ├── prompts.py            # System prompts and prompt builder
│   ├── test_compare.py       # Test: single page comparison
│   └── test_retrieve.py      # Test: rule retrieval query
│
├── stage4_report/
│   ├── template.html         # HTML report template with {{ placeholders }}
│   ├── style.css             # Report stylesheet
│   ├── report_generator.py   # Fills template from data and writes report.html
│   └── report.html           # Final generated report (open in browser)
│
└── output/                   # Reserved for additional outputs
```

---

## Known Limitations

1. **Comparison granularity:** Stage 3 currently compares one page at a time (up to 50 elements per page snapshot) against 3 RAG-retrieved rules. Fine-grained element-by-element matching would require more API calls.

2. **Dynamic JS content:** Playwright captures the DOM after a 2.5 second wait. Lazy-loaded or interaction-triggered components may be missed.

3. **ChromaDB embedding model:** Uses the default `all-MiniLM-L6-v2` sentence transformer (local). A larger model or OpenAI embeddings would improve retrieval quality.

4. **`/applications` route returns 404:** The live site currently returns a 404 error on `/applications`, which is documented as a discrepancy in the report.

5. **PDF section detection:** Heading regex (`^\d+[\.\)]\s+[A-Z]...`) may miss non-standard headings. Worked well for this specific PDF.

---

## What I Would Improve Next

- **Richer extraction:** Add aria-label, role, and data-testid capture for better selector stability.
- **Element-level comparison:** Match each UI element to its closest guideline rule individually, not per-page.
- **Parallel crawling:** Run multiple Playwright pages concurrently for faster extraction.
- **CI pipeline:** Add GitHub Actions to re-run the audit on a schedule and alert on new discrepancies.
- **Coverage report:** Track which guideline sections had zero matching UI elements (blind spots).
- **Screenshot diffing:** Add pixel-level comparison using Playwright's screenshot diff capabilities.

---

## Disclaimer

> This is an **automated compliance check**, not a manual QA replacement. Results should be reviewed by a human before acting on discrepancies. LLM judgments may have false positives or false negatives.
