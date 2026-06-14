# capture-endpoint

A Claude Code plugin that captures every test scenario for an SAP Signavio Process Manager (SPM) API endpoint by reading the source, hitting the endpoint with each scenario, and producing a structured JSON knowledge base file.

Built for the SAP Signavio dictionary knowledge base — the captured data feeds an MCP server so any team can look up endpoint behavior backed by real responses.

---

## What you get

Two skills are bundled:

- **`capture-endpoint`** — the main skill. Run `/capture-endpoint` and it will walk you through capturing one endpoint end-to-end (positive + negative scenarios, baseline protocol negatives, string-shape edge cases).
- **`metadata-context`** — a regression-history reference for the SPM metadata package. The capture skill loads it automatically when the endpoint under test touches metadata classes, so historical bug patterns become part of the negative scenario set.

---

## Install

Inside Claude Code:

```
/plugin marketplace add https://github.tools.sap/I775890/cc-capture-endpoint.git
/plugin install capture-endpoint
```

To pull updates later:

```
/plugin install capture-endpoint
```

---

## Use

```
/capture-endpoint
```

The skill will ask for four inputs:

| Input | Example |
|---|---|
| `BASE_URL` | `https://staging.signavio.com` |
| `ENDPOINT_PATH` | `POST /p/glossary` |
| `SOURCE_CODE_PATH` | absolute path to your local `process-manager` checkout |
| `OUTPUT_DIR` | absolute path to a folder for the two output files |

It will then auto-fetch a session token via Playwright login (you choose between typing credentials in chat or logging in manually in the opened browser), discover scenarios from source, ask for your approval on the scenario list, execute every scenario, and write two files:

- `<endpoint>-<METHOD>-response.json` — the full knowledge base
- `<endpoint>-<METHOD>-Scenarios.txt` — flat per-scenario index with status comparison

After capture it offers safe cleanup of test resources it created (only resources tagged with the run's `RUN_ID` are eligible — never anything else on the shared staging tenant).

---

## Safety guarantees

The plugin is designed to run safely on shared SAP staging:

- **Run tagging** — every resource the skill creates embeds a unique `RUN_ID` in its name. Cleanup uses this tag to prove ownership before deleting.
- **Allow-list cleanup** — only IDs the skill itself created (tracked in `CREATED_RESOURCES`) are eligible for deletion. No bulk operations, no name-prefix scans, no "delete all of type X."
- **When in doubt, leave it alone** — if a resource cannot be positively identified as ours (missing name, RUN_ID stripped, ID collision, unexpected response shape), it is skipped and reported, never substituted with a different deletion target.
- **No tenant-wide writes** — the skill never touches users, roles, permissions, workspaces, feature flags, or system settings.
- **Credential handling** — tokens, cookies, and credentials are replaced with `{{token}}`, `{{base_url}}`, etc. placeholders in the output file. Output validation rejects raw credentials before writing.

---

## What's in the captured output

Each scenario is a JSON object with `scenario` code (P-XX / N-XX), `name`, `description`, `expected_status`, full `request` (method, URL with `{{base_url}}` placeholder, headers, body), full `response` (status, headers, parsed body), and `notes` (status code observation, key fields verified).

Response bodies are kept as the real server response. The only shape change is for very long arrays: arrays with more than 5 items keep their first 5 items in full and append a `{__dropped__: N, __note__: ...}` marker. Strings are never character-truncated and objects keep all keys, so readers always see real data — never a synthetic summary.

---

## Hard gates the skill enforces

The skill cannot proceed past four checkpoints without explicit approval, so coverage gaps cannot silently slip through:

1. **Endpoint scope lock** — only the handler whose route matches `ENDPOINT_PATH` + METHOD exactly is in scope. Sibling endpoints sharing a path prefix (e.g. `/p/glossary` vs `/p/glossarycategory`) are excluded.
2. **Delegation coverage audit** — before generating scenarios, the skill outputs every class/file it has read with line ranges and asks you to confirm no layer was missed.
3. **JSON-string sub-checklist** — params encoded as JSON strings are expanded leaf-by-leaf and per-type-branch before scenarios are generated.
4. **Coverage floor** — every leaf in the param checklist plus a mandatory baseline of protocol-level negatives and string-shape edge cases must be present before the file is written.

---

## Requirements

- Claude Code with plugin support
- A local checkout of `process-manager` (the skill greps source to discover scenarios)
- An SAP Signavio account that can log in to the staging instance
- Playwright MCP server enabled in your Claude Code config (used for token fetch)

---

## Repository layout

```
.
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   ├── capture-endpoint/
│   │   └── SKILL.md
│   └── metadata-context/
│       ├── SKILL.md
│       ├── metadata-guide.md
│       ├── documentation-philosophy.md
│       └── metadata-bugs/
│           ├── README.md
│           ├── causal-chains.md
│           ├── bug-patterns.md
│           └── shadow-bugs.md
└── README.md
```

---

## License

Internal SAP use only.
