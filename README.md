# Field-Service-Document-Intelligence

A FastAPI service for ingesting, structuring, and reasoning over
field-service visit documents (visit notes, care plans, intake forms).
GCP Document AI for OCR / field extraction, MongoDB persistence, and a
Google ADK agent that summarizes visit history, surfaces follow-up
actions, and answers grounded questions over a subject's record.

This README is a direct walkthrough of `Docs/Design/architecture.drawio`.
Each section maps one-to-one to a tab in that file.

## Business domain

In-home care, field maintenance, social work, mobile inspection — any
operation where the work happens out in the field and gets recorded on
paper or PDF visit forms. Each subject (a patient at home, a property
under inspection, a member receiving services) accumulates a long tail
of documents over time: intake forms, visit notes, care plans, consent
forms, checklists. Three recurring problems:

1. The visit document is filled in *during* the visit, often by hand,
   with key fields buried inside free-text or in scanned form sections.
2. The next person on the case (a different field worker, a care
   coordinator, a supervisor) needs the **history** before they can
   make their next decision — and reading every prior visit by hand
   does not scale.
3. Important follow-up tasks ("subject reported chest pain", "consent
   form expired", "next appointment in 14 days") get lost in free-text
   unless the system surfaces them.

This service ingests every visit document, extracts structured fields,
keeps a per-subject longitudinal record, and lets the next worker ask
grounded questions over that record before they walk in the door.

## Business features

| Feature | What the user gets |
|---|---|
| Visit document upload | A `POST /visits/{id}/documents` endpoint that accepts a file plus the visit + doc type. |
| Structured field extraction | Document AI parses the form; the service normalizes the fields, validates them, and stores them next to the raw document. |
| Longitudinal subject record | `subjects` ↔ `visits` ↔ `documents` ↔ `extracted_fields` so a coordinator can pull every recorded fact about a subject in one query. |
| Follow-up detection | Care-plan rules scan extracted fields on ingest; new tasks are created automatically. |
| Subject summarization | A Google ADK agent reads the recent history and writes a short, cited narrative — perfect for the "what's new since last visit?" pre-brief. |
| Grounded Q&A | "What is outstanding for subject X?" returns an answer plus the document IDs the answer rests on. |
| PHI-aware logging | The structured logger and the response paths know which fields are PHI and never log them in the clear. |

## Architecture (mapped to `Docs/Design/architecture.drawio`)

### Tab 1 — System Overview

The end-to-end flow for one visit document and one subject query:

- **Visit documents** (intake forms, visit notes, care plans,
  checklists) → **GCP Document AI** (OCR + form field extraction)
- → **FastAPI ingestion** normalizes, validates, attaches the
  document to the right subject and visit
- → **MongoDB** holds `subjects`, `visits`, `documents`,
  `extracted_fields`, `summaries`
- A **Subject record API** (`/subjects`, `/visits`, `/summary`,
  `/ask`) reads from the same store
- A **Google ADK agent** sits next to the API and uses it to
  summarize history, surface follow-ups, and answer grounded
  questions
- A **care coordinator / field staff** user reaches both the API
  and the agent

### Tab 2 — Visit Document Ingestion

The ingestion path: `POST /visits/{id}/documents` → Doc AI processor
(intake form / visit note / care plan) → **Field normalizer** (coerce
to schema, units, enums) → **Validation** (required fields, ranges,
cross-field rules) → **Persist to MongoDB** (raw + parsed `documents`
plus `extracted_fields`, both linked to `subject_id` and `visit_id`)
→ **Followup detection** (rule scan creates new tasks if extracted
fields trigger care-plan rules).

### Tab 3 — Subject Record Aggregation

Five MongoDB collections and how they reference each other:

- `subjects` — `_id`, `tenant_id`, hashed `display_id`, enrollment
  dates, assigned team
- `visits` — `subject_id`, visit_date, visit_type, hashed clinician
  id, status
- `documents` — `visit_id`, `subject_id`, doc_type, mime_type,
  raw_text, ocr_confidence
- `extracted_fields` — `document_id`, field_name, value, confidence,
  `phi: bool` flag
- `summaries` — `subject_id`, produced_at, model, prompt_hash,
  narrative, `cited_doc_ids[]`, `outstanding_items[]`

### Tab 4 — ADK Agent Tools

One `LlmAgent` (`mesa_field_intel`, model `gemini-2.0-flash`) with an
instruction that grounds every answer in cited documents and never
invents facts. Four tool functions it can call:

- `get_subject(subject_id)` — Mongo `subjects` + `visits` join
- `get_visit_documents(visit_id)` — `documents` + `extracted_fields`
- `search_documents(subject_id, query, k)` — vector or text search
  scoped to one subject
- `list_outstanding_items(subject_id)` — rule-based scan over recent
  extracted fields

The agent's output is persisted to the `summaries` collection so the
next caller doesn't pay the same LLM cost twice.

### Tab 5 — Sequence: "what is outstanding for subject X?"

UML sequence: care coordinator → `POST /subjects/X/ask` → ADK agent →
`get_subject(X)` → `list_outstanding_items(X)` → `search_documents(X,
question, k=5)` → grounded prompt to Gemini → `answer + cited_doc_ids`
back to the agent → summary persisted to `summaries` → response back
to the coordinator.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| HTTP | FastAPI | async, Pydantic-typed request/response, OpenAPI for free |
| OCR / form extraction | GCP Document AI | covers OCR plus form-specific processors per document type |
| Storage | MongoDB | per-subject document model fits the longitudinal record naturally; aggregation pipelines for the summary join |
| Agent runtime | Google ADK (`google.adk.agents.LlmAgent` + `Runner`) | one LlmAgent per concern, scoped tool functions, structured prompt instructions |
| LLM | Gemini (default `gemini-2.0-flash`, configurable via `LLM_MODEL`) | reached via google-adk |
| Settings | pydantic-settings | typed env-driven config |
| Logging | structlog with PHI processor | JSON output; PHI fields tagged on the schema are never logged in the clear |
| Tests | Pytest + MagicMock | offline-first; mock Document AI and Gemini at the boundary |

## Why this exists

This is a portfolio bridge repo authored to map a previous
field-service / in-home delivery problem into the JD stack of a
current opportunity (FastAPI, PyMongo, GCP Document AI, RAG, Google
ADK agent, prompt grounding, Pytest + MagicMock). The architecture
diagram is the authoritative source — this README walks through it
for readers who want the prose version.

## Status

Scaffolded. The architecture diagram is committed; module
implementation is the next iteration. See
`Docs/Design/architecture.drawio` for the target architecture.
