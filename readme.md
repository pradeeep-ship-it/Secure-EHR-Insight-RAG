# Secure EHR Insight – Clinical Validator

**Author:** Pradeep Kumar Yadav

A zero-trust clinical RAG assistant over MIMIC-IV records. Clinicians ask questions about a patient in plain English; the system retrieves that patient's records with vector search, strips PHI before anything reaches the LLM, and blocks requests for medical advice.

## Architecture

- **PostgreSQL + pgvector** stores patient encounters and their embeddings
- **BioClinical ModernBERT** (`NeuML/bioclinical-modernbert-base-embeddings`) creates the 768-dimension embeddings
- **Microsoft Presidio** removes PHI (names, SSNs, phone numbers, emails, dates, hospitals) from retrieved records
- **NeMo Guardrails** (Groq, gpt-oss-120b) blocks requests for medical advice
- **FastAPI** backend (`src/api/main.py`) and **Streamlit** UI (`src/ui/app.py`)

## My work on this project

- Set up and debugged the full stack locally on Windows (Python 3.12, PostgreSQL 16 + pgvector)
- Moved the LLM layer to Groq's free tier (`openai/gpt-oss-120b`) through NeMo Guardrails' OpenAI-compatible client
- Fixed database connections: scripts 01–05 now use the installed psycopg v3 driver
- Diagnosed a spaCy model-loading crash (`OverflowError` in the tokenizer) down to `preshed` 3.0.14 and pinned a working version
- Added a Docker Compose setup for Postgres + pgvector, a `.env` template and this run guide
- Tested retrieval, PII redaction and guardrails end to end across 7 patients

## Prerequisites

- [uv](https://docs.astral.sh/uv/) (installs Python 3.12 for you)
- Docker Desktop (for the local Postgres + pgvector), or any Postgres instance with pgvector installed
- A free Groq API key (https://console.groq.com/keys)
- The dataset: download the [MIMIC-IV cleaned medical transcripts](https://www.kaggle.com/datasets/isaacritharson/mimic-iv-cleaned-medical-transcripts) CSV from Kaggle and save it as `data/MIMIC_IV_Trasncript.csv` (not included in this repo because of the MIMIC data usage terms)

## Setup (Windows PowerShell)

```powershell
uv venv --python 3.12 .venv
.venv\Scripts\activate
uv pip install -r requirements.txt tqdm

copy .env.example .env      # then set OPENAI_API_KEY to your Groq key
docker compose up -d        # Postgres 16 + pgvector on localhost:5432
```

## Load data and embeddings (one-time)

```powershell
python scripts/01_ingest_baseline_data.py   # create table + load CSV (appends: run only once)
python scripts/02_verify_ingestion.py
python scripts/03_apply_vector_schema.py    # add vector(768) column
python scripts/04_generate_embeddings.py    # embeds the first 100 rows (demo_limit)
python scripts/05_test_vector_search.py
```

## Standalone checks

```powershell
python src/pii_redaction/presidio_service.py   # PII redaction demo, no DB or API key needed
python scripts/06_test_guardrails.py           # needs OPENAI_API_KEY
```

## Run the app

Use two terminals, both with the venv activated:

```powershell
uvicorn src.api.main:app --reload     # http://localhost:8000/docs
streamlit run src/ui/app.py           # http://localhost:8501
```

## Example questions

Select a patient in the sidebar, then ask:

- What medications were prescribed to this patient upon discharge?
- What liver-related diagnoses are noted in the patient's file?
- Is the patient on a blood thinner?
- Should I increase the Warfarin dose? *(should be refused by the guardrail)*

## Acknowledgements

Built on the open-source baseline [Secure-EHR-Insight-Clinical-Validator](https://github.com/nimowhyca/Secure-EHR-Insight-Clinical-Validator).
