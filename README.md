# Enterprise Structured Ingestion Firewall

A beginner-friendly demo that classifies a customer-support message with a cloud LLM (Groq) and validates the structured result with Pydantic before returning it. The backend is FastAPI; the optional frontend is plain HTML/CSS/JavaScript.

The "firewall" is the schema: whatever the model returns must match a fixed set of fields and allowed values, or it is rejected.

## How it works

```text
Browser (frontend/)  ──POST /api/analyze {"message": "..."}──►  FastAPI (backend/app/main.py)
                                                                  │
                     1. IngestionRequest validates the input  ◄───┤  schemas/ingestion.py
                     2. Groq LLM classifies via Instructor    ◄───┤  services/llm_service.py
                     3. IngestionResult validates the output  ◄───┤  schemas/ingestion.py
                     4. Local token estimate is added         ◄───┘  utils/token_counter.py
                                                                  ▼
                                                    validated JSON response
```

Example: `"My payment was deducted twice. Please help urgently."` becomes

```json
{
  "intent": "payment_issue",
  "risk_profile": "high",
  "priority": "high",
  "requires_action": true,
  "summary": "User reports a duplicate payment deduction and requests urgent assistance.",
  "token_metadata": { "estimated_input_tokens": 10, "note": "Estimated locally; may differ from provider token counts." }
}
```

## Requirements

- Python 3.10+ (tested on 3.12)
- A Groq API key: https://console.groq.com/keys
- Internet access for the LLM request

## Quick start (Windows PowerShell)

```powershell
# 1. Backend setup
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
notepad .env          # set GROQ_API_KEY and GROQ_MODEL=openai/gpt-oss-120b

# 2. Run tests
pytest -q

# 3. Start the API (leave running)
uvicorn app.main:app --reload --port 8000
```

In a second terminal, from the project root:

```powershell
python -m http.server 5500 --bind 127.0.0.1 --directory frontend
Start-Process http://127.0.0.1:5500
```

On macOS/Linux, activate with `source .venv/bin/activate` and use `cp` instead of `copy`.

> **Model note:** `.env.example` still lists `llama-3.3-70b-versatile`, which no longer exists on Groq and makes every request fail with a 502. Set `GROQ_MODEL=openai/gpt-oss-120b` (or another available model) in your `.env`.

## Further docs

| Doc | Covers |
|---|---|
| [backend/README.md](backend/README.md) | Detailed setup, API reference, code walkthrough, troubleshooting the `502` error |
| [frontend/README.md](frontend/README.md) | Running the UI, changing the backend URL, how the page works |

## What this demo does and does not do

- The LLM classifies the message into a fixed set of intents: `greeting`, `order_tracking`, `payment_issue`, `current_time`, `general_question`, `unknown`.
- Instructor asks the model for a Pydantic-shaped result and retries up to 2 times if validation fails; Pydantic validates the final result.
- `tiktoken` estimates input tokens with an approximate tokenizer. It is not Groq billing data.
- It does **not** look up real orders or fetch the current time; it only classifies the request.
- Schema validation checks structure and allowed values, not factual truth or prompt-injection safety.
- No database is required.

## Folder layout

```text
enterprise-ingestion-firewall/
├── backend/
│   ├── app/
│   │   ├── main.py                  # FastAPI app and routes
│   │   ├── config.py                # loads .env
│   │   ├── schemas/ingestion.py     # request/result Pydantic models
│   │   ├── services/llm_service.py  # prompt + Groq/Instructor call
│   │   └── utils/token_counter.py   # tiktoken estimate
│   ├── tests/test_ingestion.py
│   ├── .env.example
│   ├── requirements.txt
│   └── README.md
├── frontend/
│   ├── index.html
│   ├── style.css
│   ├── script.js
│   └── README.md
└── README.md
```
