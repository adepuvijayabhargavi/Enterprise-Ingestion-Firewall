# Backend: Setup, Run and Troubleshooting (Windows)

Step-by-step commands to get the Ingestion Firewall API running on Windows PowerShell, plus how to diagnose the most common failure. Run every command from the `backend` folder unless a step says otherwise.

## 1. Add your Groq API key

Create `backend\.env` from the example (the repo only ships `.env.example`):

```powershell
copy .env.example .env
notepad .env
```

Set both values. No quotes, no spaces around `=`:

```env
GROQ_API_KEY=gsk_your_real_key_here
GROQ_MODEL=openai/gpt-oss-120b
```

Get a key at https://console.groq.com/keys. Never commit `.env`.

> **Model note:** the original default, `llama-3.3-70b-versatile`, no longer exists on Groq and causes every request to fail. Use `openai/gpt-oss-120b`, or another model from the list in step 8.

## 2. Create the virtual environment and install dependencies

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

If PowerShell blocks `Activate.ps1`, run this once, then activate again:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

## 3. Run the tests

```powershell
pytest -q
```

Expected: `4 passed`. These tests only check the Pydantic schemas; they do not call the LLM.

## 4. Start the backend (terminal 1)

```powershell
uvicorn app.main:app --reload --port 8000
```

Check it: http://127.0.0.1:8000/health should return `{"status":"ok",...}`.

## 5. Start the frontend (terminal 2)

From the project root (one level above `backend`):

```powershell
python -m http.server 5500 --bind 127.0.0.1 --directory frontend
```

## 6. Open in the browser

```powershell
Start-Process http://127.0.0.1:5500        # frontend
Start-Process http://127.0.0.1:8000/docs   # interactive API docs
```

## 7. Send a test request

```powershell
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8000/api/analyze `
  -ContentType "application/json" `
  -Body '{"message":"Where is my order?"}'
```

Expected response:

```json
{
  "intent": "order_tracking",
  "risk_profile": "medium",
  "priority": "medium",
  "requires_action": true,
  "summary": "User is asking for the status/location of their order.",
  "token_metadata": { "estimated_input_tokens": 5, "note": "..." }
}
```

Exact `summary` wording varies between runs.

## 8. Troubleshooting: `502 LLM request failed`

The API hides the real cause behind this generic message. Call the service directly to see the actual error (venv active):

```powershell
python -c "from app.services.llm_service import analyze_message; print(analyze_message('Hi'))"
```

| Error shown | Meaning | Fix |
|---|---|---|
| `401` / invalid API key | Key is wrong or revoked | Fix `GROQ_API_KEY` in `.env` |
| `404 model_not_found` | Model retired or not available to your key | Pick a model from the list below |
| `GROQ_API_KEY is missing` | `.env` not found or empty | Check `backend\.env` exists and is not `.env.txt` |
| Connection / timeout | No internet or Groq is down | Check network |

List the models your key can use:

```powershell
python -c "from groq import Groq; from app.config import GROQ_API_KEY; [print(m.id) for m in Groq(api_key=GROQ_API_KEY).models.list().data]"
```

To try a model without editing `.env` (the environment variable overrides the file for that terminal only):

```powershell
$env:GROQ_MODEL='openai/gpt-oss-120b'
uvicorn app.main:app --reload --port 8000
```

Restart `uvicorn` after any `.env` change.

## API reference

### `GET /health`

Returns `{"status": "ok", "message": "Ingestion Firewall API is running"}`. Does not call the LLM.

### `POST /api/analyze`

Request body:

```json
{ "message": "My payment was deducted twice." }
```

- `message`: 1–5000 characters, trimmed; whitespace-only is rejected. No other fields allowed.

Response fields:

| Field | Type / allowed values |
|---|---|
| `intent` | `greeting`, `order_tracking`, `payment_issue`, `current_time`, `general_question`, `unknown` |
| `risk_profile` | `low`, `medium`, `high`, `unknown` |
| `priority` | `low`, `medium`, `high` |
| `requires_action` | `true` if a follow-up service or tool would be needed to complete the request |
| `summary` | 1–500 characters describing the request |
| `token_metadata.estimated_input_tokens` | Local tiktoken estimate of the message, not billing data |

Status codes:

| Code | When |
|---|---|
| `200` | Valid, schema-checked result |
| `422` | Invalid request body (empty, too long, extra fields) |
| `502` | LLM call failed or output never passed validation (see troubleshooting) |

## Code walkthrough

| File | Role |
|---|---|
| `app/config.py` | Loads `GROQ_API_KEY` and `GROQ_MODEL` from `backend/.env`. Real environment variables take priority over the file. |
| `app/schemas/ingestion.py` | `IngestionRequest` (input) and `IngestionResult` (output) Pydantic models. `Literal` types restrict allowed values; `extra="forbid"` rejects unknown fields. |
| `app/services/llm_service.py` | System prompt with category and risk definitions. Wraps the Groq client with Instructor in JSON mode so the reply is parsed into `IngestionResult`; `temperature=0`, up to 2 retries on validation failure. |
| `app/utils/token_counter.py` | Counts tokens with tiktoken `cl100k_base`; falls back to word count × 2 if tiktoken fails. |
| `app/main.py` | FastAPI app, open CORS for local dev, `/health` and `/api/analyze`. Maps any failure to `502`. |
| `tests/test_ingestion.py` | 4 schema unit tests. No endpoint or LLM tests. |

## Known limitations

- When Instructor exhausts its retries it raises `InstructorRetryException`, not Pydantic's `ValidationError`, so validation failures reach the generic `502` message rather than the detailed one in `main.py`.
- `token_metadata` is part of the schema sent to the model, so the model may fill it; the token count is overwritten afterwards.
- A new Groq client is created on each request.
- CORS allows all origins; restrict it before deploying anywhere public.
