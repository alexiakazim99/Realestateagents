# Document Analysis Agent — Technical Summary

> **Naming note:** the workflow's actual name in n8n is **"Document summary Agent"**, not "Document Analysis Agent". Documented here under the requested folder/label for consistency with the other two agents, but the n8n instance itself still shows the older name.

**n8n workflow name:** `Document summary Agent`
**Workflow ID:** `y4PQC8qh4QLQOhvM`
**Status:** Active (production)
**Models:** GPT-4.1 for PDF/TXT, **GPT-4** for CSV/XLSX (see "Notes" below)

## Purpose

Lets a broker upload a document (contract, inspection report, annual report, financial spreadsheet, etc.) plus optional keywords, and returns a structured risk-flagged analysis: summary, a 🟢/🟡/🔴 risk level, concrete warnings with source references, extracted key data, and suggested next steps — explicitly without giving legal/tax advice.

## Trigger

An n8n **Form Trigger** ("On form submission") titled "ladda upp dokument" with two fields: `nyckelord` (optional keywords to focus the analysis on) and `dokument` (file upload).

## Node-by-node flow

| Order | Node | Type | What it does |
|---|---|---|---|
| 1 | **On form submission** | `n8n-nodes-base.formTrigger` | Entry point; collects `nyckelord` + the uploaded `dokument` file. |
| 2 | **Switch** | `n8n-nodes-base.switch` | Routes based on the uploaded file's MIME type: `application/pdf`, `text/plain`, `text/csv`, `application/vnd...spreadsheetml.sheet` (xlsx), with a fallback branch ("extra") for anything else. |
| 3a | **PDF-file** | `n8n-nodes-base.extractFromFile` (pdf) | Extracts text from PDF uploads. |
| 3b | **TEXT-file** | `n8n-nodes-base.extractFromFile` (text) | Extracts text from `.txt` uploads into `text`. |
| 3c | **CSV-file** | `n8n-nodes-base.extractFromFile` | Parses CSV into rows/columns. |
| 3d | **XLSX-file** | `n8n-nodes-base.extractFromFile` (xlsx) | Parses XLSX into rows/columns. |
| 3e | **Fel filtyp** | `n8n-nodes-base.set` | Fallback branch for unsupported file types — sets a Swedish error message ("Filformatet stöds inte...") and the branch dead-ends (no further connection). |
| 4 | **If statment / If statment1** | `n8n-nodes-base.if` | For the TXT and PDF branches respectively: checks `$json.text` is non-empty before proceeding to analysis; empty → routed into `Merge`. |
| 4 | **If statment2** (XLSX) / **If statment** (CSV, reused name) | `n8n-nodes-base.if` | For CSV/XLSX branches: checks the parsed item count is non-empty; empty → routed into `Merge`. |
| 5a | **textanalysis(PDF/TXT)** | `@n8n/n8n-nodes-langchain.openAi` (GPT-4.1, `executeOnce: true`) | Sends the extracted text + keywords to GPT-4.1 with the document-analysis system prompt (see `system-prompts.md`) — produces the summary/risk/warnings/key-data/next-steps structure. |
| 5b | **Dataanalysis(CSV/XLSX)** | `@n8n/n8n-nodes-langchain.openAi` (**GPT-4**, `executeOnce: true`) | Sends the parsed row data (`JSON.stringify(...)`) + keywords to GPT-4 with a separate, table-data-focused system prompt (see `system-prompts.md`). |
| 6a | **Markdown** | `n8n-nodes-base.markdown` | Converts the PDF/TXT analysis output to HTML. |
| 6b | **Markdown1** | `n8n-nodes-base.markdown` | Converts the CSV/XLSX analysis output to HTML. |
| 7 | **Merge** | `n8n-nodes-base.merge` (4 inputs) | Collects the "empty document" branches from all four file-type paths. |
| 8 | **Tomt dokument** | `n8n-nodes-base.set` | Sets a Swedish error message ("Dokumentet kunde inte läsas...") for the merged empty-document case. |

## Branching logic summary

- **Happy path:** file type recognized → text/data successfully extracted (non-empty) → sent to the matching analysis node → Markdown → HTML output.
- **Empty/unreadable file:** file type recognized but extraction yields nothing → merged into "Tomt dokument" → generic Swedish error message, no LLM call.
- **Unsupported file type:** Switch fallback → "Fel filtyp" → generic Swedish error message ("PDF, TXT, CSV eller XLSX"), dead-end, no LLM call.

## Credentials used

- **OpenAI account** (`lBbz6qPfrtWoFyLz`) — used by both "textanalysis(PDF/TXT)" and "Dataanalysis(CSV/XLSX)".

## Notes / potential inconsistencies worth flagging

- **Model mismatch:** every other agent/node in this system (Lead Agent, Text Content Generator) uses **GPT-4.1**, but the CSV/XLSX analysis branch here uses plain **GPT-4**. Unclear if intentional; worth confirming.
- Two `If` nodes are both named "If statment" / "If statment " (trailing space distinguishes them) and another pair "If statment1" / "If statment2" — functionally fine since n8n keys connections by internal node ID, but easy to mix up when reading the canvas.
- The two system prompts are structurally similar (summary → risk flag → findings → key data → next steps) but tuned differently: the PDF/TXT prompt is oriented around contract/inspection-style risk (fukt, skulder, klausuler), the CSV/XLSX prompt around financial-data anomalies (avvikande kostnader, negativa värden).
