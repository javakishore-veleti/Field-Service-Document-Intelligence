# Development Plan — Field-Service-Document-Intelligence

A focused punch list to take this repo from a scaffold to the
architecture in `Docs/Design/architecture.drawio`. Each block builds
out one tab of that diagram so the diagram and the code stay in sync.

Legend: `[ ]` not started · `[~]` in progress · `[x]` done

## Block 1 — Project skeleton

- [ ] **T1.1** `pyproject.toml` with FastAPI, pydantic-settings,
       pymongo, structlog, pytest, pytest-asyncio, httpx,
       mongomock as the dev extras. `[adk]` extra pulls
       `google-adk`.
- [ ] **T1.2** `src/field_service_document_intelligence/`
       package layout — `ingestion/`, `subjects/`, `agent/`,
       `api/`, `config.py`, `logging.py`, `main.py`.
- [ ] **T1.3** `tests/` mirror the same tree.
- [ ] **T1.4** `pytest -q` green on a smoke test.

## Block 2 — Subject record store (Tab 3)

- [ ] **T2.1** `subjects/schemas.py` — Pydantic v2 models for
       `Subject`, `Visit`, `Document`, `ExtractedField`, `Summary`.
       PHI-tagged fields use `Field(json_schema_extra={"phi":
       True})`.
- [ ] **T2.2** `subjects/indexes.py` — `(tenant_id, subject_id)`,
       `(subject_id, visit_date)` compound, `document_id` unique.
- [ ] **T2.3** `subjects/store.py` — `SubjectStore` wrapping
       `pymongo.MongoClient`. Tenant filter enforced inside the
       class. `mongomock`-friendly.
- [ ] **T2.4** `tests/subjects/test_store.py` — round-trip tests.

## Block 3 — Visit document ingestion (Tab 2)

- [ ] **T3.1** `ingestion/docai.py` — Document AI wrapper with a
       `FakeDocAIClient` for tests.
- [ ] **T3.2** `ingestion/normalizer.py` — coerce extracted fields
       to schema (units, enums, type cast).
- [ ] **T3.3** `ingestion/validator.py` — required fields, ranges,
       cross-field rules.
- [ ] **T3.4** `ingestion/followups.py` — care-plan rule scan over
       extracted fields; emits new task records.
- [ ] **T3.5** `ingestion/pipeline.py` — composes Doc AI →
       normalizer → validator → store → followup detection.
- [ ] **T3.6** `tests/ingestion/test_pipeline.py` — happy path
       end-to-end against the fakes.

## Block 4 — ADK agent + tools (Tab 4)

- [ ] **T4.1** `agent/tools.py` — four tool functions:
       `get_subject(subject_id)`, `get_visit_documents(visit_id)`,
       `search_documents(subject_id, query, k)`,
       `list_outstanding_items(subject_id)`. Each takes the
       `SubjectStore` directly so the suite stays offline.
- [ ] **T4.2** `agent/llm.py` — `LlmClient` Protocol +
       `FakeLlmClient` with canned `NarrativeResponse`.
- [ ] **T4.3** `agent/google_adk_client.py` — concrete
       `GoogleAdkClient` wrapping `google.adk.agents.LlmAgent` +
       `InMemoryRunner` (deferred import; `pip install ".[adk]"`
       required to instantiate).
- [ ] **T4.4** `agent/agent.py` — orchestrates the four tool calls
       + an LLM summary call; persists the result to `summaries`.
- [ ] **T4.5** `tests/agent/test_agent.py` — fake LLM round trip
       against a seeded `SubjectStore`.

## Block 5 — FastAPI surface (Tab 1)

- [ ] **T5.1** `api/app.py` — `create_app()` factory; mounts
       `/healthz`, `/version`, `/subjects`, `/visits`,
       `/visits/{id}/documents`, `/subjects/{id}/summary`,
       `/subjects/{id}/ask`.
- [ ] **T5.2** `api/deps.py` — dependency wiring for
       `SubjectStore` and the agent.
- [ ] **T5.3** `tests/api/test_app.py` — `httpx.AsyncClient` smoke
       tests for `/healthz` and one round-trip ingest + ask
       (Tab 5 sequence).

## Block 6 — Observability + CI

- [ ] **T6.1** `logging.py` with a PHI-aware structlog processor
       that honors `phi: True` field metadata on the schemas.
- [ ] **T6.2** `.github/workflows/ci.yml` — pytest on every push
       and PR to `main` against Python 3.13 with the dev extras.
- [ ] **T6.3** `pytest -q` green locally and in CI.

## Definition of done

1. `pytest -q` green locally and in GitHub Actions.
2. `uvicorn field_service_document_intelligence.main:app` starts
   cleanly with the in-memory backend.
3. `POST /visits/{id}/documents` against a sample form lands a
   `Document` + `ExtractedField` rows in the store.
4. `POST /subjects/{id}/ask` returns an answer with cited
   `doc_ids` and persists a `Summary` row.
5. Every block lands as its own commit.

## Status

Scaffold + architecture diagram + README walkthrough committed.
Block 1 is the next iteration.
