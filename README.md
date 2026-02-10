# Dynamic API Testing Tool

A browser-based, no-backend API testing tool. Enter a URL to a JSON file that defines your APIs; the tool fetches each endpoint, validates status codes and response keys, and shows a clear pass/fail report.

## Quick Start

1. Open `index.html` in a browser (double-click or drag into Chrome/Edge/Firefox).
2. In the text field, enter a URL to either an **API list JSON** or an **API doc (OpenAPI/Swagger) JSON**:
   - **API list:** `https://your-server.com/apis.json`
   - **API doc:** `https://api.example.com/openapi.json` or any Swagger/OpenAPI spec URL (e.g. `https://petstore.swagger.io/v2/swagger.json`)
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
   - **Download Report (JSON)** saves the last run’s full data (including response bodies). **Download Report (HTML)** saves a standalone HTML table of the current view. **Download QA Report (Excel)** generates a multi-tab spreadsheet (Positive Test Cases, Negative Test Cases, Security Test Cases, Summary Dashboard) for stakeholders; see [QA Report (Excel)](#qa-report-excel) below.

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

## Why APIs might pass in Swagger but fail in the script (and what was fixed)

| Root cause | Fix in script |
|------------|----------------|
| **Headers differ** | Swagger sends `Content-Type` and `Accept` from the spec’s `consumes` and `produces`. The script sets these from the OpenAPI/Swagger spec when converting to API list, so the same media types are sent. |
| **Exact status code** | Swagger may document 200 while the server returns 201 Created. The script accepts **any 2xx** when the expected status is 2xx, so 200/201/204 no longer cause false failures. |
| **List response keys** | For endpoints that return an array (e.g. `GET /users`), mandatory key checks were applied to the root and failed. The script validates **expected keys on the first element** of the array when the response is an array. |
| **Test data collision** | Using the same email/username as in Swagger or a previous run can cause 409 or validation errors. A **dynamic test data generator** produces unique emails and usernames per run (and for schema-driven bodies). |
| **Flaky 5xx / network** | Transient 5xx or network errors were reported as failures. The script **retries up to 2 times** with a 1s delay for 5xx or network errors. |
| **Debugging mismatch** | It was hard to see what the script actually sent. Each result includes **requestSent** (method, URL, headers, body snippet), and the UI has **“Show request sent (for Swagger comparison)”** so you can compare with Swagger. |

## QA Report (Excel)

After running API tests (and optionally **Run security tests**), use **Download QA Report (Excel)** to get a single `.xlsx` file with four sheets, suitable for Excel or Google Sheets:

| Tab | Content |
|-----|--------|
| **Positive Test Cases Report** | APIs tested with valid data: Test Case ID, API Name, Endpoint, HTTP Method, Test Scenario, Request Details, Expected/Actual Result, Status (Pass/Fail), Response Time, Remarks. |
| **Negative Test Cases Report** | APIs tested with invalid or edge-case data (from security run): Test Case ID, API Name, Endpoint, Negative Scenario, Invalid Input/Condition, Expected Error Code, Actual Status Code, Error Message, Status, Remarks. Populated when you run security tests (e.g. invalid auth, wrong method, injection). |
| **Security Test Cases Report** | Security validations: Test Case ID, API Name, Security Scenario, Input/Attack Type, Expected Behavior, Actual Behavior, Status, Remarks. |
| **Summary Dashboard** | Total APIs Tested, Total Positive/Negative/Security Test Cases, Passed/Failed counts, Pass Percentage, High-Risk Failures, Observations & Recommendations. |

Failed test cases are clearly marked (Status = Fail) with remarks. The report uses professional QA wording and is readable by non-technical stakeholders.

## Security Testing (OWASP API & Best Practices)

After running API tests, use **Run security tests** to run automated security checks derived from the same endpoints.

### What is checked

| Check | Purpose |
|-------|--------|
| **Broken Authentication** | Requests **without** auth headers, or with an **invalid token**, should return **401 Unauthorized**. Ensures protected endpoints reject unauthenticated or bad credentials. |
| **BOLA (Broken Object Level Authorization)** | For URLs that include a resource ID (e.g. `/user/1`), a request with a **different ID** (e.g. `/user/99999`) should return **403 Forbidden** or **404 Not Found**, not 200. Prevents accessing other users’ resources. |
| **Invalid / tampered headers** | Requests with `X-Forwarded-For` and `X-Original-URL` are sent to ensure the server does not trust these for routing or auth. No specific status is required; results are reported for review. |
| **Unexpected HTTP method** | Sending the wrong method (e.g. GET to a POST-only endpoint) should return **405 Method Not Allowed**. |
| **Injection (SQL/NoSQL)** | For POST/PUT/PATCH with a body, a payload containing a SQL-like string (e.g. `1' OR '1'='1`) is sent. The API should return **400** or otherwise reject it, and must not return 200 with injected data. |
| **Mass assignment** | A body that includes extra fields such as `role: 'admin'` or `isAdmin: true` is sent. The server should not grant elevated access; the check reports the status for manual review. |
| **Sensitive data leakage** | Every 4xx/5xx response body is scanned for patterns that suggest **information leakage**: stack traces, SQL fragments, internal file paths, or exposed secrets. Any match is reported as a security failure. |

### How to use

1. Run **Run API Tests** as usual (URL or file with your API list).
2. Click **Run security tests**.
3. Review the **Security test results** section: total checks, passed/failed, and per-category results with endpoint and assertion message.

### Important notes

- Security tests use the **last run’s** API list (same URLs, methods, and auth). No token is refreshed; “expired token” is only implied if your last run used an expired token and you re-run security without re-running API tests with a fresh token.
- **Rate limiting / brute-force** is not exercised (no burst of requests) to avoid impacting the server; you can add such checks in your environment if needed.
- **Role-based access** can be tested by defining separate APIs in your JSON for admin vs. user endpoints and running the tool twice with different tokens; security tests will then run for each set.
- Failures are logged in the UI with category, check name, expected vs. actual status, and a short assertion message. Any “sensitive data leakage” finding is explicitly labeled.
