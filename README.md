# Dynamic API Testing Tool

A browser-based, no-backend API testing tool. Enter a URL to a JSON file that defines your APIs; the tool fetches each endpoint, validates status codes and response keys, and shows a clear pass/fail report.

## Quick Start

1. Open `index.html` in a browser (double-click or drag into Chrome/Edge/Firefox).
2. In the input field, enter the URL of your API list JSON:
   - **Remote:** `https://your-server.com/apis.json`
   - **Local:** Use a path like `./sample-api-list.json` only if you serve the folder (e.g. `npx serve .` or VS Code Live Server). Browsers block `file://` fetch to other files.
3. Click **Run API Tests**.
4. Review the summary and per-API results; expand cards to see response bodies. Use **Download Report (JSON)** or **(HTML)** to save the report.

## Sample JSON

Use `sample-api-list.json` as a reference. It targets [JSONPlaceholder](https://jsonplaceholder.typicode.com/) so you can run it as-is if the JSON is loaded from a URL (e.g. host the file or use a raw URL from GitHub/Gist).

### JSON Schema

The tool expects a **JSON array** of API definitions, or an object with one of these keys: `apis`, `apiList`, `endpoints`.

Each API object can have:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Recommended | Display name in the report |
| `method` | string | Yes | `GET`, `POST`, `PUT`, `DELETE`, `PATCH` |
| `endpoint` or `url` | string | Yes | Full request URL |
| `headers` | object | No | Key-value headers (defaults include `Content-Type: application/json`) |
| `body` or `requestBody` | object or string | No | Request body for POST/PUT/PATCH |
| `expectedStatusCode` | number | No | Expected HTTP status (e.g. 200, 201). If omitted, only `response.ok` is checked |
| `expectedResponseKeys` or `expectedKeys` | string[] | No | Dot-notation keys that must exist in response JSON (e.g. `["data", "user.id"]`) |

Example single API:

```json
{
  "name": "Get User",
  "method": "GET",
  "endpoint": "https://api.example.com/users/1",
  "headers": { "Authorization": "Bearer token123" },
  "expectedStatusCode": 200,
  "expectedResponseKeys": ["id", "name", "email"]
}
```

## How the Script Works

1. **Load config**  
   On Submit, the script uses `fetch(jsonUrl)` to get the JSON. The response is parsed; the list of APIs is taken from the root array or from `apis` / `apiList` / `endpoints`.

2. **Run tests**  
   For each API definition, the script:
   - Builds `fetch(url, { method, headers, body })`.
   - Sends the request and measures time with `performance.now()`.
   - Reads response status and body (text then optional `JSON.parse`).
   - Compares status to `expectedStatusCode` (or `response.ok` if not set).
   - If `expectedResponseKeys` is present, checks that each key path exists in the parsed JSON (e.g. `user.id` → `body.user.id`).
   - Sets `pass` and `reason` (e.g. wrong status, missing keys, or network error).

3. **Report**  
   Results are shown in the UI:
   - Summary: total, passed, failed, total execution time.
   - One card per API: name, method, URL, expected vs actual status, response time, pass/fail, and optional failure reason.
   - Click a card to expand; use “Show response body” to see the raw/parsed response.
   - **Download Report (JSON)** saves the last run’s full data (including response bodies). **Download Report (HTML)** saves a standalone HTML table of the current view.

4. **Error handling**  
   Invalid URL, non-200 on config fetch, missing or empty API list, and per-request network/errors are caught and shown in the error banner; the loader is hidden and the button re-enabled.

## Tech Stack

- **HTML** – Form (URL input + Submit), loader, results area, summary, and API cards.
- **CSS** – Layout, dark theme, method/result colors, expand/collapse, responsive summary grid.
- **JavaScript** – No frameworks; `async/await`, single global for “last run” data, and modular functions for fetch config, execute one API, validate keys, render summary/cards, and export JSON/HTML.

## Extending for Future Assertions

- **More validations:** In `executeApi()`, after status and `expectedResponseKeys`, add checks (e.g. value of a field, array length, status in payload). Set `result.pass = false` and `result.reason` when a check fails.
- **Custom headers per run:** Add an optional “Headers” textarea in the UI and merge its JSON into each request’s `headers`.
- **Environment/base URL:** Let the user set a base URL and in the JSON use paths like `"/users/1"`; in `executeApi()` concatenate base + path.
- **Retries:** Wrap `fetch` in a loop with a delay and retry count before pushing the result.

## CORS Note

APIs must allow browser requests (CORS). If the API does not send proper CORS headers, the request will fail in the browser. For same-origin or CORS-enabled APIs, the tool works as-is.
