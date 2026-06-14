# capture-endpoint

Captures all test scenarios for a given API endpoint by reading the source code, hitting the endpoint with each scenario, and producing a structured JSON knowledge base file with a summary.

---

## Purpose and Context

This skill exists to build a **dictionary knowledge base** for the SAP Signavio Process Manager (SPM) dictionary product team.

**Background:**
- SAP Signavio has multiple teams working across multiple branches. When one team needs a resource or information owned by another team, they raise a request and someone has to manually resolve it.
- To reduce this friction, the dictionary team is building a knowledge base that contains real captured API response data — covering queries, API calls, and all related information — so anyone on any team can look up answers instantly.
- This knowledge base will also be fed into an MCP (Model Context Protocol) server, so team members can ask questions directly through tools like Claude Code and get accurate answers backed by real data.
- The data being collected comes from the SPM staging instance at `https://staging.signavio.com`.
- The reference for which endpoints to cover is the Dictionary Knowledge Base API overview at: `https://pages.github.tools.sap/signavio-next/dictionary-knowledge-base/api/overview.html` — focus on **Core Endpoints** only.

**What this means for how you generate responses:**
- Every scenario, note, and observation must be written with a future reader in mind — someone on another team who has never seen this endpoint and needs to understand exactly what it does, what it returns, and what each field means.
- `notes.key_fields_verified` should explain the *business meaning* of each field, not just its value. For example: `"type=ORGANIZATION - indicates this item belongs to the Organizational Units category"` is better than just `"type=ORGANIZATION"`.
- `notes.status_code_observation` should explain *why* the server returns that status in business terms, not just restate the HTTP code.
- `description` for each scenario should read like documentation — clear enough for a new team member to understand the scenario without any other context.
- The final JSON file is the source of truth for the MCP knowledge base, so accuracy and completeness matter more than brevity.

---

## Input

The skill collects four pieces of information from the user at the start of each run. Do NOT assume any defaults — ask the user explicitly for each one.

| Variable | What to ask | Example |
|---|---|---|
| `BASE_URL` | The full base URL of the SPM instance to test against | `https://staging.signavio.com` |
| `ENDPOINT_PATH` | The endpoint path including the HTTP method | `POST /p/glossarycategory` |
| `SOURCE_CODE_PATH` | Absolute path to the local `process-manager` source code on the user's machine | `C:\Users\<name>\...\process-manager` |
| `OUTPUT_DIR` | Absolute path to a folder where the two output files will be saved | `C:\Users\<name>\Desktop\captures` |

The token, required params, and optional params are all discovered automatically (see Steps 1a and 2). The user does NOT need to provide them.

Example invocation:
```
/capture-endpoint
```
The skill will then prompt the user for each of the four inputs above one by one, or all together in a single chat message.

> **Why each input is asked, not hardcoded:** This skill is meant to be portable across users and environments. Hardcoding paths or URLs locks the skill to one machine and one tenant. Asking at runtime makes the skill reusable.

---

## Execution Steps

### Step 1 — Collect Inputs and Set Up Run Identifier

1. Use `AskUserQuestion` (or simple chat prompts) to collect all four required inputs from the user:
   - `BASE_URL` (e.g. `https://staging.signavio.com`)
   - `ENDPOINT_PATH` (e.g. `POST /p/glossarycategory`)
   - `SOURCE_CODE_PATH` (absolute path to `process-manager` source on the user's machine)
   - `OUTPUT_DIR` (absolute path to a folder where output files will be saved)

2. Validate each input before proceeding:
   - `BASE_URL` must start with `http://` or `https://` and have no trailing slash
   - `ENDPOINT_PATH` must start with an HTTP method (GET, POST, PUT, DELETE, PATCH) followed by a path starting with `/`
   - `SOURCE_CODE_PATH` must exist on disk and contain a `platform` subfolder (the SPM module marker)
   - `OUTPUT_DIR` must exist on disk and be writable. If it does not exist, ask the user to create it or provide a different path.

3. Parse `ENDPOINT_PATH` into:
   - `METHOD` (e.g. `POST`)
   - `URL` = `BASE_URL` + path (e.g. `https://staging.signavio.com/p/glossarycategory`)
   - `ENDPOINT_NAME` derived from the path's last segment (e.g. `/p/glossarycategory` → `glossarycategory`)

4. **Generate a unique run identifier** that will be used to tag every resource the skill creates. This is critical for safe cleanup later (see Step 7a):
   ```
   RUN_ID = "CAPTURE-" + <ISO timestamp without colons>
   Example: CAPTURE-20260611T093015
   ```
   Every test resource the skill creates must have this `RUN_ID` embedded in its name (e.g. `P-01 CAPTURE-20260611T093015 Test Item`).

5. **Initialize an empty list** `CREATED_RESOURCES = []` in working memory. Every successful POST/PUT that creates a resource must append the new resource ID to this list. This is the only authoritative source for what the skill is allowed to delete in Step 7a.

---

### Step 1a — Auto-Fetch a Fresh Token

Do NOT ask the user for a token directly. Obtain one automatically by logging in via Playwright:

1. **Ask the user how they want to provide login credentials.** Use `AskUserQuestion` with these options:
   - **Type credentials in chat** — user provides email and password as a chat message; the skill fills the form and submits it
   - **Log in manually in the browser** — the skill opens the login page in Playwright, then waits for the user to log in themselves; once logged in, the skill resumes and extracts the token

2. **If the user chose "Type in chat":**
   - Ask the user to provide their email and password in the chat
   - Use `mcp__playwright__browser_navigate` to open `${BASE_URL}/p/login`
   - Use `mcp__playwright__browser_fill_form` to fill the login form with the provided credentials
   - Submit the form (press Enter or click the login button)

3. **If the user chose "Log in manually":**
   - Use `mcp__playwright__browser_navigate` to open `${BASE_URL}/p/login`
   - Tell the user explicitly: "A browser window is open at the SPM login page. Please log in manually, then reply with `done` in the chat."
   - Wait for the user's confirmation message before proceeding
   - Do NOT auto-fill or submit anything — let the user complete the login themselves

4. After login (regardless of method), confirm the page has loaded past the login screen using `mcp__playwright__browser_wait_for` or by checking the page URL is no longer `/p/login`.

5. Extract the token from cookies using `mcp__playwright__browser_run_code_unsafe`:
   ```javascript
   async (page) => {
     const cookies = await page.context().cookies(BASE_URL);
     const tokenCookie = cookies.find(c => c.name === 'token');
     return tokenCookie ? tokenCookie.value : null;
   }
   ```

6. If the token cookie is not found, fall back to capturing it from a response header using `mcp__playwright__browser_evaluate`:
   ```javascript
   async () => {
     const resp = await fetch(`${BASE_URL}/p/info`, {
       credentials: 'include',
       headers: { 'x-requested-with': 'XMLHttpRequest' }
     });
     return resp.headers.get('x-signavio-id');
   }
   ```

7. Store the extracted token as `TOKEN` for use in all subsequent scenario requests.

8. If login fails (wrong credentials, server down, unexpected page) or the token cannot be extracted, stop and report the error to the user — do not proceed with a stale or empty token.

---

### Step 2 — Discover Scenarios From Source Code

> **IMPORTANT: This step must fully complete before proceeding to Step 2a. If it is interrupted for any reason, restart Step 2 from the beginning. Never proceed with partial source code knowledge — doing so will result in missing scenarios.**

> **ENDPOINT SCOPE LOCK (read before any grep).**
> Scenarios for the endpoint under test MUST be derived ONLY from the handler that serves the exact path + HTTP method the user provided in `ENDPOINT_PATH`. Sibling endpoints that share a path prefix (e.g. `/p/glossary` vs `/p/glossarycategory`, or `POST /p/glossary` vs `POST /p/glossary/<id>`) are different endpoints with different params, validation, and exceptions. Their params must NOT bleed into this run's scenario list.
>
> Concretely:
> - A `grep -r "glossary"` will return BOTH `/p/glossary` and `/p/glossarycategory` handlers. After grepping, the skill MUST identify the one handler whose route mapping (e.g. `@Path`, servlet mapping, registry entry) matches the exact `ENDPOINT_PATH` and METHOD, and discard the rest.
> - The delegation drill-down in Gate 3a is bounded to the call graph rooted at THAT one handler only. Classes that happen to be referenced from a sibling endpoint but not from the chosen handler are out of scope.
> - The PARAM CHECKLIST must list only params the chosen handler reads from the request. Params that exist on a sibling endpoint but not on this one (e.g. `metaDataValues` exists on `/p/glossary` but the category-only fields on `/p/glossarycategory` do not) must NOT appear in the checklist.
> - The skill MUST output a one-line **"Scope-locked to: <METHOD> <path> served by <HandlerClass>"** confirmation before listing the param checklist, so the user can spot if the wrong handler was picked.

Search the source code at the path the user provided in Step 1:
```
${SOURCE_CODE_PATH}
```

Run the following searches to understand what the endpoint does and what inputs it accepts:

1. Search for the endpoint handler — grep for the URL path segment (e.g. `glossary`) in Java controller/handler files:
   ```
   grep -r "glossary" --include="*.java" -l
   ```
   Expect multiple hits when path prefixes are shared. Use the route mapping inside each candidate (annotation, registry, servlet config) to pick the ONE handler that matches `ENDPOINT_PATH` + METHOD exactly. Record the rejected siblings in a one-line note so future-you can verify the choice.

2. Read the handler class **in full** to identify:
   - All accepted parameters and their types
   - Validation logic (required vs optional fields, length limits, format checks)
   - Business rules (duplicate checks, foreign key lookups, enum values)
   - Error conditions and what exceptions are thrown

3. **Auto-derive the param lists from source code** — do NOT ask the user. From the handler class, extract:
   - `REQUIRED_PARAMS`: parameters that have null/empty checks, `@NotNull`, `@NotEmpty`, or throw an exception if missing
   - `OPTIONAL_PARAMS`: all remaining accepted parameters that have no mandatory validation
   - For each param, note its type (String, boolean, int, enum, foreign key reference, JSON string) and any constraints (max length, allowed values, format)
   - Document enum values found in the source so all valid values can be tested as positive scenarios

3a. **Mandatory delegation drill-down — never stop at the handler.**
   The top-level handler is almost never where the real validation lives. It typically delegates to a manager / service / value class. Skipping that drill-down is the #1 cause of missed scenarios (this is exactly how `metaDataValues` types and subtypes were missed in the original `/p/glossary` capture).

   For each parameter in the handler, walk every layer it touches and read each one in full:
   - Manager classes (e.g. `MetaDataManager`, `GlossaryManager`)
   - Value / entity classes (e.g. `MetaDataEntityValue`, `MetaDataDefinitionInfo`, `MetaDataGlossaryItemRevisionValue`)
   - Validator classes
   - Parser/deserializer classes for any JSON-string param
   - Permission / access-control classes invoked before the write

   For each layer, capture:
   - Every type/subtype branch (e.g. metadata types: `text`, `number`, `date`, `boolean`, `link`, `enum`, ...)
   - Every conditional that can change behavior (`if (type == X)` → distinct scenario)
   - Every exception thrown and the condition that throws it (each becomes a negative scenario)
   - Every enum, including its full list of values

   **HARD GATE 3a — Delegation Coverage Audit (cannot be skipped).**
   Before proceeding to Step 3b, the skill MUST output to the chat a structured table of every class/file it has read for this endpoint. The layers below are the *types* to look for; actual class names will vary by endpoint:
   ```
   DELEGATION COVERAGE AUDIT
   =========================
   Layer          | File                                          | Lines read | Why it matters
   handler        | <path>/<Endpoint>Handler.java                 | <range>    | top-level param parsing
   manager        | <path>/<Endpoint>Manager.java                 | <range>    | business rules, persistence
   value/entity   | <path>/<RelevantValue>.java                   | <range>    | per-type validation branches
   parser         | <path>/<JsonStringParser>.java                | <range>    | JSON-string deserialization
   permission     | <path>/<EndpointPermissionChecker>.java       | <range>    | pre-write access control
   validator      | <path>/<RelevantValidator>.java               | <range>    | input format validation
   ...
   ```
   Not every endpoint has all of these layers; include only the ones that exist and add any other layer the source actually delegates to (e.g. event publisher, audit logger, cache invalidator).
   Then call `AskUserQuestion` with the question: "Have I covered every delegated layer for this endpoint?" and options:
   - **Approve — proceed to Step 3b**
   - **A layer is missing — I'll name it** (skill must read the named file in full and re-display the audit)

   The skill MUST NOT proceed to Step 3b without an explicit Approve answer. This gate exists because soft "read everything" instructions were not enforced and layers were skipped in past runs (e.g. nested validation classes were missed during the original `/p/glossary` capture).

3b. **Expand JSON-string params into a sub-checklist.**
   A JSON-encoded string inside a urlencoded body (e.g. `metaDataValues=[{...}]`) is NOT one param — it is a nested schema. For every such param:
   - Identify the parser (e.g. Jackson `readValue`, `JSONObject`)
   - Read the target POJO / DTO class to extract every field, type, and constraint
   - Treat each internal field as its own line in the checklist with its own positive/negative scenarios
   - Generate per-type/per-subtype scenarios — one positive per variant, one negative per variant where the variant adds a constraint

   **HARD GATE 3b — JSON-String Sub-Checklist (cannot be skipped).**
   For EVERY param whose source type is a JSON string, JSON array, or JSON object, the skill MUST output a leaf-by-leaf sub-checklist BEFORE generating any scenarios in Step 2 step 6. Format:
   ```
   JSON-STRING PARAM EXPANSION: <paramName>
   ----------------------------------------
   Parser: <e.g. Jackson readValue into <TargetClass>>
   Target class: <fully qualified class name + line range>

   Leaf fields:
   - <leaf>.<field1> : <type> : <constraints> : [scenarios needed: P-positive-shape, N-bad-shape]
   - <leaf>.<field2> : <type> : <constraints> : [...]
   ...

   Type/Subtype branches (one P + one N per branch):
   - branch=<value1> -> P-XX (valid <value1>) + N-XX (violation of <value1> rule)
   - branch=<value2> -> ...
   ```
   Then call `AskUserQuestion`: "Is the JSON-string sub-checklist for `<paramName>` complete?" with options:
   - **Approve — generate scenarios from this**
   - **Missing fields/branches — I'll name them**

   Repeat for every JSON-string param. The skill MUST NOT generate scenarios for any JSON-string param until its sub-checklist is approved. This gate exists because in past runs JSON-string params were treated as one opaque string and per-type branches were missed (e.g. `metaDataValues` in `/p/glossary` had multiple type variants — text, number, date, boolean, link, enum — that were not captured).

3c. **High-risk packages require exhaustive coverage.**
   `AGENTS.md` flags certain packages as high-risk (currently: `com.signavio.metadata`, `MetaDataEntityValue`, `MetaDataDefinitionInfo`, `MetaDataGlossaryItemRevisionValue`, `MetaDataManager`). If any param of the endpoint under test is backed by one of these packages:
   - Read `.claude/skills/metadata-context/metadata-guide.md` and `.claude/skills/metadata-context/metadata-bugs/causal-chains.md` before writing scenarios
   - Generate at least one positive AND one negative scenario per type/subtype branch — no "covered by one scenario" shortcuts
   - Generate scenarios for every error path documented in `causal-chains.md`
   - Mark these scenarios with `notes.high_risk_package = true` so reviewers know to scrutinize them

4. Search for related test files to discover scenarios already known to the codebase:
   ```
   grep -r "glossary" --include="*Test*.java" -l
   grep -r "glossary" --include="*test*.java" -l
   ```

5. Read those test files to extract:
   - Existing positive scenarios (valid inputs tested)
   - Existing negative scenarios (invalid inputs tested)
   - Any boundary values tested (max length, special characters, etc.)

6. Before moving to Step 3, produce a **complete param checklist** — every top-level param AND every nested field of any JSON-string / object param, with full type/subtype expansion. This list will be used in Step 5a to verify full coverage. Example for `/p/glossary`:
   ```
   PARAM CHECKLIST:
   - title (required, String, max 4000 chars)
   - category (required, foreign key /glossarycategory/<id>)
   - originId (required, String)
   - description (optional, String, rich text)
   - formats (optional, JSON string)
       - format[].type (enum: PNG, SVG, ...)
       - format[].content (String, base64)
   - metaDataValues (optional, JSON string)        <-- expand fully
       - metaDataValues[].id (required if entry present, foreign key to MetaDataDefinition)
       - metaDataValues[].value
           * when definition.type = text     -> String, max length per definition
           * when definition.type = number   -> numeric, range per definition
           * when definition.type = date     -> ISO 8601 date
           * when definition.type = boolean  -> true/false
           * when definition.type = link     -> URL or model reference
           * when definition.type = enum     -> one of definition.allowedValues
           * when definition.type = <other>  -> document any further variants found in source
   - attachments (optional, JSON string)
       - attachment[].filename (String)
       - attachment[].content (base64 or reference)
   - glossaryLinks (optional, JSON string)
       - link[].targetId (foreign key)
       - link[].relation (enum)
   - linked (optional, boolean)
   ```

   Each line that has sub-bullets (`*`) requires its own positive AND negative scenario. The checklist is "complete" only when every leaf has at least one positive and (where validation exists) at least one negative scenario assigned.

   **Positive scenarios** — all valid input combinations:
   - Minimal required fields only
   - Each optional field individually
   - Each optional field combined
   - Each valid enum/category value
   - **Each type/subtype variant of every nested param** (one per leaf in the checklist)
   - Multiple nested entries set at once (e.g. two metadata values together)
   - Boundary values (max length strings, empty strings where allowed)
   - Special characters in string fields (HTML entities, unicode, whitespace)
   - Rich text vs plain text fields
   - Boolean flag variations
   - Large payloads within accepted limits

   **Negative scenarios** — all invalid/missing input combinations:
   - Each required field missing individually
   - Each required field empty
   - Each required field with invalid value/format
   - Invalid foreign key references (non-existent IDs)
   - Duplicate entries (where uniqueness is enforced)
   - Fields exceeding max length
   - Wrong data types (object where array expected, etc.)
   - Malformed JSON in JSON-string fields
   - **For each nested type/subtype: a value that violates that type's constraint** (e.g. text in a number field, unknown enum value, malformed date)
   - **Unknown / non-existent definition ID for nested foreign-key fields**
   - **Empty array vs omitted entirely** for list-typed JSON params (these can behave differently)
   - Wrong HTTP method (GET, PUT, DELETE instead of correct method)
   - Wrong Content-Type header
   - Missing authentication header (`x-signavio-id`)
   - Invalid/expired authentication token
   - Missing `x-requested-with` header
   - Values at and beyond boundary limits

   ---

   **MANDATORY BASELINE BLOCK A — Protocol-level negatives (auto-applies to EVERY endpoint, regardless of handler content).**
   The skill MUST inject these scenarios into the negative list for every run. They are not optional and are not contingent on what the source code says. Each gets the next free `N-XX` code:
   - Wrong HTTP method × every other verb in {GET, POST, PUT, DELETE, PATCH} minus the actual method (one scenario per verb, expected 405)
   - Missing `x-signavio-id` header AND missing auth cookies (expected 401)
   - Invalid `x-signavio-id` token (32 z-chars or similar) (expected 401)
   - Missing `x-requested-with` header (expected 403)
   - Wrong `Content-Type` (e.g. `application/json` for a urlencoded endpoint) (expected 400)
   - Empty body (no params at all) (expected 400)
   - Malformed body (truncated urlencoded / invalid JSON) (expected 400)

   **MANDATORY BASELINE BLOCK B — String-shape edge cases (auto-applies to EVERY String param the handler accepts).**
   For every param whose type is `String` (top-level OR nested inside a JSON-string param), the skill MUST generate one scenario per shape below. Each gets the next free `P-XX` or `N-XX` code depending on whether validation rejects it:
   - Empty string `""`
   - Whitespace only `"   "`
   - Single character `"a"`
   - Boundary at MAX length (exactly the documented limit)
   - Boundary at MAX+1 (one over the limit, expect 400)
   - Unicode mix (German umlauts, CJK characters, emoji)
   - HTML / script chars (`<script>alert(1)</script>`, `&amp;`, `<>`)
   - Literal string `"null"` (tests whether the server treats it as the null value)
   - Numeric-looking string `"12345"` (tests type coercion)
   - Leading/trailing whitespace (tests normalization)
   - For URL-typed strings: `javascript:`, `data:`, `vbscript:` protocol prefixes (XSS check, expect 400)

   These two baseline blocks are the floor, not the ceiling. Source-derived scenarios from steps 1-5 are added on top.

Assign each scenario a code:
- Positive: `P-01`, `P-02`, ... (not `H-` for happy path)
- Negative: `N-01`, `N-02`, ...

---

### Step 2a — User Scenario Review and Approval Gate

> **This step is mandatory. The skill must not execute any scenario against the live endpoint until the user has explicitly approved the scenario list.**

Before any HTTP requests are sent in Step 3:

1. Display the **complete scenario list** to the user in the chat as a numbered, readable summary. Format:
   ```
   SCENARIO LIST FOR REVIEW (N total)
   =====================================

   PARAM CHECKLIST FROM SOURCE:
   - <param>: <required|optional>, <type>, <constraints>
   ...

   POSITIVE (P)
   P-01: <name> - <one-line description> [params: a, b, c]
   P-02: <name> - <one-line description> [params: a, b, c]
   ...

   NEGATIVE (N)
   N-01: <name> - <one-line description> [params: a, b, c]
   ...
   ```

2. Use `AskUserQuestion` to ask the user:
   - **Approve as-is** — proceed to Step 3 with the list above
   - **Add scenarios** — user dictates additional scenarios to add (skill assigns next P-XX / N-XX codes and adds them to the list)
   - **Remove scenarios** — user lists scenario codes to drop
   - **Edit scenarios** — user lists changes (e.g. rename P-05, change body of N-03)

3. If the user picks anything other than "Approve as-is", apply the requested changes, redisplay the updated list, and ask again. Loop until the user approves.

4. Only after explicit approval, proceed to Step 3. Record the approval in working memory so it is clear which scenario set was authorized for execution.

5. Never silently add new scenarios after approval. If Step 5a (gap check) discovers a missing parameter, surface the proposed new scenario(s) to the user for a second mini-approval before executing them.

---

### Step 3 — Execute Each Scenario

For **every single scenario** identified in Step 2, execute it individually. Do not batch or skip any scenario.

#### Tool Selection

Use the right tool for each situation:

- **Bash tool (curl)** — default for all API calls. Fast, direct, captures raw HTTP response including headers and body.
- **MCP Playwright** (`mcp__playwright__*`) — use when:
  - The endpoint requires a browser session/cookie that curl cannot replicate
  - You need to navigate the UI to obtain a valid session token first
  - The request must originate from a browser context (e.g. CSRF token tied to browser session)
  - Use `mcp__playwright__browser_navigate` to load the app, `mcp__playwright__browser_network_requests` to capture real requests the UI makes, and `mcp__playwright__browser_snapshot` to inspect page state
- **MCP Chrome DevTools** (`mcp__chrome-devtools__*`) — use when:
  - You need to inspect actual network request/response pairs at the browser level
  - You need to verify cookies or session headers the browser attaches automatically
  - Use `mcp__chrome-devtools__list_network_requests` to see all requests, `mcp__chrome-devtools__get_network_request` to get full request and response details for a specific call
  - Use `mcp__chrome-devtools__take_snapshot` to verify the UI state after an action

**Decision rule:** Start with curl. If curl returns 401/403 unexpectedly or the endpoint behaves differently than expected, switch to Playwright/Chrome DevTools to capture how the browser makes the same request, then replicate those exact headers/cookies in curl.

---

For each scenario:

1. Build the curl command with:
   - Correct HTTP method (`-X METHOD`)
   - Headers: `Content-Type: application/x-www-form-urlencoded`, `Accept: application/json`, `x-requested-with: XMLHttpRequest`, `x-signavio-id: TOKEN`
   - Body params using `-d` flags
   - Include `-i` flag to capture response headers alongside body
   - Include `-s` flag to suppress progress output

   Example:
   ```bash
   curl -s -i -X POST "https://staging.signavio.com/p/glossary" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -H "Accept: application/json" \
     -H "x-requested-with: XMLHttpRequest" \
     -H "x-signavio-id: TOKEN" \
     -d "title=P-01 Test Item" \
     -d "category=/glossarycategory/CATEGORY_ID" \
     -d "originId=GLOSSARY_ID"
   ```

2. Execute using the appropriate tool (curl via Bash, or MCP Playwright/Chrome DevTools if browser context is needed).

3. Parse the response:
   - Extract HTTP status code and status text from the first response line
   - Extract all response headers (key: value pairs)
   - Extract and parse the response body as JSON where possible
   - If using Chrome DevTools, use `mcp__chrome-devtools__get_network_request` with `requestFilePath` and `responseFilePath` to save large bodies to file, then read them

4. Verify the response makes sense for the scenario:
   - Positive scenarios should return 2xx status codes
   - Negative scenarios should return 4xx or 5xx status codes
   - If a positive scenario returns an error, or a negative scenario returns success, flag it with a note and investigate why before continuing
   - If the response differs from expected, use `mcp__chrome-devtools__list_console_messages` to check for browser-side errors

5. If a scenario fails unexpectedly (network error, timeout, server crash), retry once using the alternative tool (switch from curl to Playwright or vice versa), then document the failure in the scenario notes.

6. **Token refresh on session expiry.** If any request returns HTTP `401 Unauthorized` and the scenario is NOT one of the explicit "missing/invalid auth" negative scenarios:
   - Treat it as a session expiry, not a real test failure
   - Re-run Step 1a to obtain a fresh `TOKEN`
   - Retry the failing scenario once with the new token
   - If the retry still returns 401, stop and report the issue to the user — something other than session expiry is wrong
   - Do NOT count the expired-session 401 as the scenario's actual response — it must be replaced with the result of the successful retry

7. **Track every resource the skill creates.** For any scenario whose response includes a created resource (typical shape: `{ "id": "...", "rep": { "id": "..." } }` for POST, or a Location header on 201/302):
   - Extract the resource ID from the response
   - Append `{ "scenario": "P-XX", "id": "<resource_id>", "type": "<resource_type>" }` to `CREATED_RESOURCES`
   - Verify the resource name embeds the `RUN_ID` generated in Step 1 — if not, that is a bug in the scenario body and must be fixed before continuing

8. **Never modify or delete resources the skill did not create.** When testing PUT/PATCH/DELETE endpoints, only target IDs in `CREATED_RESOURCES`. If a scenario *requires* an existing resource as a foreign key (e.g. a parent category), the skill must first POST a fresh disposable parent (tagged with `RUN_ID`) and reference that — never reuse arbitrary IDs found in the staging tenant.

---

### Step 4 — Structure Each Response

For each executed scenario, build a JSON object in this exact format:

```json
{
  "scenario": "P-01",
  "name": "Short descriptive name",
  "description": "One sentence explaining what this scenario tests.",
  "expected_status": 200,
  "request": {
    "method": "POST",
    "url": "{{base_url}}/p/glossary",
    "headers": {
      "Content-Type": "application/x-www-form-urlencoded",
      "Accept": "application/json",
      "x-requested-with": "XMLHttpRequest",
      "x-signavio-id": "{{token}}"
    },
    "body": {
      "mode": "urlencoded",
      "params": {
        "title": "P-01 CAPTURE-20260611T093015 Test Item",
        "category": "/glossarycategory/{{orgUnitsCategoryId}}",
        "originId": "{{glossaryId}}"
      }
    }
  },
  "response": {
    "status": 200,
    "statusText": "OK",
    "headers": {
      "content-type": "application/json;charset=utf-8",
      "cache-control": "no-cache, no-store, must-revalidate, max-age=0"
    },
    "body": { }
  },
  "notes": {
    "status_code_observation": "HTTP 200 returned for successful creation.",
    "key_fields_verified": [
      "field=value - observation about this field"
    ],
    "template_variables": {
      "{{token}}": "{{token}}",
      "{{glossaryId}}": "{{glossaryId}}"
    }
  }
}
```

Rules:
- `scenario` — scenario code (P-01, N-01, etc.)
- `name` — 5-8 word descriptive title
- `description` — one sentence, what is being tested and what is expected
- `expected_status` — the HTTP status code the scenario is designed to produce (e.g. `200` for positive, `400` for missing-required-field, `401` for missing-auth). After execution, compare this to `response.status`; if they differ, flag the scenario for re-investigation.
- `request.body.params` — only include params actually sent; omit params intentionally left out for negative scenarios. Every created resource name must embed the `RUN_ID` (e.g. `"P-01 CAPTURE-20260611T093015 Test Item"`).
- `response.body` — exact parsed JSON response from the server; if response is empty string use `""`. Real response data is the whole point of this knowledge base — readers must be able to see what the server actually returns. Apply the array cap below to keep file size manageable, but do NOT truncate strings, drop object keys, or replace lists with summaries.

**Body policy (mandatory): array-cap-5, applied recursively.**

Teammates need real response bodies in every scenario, not summaries. The only allowed shape change is for arrays that are too long to be useful in full — and even then we keep real items, not a synthetic summary.

The rule:
- An **array longer than 5 items** keeps its first 5 items in full, then appends ONE marker object: `{"__dropped__": <count>, "__note__": "array had <total> items; kept first 5 in full"}`. The marker is appended as the last array element so the result is still a valid array of length 6.
- An **array of 5 or fewer items** is kept verbatim — every item, no marker.
- **Strings** are kept in full, no character truncation. (Long echoed inputs that show up in error responses ARE useful — readers want to see the exact server response.)
- **Objects** keep all keys. No "first 8 keys" cap.
- The cap applies **recursively** into nested values. If an object has a property whose value is a 2000-item array, that nested array gets the array-cap-5 treatment; the object itself is untouched.

When the policy is applied, the top-level KB object MUST include a `body_policy` field documenting the rule verbatim, so a reader of the file can interpret `__dropped__` markers without consulting the skill source.

Privacy note: when an unintended fall-through listing returns data created by other users (e.g. a wrong-method scenario that lands on a listing handler), the array-cap-5 rule still applies — but PII redaction (Step 5) MUST run first so `@sap.com` emails and similar identifiers in those first 5 kept items are replaced with `{{user_email}}` placeholders before the file is written.
- `notes.key_fields_verified` — list the most important fields from the response and what they confirm, using ` - ` as separator (e.g. `"type=ORGANIZATION - derived from category"`)
- `notes.status_code_observation` — one sentence explaining the HTTP status returned and why
- All string values must be ASCII only — no em dashes, curly quotes, or non-ASCII characters

**The POST file must contain ONLY POST request/response data. Do not include:**
- `stored_representation_via_get` or any GET response bodies (GET endpoint coverage belongs in a separate `/capture-endpoint` run)
- Top-level `source_reference`, `key_findings`, `schema_notes`, or any "context" sections
- Per-scenario `notes.source_reference` block
- Java source file paths, line numbers, handler/service class names
- Behavioral findings or platform-team notes (those are separate review artifacts, not part of the captured data)

The output should read as a pure HTTP capture: each scenario is one request and its response, plus the minimum metadata (`expected_status`, `actual_status`, `status_match`, `high_risk_package`).

---

### Step 5 — Replace Credentials With Placeholders

After all scenarios are structured, scan every scenario's request and response for hardcoded credential values and replace with named placeholders:

| Real value | Placeholder |
|---|---|
| `BASE_URL` (e.g. `https://staging.signavio.com`) — appears in `request.url` | `{{base_url}}` |
| The token value provided by user | `{{token}}` |
| User ID found in response bodies | `{{userId}}` |
| Glossary/origin ID provided by user | `{{glossaryId}}` |
| Each category ID | `{{categoryNameId}}` — derive name from category type |

Rules:
- Replace in: `request.url` (full base URL → `{{base_url}}`), request headers, request body params, response body fields (`author`, `glossaryId`, `granted_revision_user`, `category`)
- Do NOT replace: per-scenario item IDs, revision IDs, requestIds — these are captured test output, not credentials
- All replacements must be exact string matches
- The KB file MUST include a top-level `"base_url"` field with the actual value (e.g. `"https://staging.signavio.com"`) so consumers can resolve `{{base_url}}` without external context

---

### Step 5a — Mandatory Parameter Coverage Gap Check

> **This step is mandatory and must not be skipped. It exists specifically to catch parameters missed due to partial source code reads or interrupted discovery.**

Re-read the handler class **and every delegated class identified in Step 3a** one final time. Using the **PARAM CHECKLIST** produced at the end of Step 2 — including every nested leaf and every type/subtype branch — verify that every leaf is covered.

**Coverage is leaf-level, not param-level.** A top-level param like `metaDataValues` is NOT "covered" because one scenario sent some metadata. Each type/subtype branch (text, number, date, boolean, link, enum, ...) must have its own scenario.

1. For each **leaf** in the checklist (every bullet, including nested `*` entries), confirm:
   - At least one **positive scenario** tests it with a valid value
   - At least one **negative scenario** tests it with an invalid value (for required leaves, also one with the value missing)
   - For enum-typed leaves: every enum value listed in source has at least one positive scenario
   - For type-discriminated leaves (e.g. `metaDataValues[].value` varies by `definition.type`): every type variant has its own positive scenario AND a negative scenario that violates that type's constraint

2. If any leaf has no scenario covering it:
   - Create the missing scenario(s) immediately and append to the scenario list
   - Assign the next available `P-XX` or `N-XX` code
   - Surface the additions to the user (mini-approval, see Step 2a) before executing them
   - Execute the new scenario(s)

3. Report the coverage check result as a tree, not a flat list, so nested coverage is visible. Example:
   ```
   PARAM COVERAGE CHECK:
   - title: covered (P-01, N-01, N-02, N-03)
   - category: covered (P-01, N-04)
   - originId: covered (P-01, N-05)
   - description: covered (P-02)
   - formats: covered
       - format[].type: covered (P-03, P-04, N-06 invalid type)
       - format[].content: covered (P-03, N-07 malformed base64)
   - metaDataValues: covered
       - metaDataValues[].id: covered (P-05, N-08 unknown definition ID)
       - metaDataValues[].value:
           * type=text:    covered (P-06 positive, N-09 oversized text)
           * type=number:  covered (P-07 positive, N-10 text-in-number)
           * type=date:    covered (P-08 positive, N-11 malformed date)
           * type=boolean: covered (P-09 positive, N-12 non-boolean)
           * type=link:    covered (P-10 positive, N-13 malformed URL)
           * type=enum:    covered (P-11 positive, N-14 value-not-in-allowed-list)
   - attachments: covered ...
   - glossaryLinks: covered ...
   - linked: covered (P-12)
   ALL LEAVES COVERED - proceeding to Step 6
   ```

4. Only proceed to Step 6 when **every leaf** in the checklist has at least one scenario. If a leaf cannot be tested (e.g. requires a plugin or system mode that is not installed), document it explicitly with a note explaining why it was skipped, and call it out in the final report.

5. If the endpoint is backed by a **high-risk package** (Step 3c), additionally verify that every error path in `metadata-bugs/causal-chains.md` is covered by a negative scenario. List uncovered error paths in the gap report and treat them as missing leaves.

6. **HARD GATE 5a — Coverage Floor (cannot be skipped).**
   Before proceeding to Step 6, the skill MUST verify all three of the following AND cannot proceed unless all three pass:
   - Every leaf in the PARAM CHECKLIST has at least one positive AND (where validation exists) one negative scenario
   - Every scenario in **MANDATORY BASELINE BLOCK A — Protocol-level negatives** is present in the negative list
   - Every scenario in **MANDATORY BASELINE BLOCK B — String-shape edge cases** has been generated for every String param (top-level and nested)

   If ANY of the three fails, the skill MUST:
   - Output a `GAPS DETECTED` block listing each missing item by category (leaf / protocol baseline / string-shape baseline)
   - Generate the missing scenarios, assign next free codes
   - Call `AskUserQuestion`: "Add the listed gap-fill scenarios?" with options **Approve & continue** / **Edit list first**
   - Re-run the gate after the additions

   The skill MUST NOT proceed to Step 6 with any of the three checks unsatisfied. This gate exists because in past runs (e.g. the original `/p/glossary` capture), baseline protocol negatives and String-shape edge cases were "mentioned in instructions" but never enforced, and were partially missed.

---

### Step 6 — Validate the Output

Before writing files, run these checks:

1. **JSON validity** — the full array must parse without errors
2. **Scenario count** — confirm total count matches the scenario list from Step 2
3. **No missing required fields** — every scenario must have `scenario`, `name`, `description`, `request`, `response`, `notes`
4. **No non-ASCII characters** — zero occurrences of any character outside the ASCII range (0x00-0x7F)
5. **No raw credentials** — confirm the token, JSESSIONID, tenant ID, LB-route value, user email, and password do NOT appear anywhere in the output
6. **Intentional gaps are documented** — negative scenarios that deliberately omit a field must have that omission reflected in `notes.key_fields_verified`
7. **No prohibited fields** — the output MUST NOT contain any of:
   - `stored_representation_via_get` (GET data leaking into POST file)
   - `source_reference` (top-level OR per-scenario `notes.source_reference`)
   - `key_findings` (top-level)
   - `schema_notes` (top-level)
   - Java file paths or `<file>:<line>` references anywhere
8. **`{{base_url}}` placeholder** — every `request.url` MUST start with `{{base_url}}`; the literal `BASE_URL` value (e.g. `https://staging.signavio.com`) MUST NOT appear inside any scenario URL. The KB MUST have a top-level `"base_url"` field giving the real value.
9. **Body policy applied** — every `response.body` that is an array of length > 6 is invalid (the cap is 5 real items + 1 marker = 6 max); every array of length 6 must have `__dropped__` and `__note__` keys on its last element; no string body has been truncated; no object body has had keys removed. The top-level KB object must contain a `body_policy` field describing the rule.

Report the validation result before writing files. If any check fails, fix it first.

---

### Step 7 — Write Output Files

Write two files to the directory the user provided in Step 1:
```
${OUTPUT_DIR}
```

**File 1 — Full knowledge base (responses):**
Filename: `<endpoint_name>-<METHOD>-response.json`
Example: `glossarycategory-POST-response.json`
Path: `${OUTPUT_DIR}\<endpoint_name>-<METHOD>-response.json`
Content: A JSON array of all scenario objects (request + response + notes), pretty-printed with 2-space indentation.

**File 2 — Scenarios summary:**
Filename: `<endpoint_name>-<METHOD>-Scenarios.txt`
Example: `glossarycategory-POST-Scenarios.txt`
Path: `${OUTPUT_DIR}\<endpoint_name>-<METHOD>-Scenarios.txt`
Content:
```
<METHOD> /<path> - Test Cases Summary (N scenarios)
====================================================

Run ID: CAPTURE-<timestamp>
Captured at: <iso-timestamp> (<environment>)

POSITIVE SCENARIOS
------------------------------
P-01: <description> -> <actual_status> (expected <expected_status>) [OK|MISMATCH]
P-02: ...

NEGATIVE SCENARIOS
------------------------------
N-01: <description> -> <actual_status> (expected <expected_status>) [OK|MISMATCH]
N-02: ...
```

The txt summary MUST NOT include:
- A `KEY FINDINGS` section
- Behavioral observations, validation gaps, or platform-team commentary
- Source code references

It is a flat per-scenario index, nothing more. Behavioral commentary belongs in a separate review document, not in the captured data summary.

If a file with the same name already exists in `OUTPUT_DIR`, ask the user before overwriting.

**Folder hygiene (mandatory):** During execution the skill may create intermediate files in `OUTPUT_DIR` (e.g. helper Python scripts like `build_kb.py` / `run_scenarios.py` / `cleanup*.py`, raw capture dumps like `all_scenarios.json`, recovery files like `positives_get.json`, ID trackers like `created_ids.json`, cleanup audit reports). After Step 7a (Safe Cleanup) completes successfully, the skill MUST delete every intermediate file in `OUTPUT_DIR`, leaving ONLY:
- `<endpoint_name>-<METHOD>-response.json`
- `<endpoint_name>-<METHOD>-Scenarios.txt`

Before deleting intermediates:
1. List every file in `OUTPUT_DIR` and present the deletion list to the user via `AskUserQuestion` ("Delete all intermediate files now / Keep them for debugging").
2. If the user picks "Keep", skip deletion but tell them which files are intermediates so they can clean up later.
3. Never delete files outside `OUTPUT_DIR`. Never use bulk patterns like `rm *` — list explicit filenames.

---

### Step 7a — Safe Cleanup of Created Resources

> **Critical safety step. The staging tenant is shared across many users at SAP. Cleanup must delete ONLY what the skill itself created in this run, never anything else.**

1. **Show the user the cleanup plan first.** Display the full `CREATED_RESOURCES` list along with the `RUN_ID`:
   ```
   CLEANUP PLAN
   ============
   RUN_ID: CAPTURE-20260611T093015
   Resources created by this run (N total):
   - P-01: <type> <id> (name embeds RUN_ID: yes/no)
   - P-02: <type> <id> (name embeds RUN_ID: yes/no)
   ...
   ```

2. **Use `AskUserQuestion`** with these options:
   - **Delete all listed resources** — proceed with cleanup
   - **Keep all resources** — skip cleanup; leave the test data in place
   - **Delete only some** — user picks specific scenario codes to clean up

3. **Cleanup safety rules** (these apply regardless of user choice):
   - Only IDs in `CREATED_RESOURCES` are eligible for deletion
   - Before deleting any ID, fetch the resource and verify its name still contains the `RUN_ID` string. If it does not, abort the deletion and warn the user — that resource may have been modified by another user
   - Never delete by name match, prefix scan, or "all resources of type X" — always delete by exact ID from the tracked list
   - Never run a bulk DELETE or any endpoint that operates on more than one resource at a time
   - If a DELETE call returns a non-2xx response, log it and continue with the next ID — do not retry aggressively
   - Never touch any user, role, permission, workspace, or tenant-level setting

   **When in doubt, do NOT delete — leave the resource in place.** Deleting the wrong thing on shared staging is far worse than leaving a stray test artifact behind. Any of the following situations means *skip this resource and move on*:
   - The resource ID is missing or malformed in `CREATED_RESOURCES` (e.g. the POST returned 200 but no extractable ID, or the ID was captured as `null`/`undefined`/empty string)
   - A pre-delete fetch returns 404, 403, or any non-2xx — the resource may not exist, may already be deleted, or may belong to someone else; do NOT attempt to "find it by other means"
   - The fetched resource has no name, an empty name, or a name that does NOT contain `RUN_ID` — even if the ID matches our list, treat it as ambiguous and skip
   - The resource type returned by the fetch does not match the type we recorded in `CREATED_RESOURCES` (ID collision across types)
   - The fetch response shape is unexpected (e.g. returns a list instead of a single resource) — never iterate the list looking for "ours"

   For every skipped resource, append an entry to the cleanup report under "Resources skipped (left in place for safety)" with the scenario code, ID, and the reason. Tell the user explicitly that some resources were left behind so they can decide whether to investigate manually. **Never substitute a different deletion target** — if the originally tracked ID can't be safely deleted, the answer is to leave it, not to find something else to delete instead.

4. **Modification safety rules** (apply during Step 3 for PUT/PATCH scenarios):
   - PUT/PATCH scenarios may only target IDs in `CREATED_RESOURCES`
   - If the endpoint under test is a PUT/PATCH on a resource the skill did not create, the skill must first POST a fresh disposable resource (tagged with `RUN_ID`) and run the modification scenarios against that, never against pre-existing tenant data

5. After cleanup, report which IDs were successfully deleted, which were skipped, and which failed. Update `CREATED_RESOURCES` to reflect the final state.

---

### Multi-Tenant Safety Guardrails (apply throughout the run)

The staging instance is shared across many users at SAP. The skill must follow these rules at every step:

- **Tag everything with `RUN_ID`.** Every resource the skill creates must have its name include the `RUN_ID` generated in Step 1. This makes the skill's footprint identifiable and reversible.
- **Track everything in `CREATED_RESOURCES`.** This list is the only authoritative cleanup target.
- **Never delete or modify resources the skill did not create.** Foreign-key dependencies must be satisfied by creating fresh disposable parents, not by reusing arbitrary tenant data.
- **Never run bulk operations.** No "delete all", no "list and delete matching", no operations that affect more than one resource per call.
- **Never touch tenant-wide state.** Do not edit users, roles, permissions, workspaces, feature flags, system settings, or anything outside the resource type the endpoint targets.
- **Never log in as a user other than the one who started the run.** Token refresh in Step 3 reuses the same login flow from Step 1a.
- **If anything looks unexpected, stop and ask.** If a response includes data clearly belonging to other users (resources with names not matching `RUN_ID`, references to other authors, etc.), do not act on it — surface it to the user and wait for direction.
- **When in doubt, leave it alone.** A stray test artifact left on staging is harmless. Deleting the wrong resource is not. If the skill cannot positively identify a resource as one of its own (no `RUN_ID` in the name, missing/blank name, ambiguous ID, unexpected response shape), skip it and report it as skipped — never substitute a different deletion target to "make up" for the failed cleanup.

---

### Step 8 — Final Report

After writing both files, report to the user:

```
Done.

Files written to ${OUTPUT_DIR}:
  - <endpoint_name>-<METHOD>-response.json  (N scenarios)
  - <endpoint_name>-<METHOD>-Scenarios.txt

RUN_ID: CAPTURE-20260611T093015

Positive scenarios: XX
Negative scenarios: XX
Total: XX

Validation:
  - JSON valid: YES
  - Non-ASCII chars: 0
  - Raw credentials: NONE FOUND
  - Missing fields: NONE (or list any intentional gaps)
  - expected_status matches response.status for all scenarios: YES (or list mismatches)

Cleanup:
  - Resources created: XX
  - Resources deleted: XX
  - Resources kept (user chose to keep): XX
  - Resources skipped (left in place for safety): XX
      * <scenario_code> <id> - <reason e.g. "no RUN_ID in name", "fetch returned 404", "ID was empty">
  - Resources failed to delete: XX (with IDs and reasons)
```
