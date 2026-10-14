# Frontend

An optional single-page UI for the Ingestion Firewall API. Plain HTML, CSS and JavaScript: no build step, no dependencies. You can skip it entirely and use the API docs at http://127.0.0.1:8000/docs instead.

## Run it

The backend must be running first (see [../backend/README.md](../backend/README.md)). Then, from the project root:

```powershell
python -m http.server 5500 --bind 127.0.0.1 --directory frontend
Start-Process http://127.0.0.1:5500
```

Any static file server works; `http.server` is just the one that ships with Python. Stop it with `Ctrl+C`.

## Using the page

1. Type a message, or click an example button (Greeting, Order tracking, Payment issue, Current time) to fill one in.
2. Click **Analyze message**.
3. The validated JSON appears in the Response panel. **Copy JSON** copies it to the clipboard.

The status line under the button tells you what happened:

| Status | Meaning |
|---|---|
| `Success — response passed the Pydantic schema.` | Request worked |
| `Request failed (422)` | Message was empty or too long (max 5000 characters) |
| `Request failed (502)` | Backend could not get a valid LLM response; see the backend troubleshooting section |
| `Could not connect. Start the FastAPI backend...` | Backend is not running on port 8000 |

## Changing the backend URL

The API address is hardcoded at the top of `script.js`:

```js
const API_URL = "http://127.0.0.1:8000";
```

Change it if you run the backend on another host or port. The backend allows all origins (CORS `*`), so no backend change is needed for local use.

## Files

| File | Purpose |
|---|---|
| `index.html` | Page layout: message box, example buttons, result panel |
| `style.css` | Styling |
| `script.js` | Fills examples, sends `POST /api/analyze`, shows the result or error, handles Copy JSON |
