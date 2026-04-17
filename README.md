# ESUN / Customer I (anonymized) Incident Ticket Analysis Web Service

A **Python 3** + **Flask** ticket-driven analysis service: orchestrates single- and batch-analysis jobs, emits static HTML reports, and serves them over constrained routes. **Customer I (anonymized)** uses a separate analysis pipeline and a monthly aggregation/reporting subsystem, decoupled from the ESUN standard domain.

---

## Feature Matrix

| Subsystem | Description |
|-----------|-------------|
| **ESUN standard analysis** | `S0527-Standard-web`: origin fetch, field extraction, LLM inference, templated HTML output; optional on-disk JSON cache under `results/` |
| **Customer I (anonymized) single-ticket analysis** | `S0527-inn-web`: isomorphic contract to the standard pipeline; domain rules and presentation implemented separately |
| **Customer I (anonymized) monthly rollup** | `Report-Inn`: scheduled monthly reports by `YYYY-MM`; under XHR, progress via **SSE** (`text/event-stream`) |
| **Routing & static** | Portal, analysis, and customer-report pages; `send_from_directory` rooted at `REPORTS_DIR` |
| **Access audit** | Optional `access_audit.jsonl` (JSON Lines); `@app.after_request` appends structured access records |

---

## Stack

| Layer | Choice |
|-------|--------|
| Application protocol | HTTP/1.1; long-running job progress uses **SSE**, not WebSocket |
| WSGI | Dev: `Flask.run`; prod: optional **Waitress** (`USE_WAITRESS`) |
| Fetch / DOM | `requests`, `beautifulsoup4` |
| Time parsing | `python-dateutil` |
| LLM | OpenAI SDK; `base_url` may point to a Chat Completions–compatible gateway; credentials only via environment |
| Concurrency | `threading` + `queue.Queue` to decouple workers from the SSE generator |

---

## Repository Layout (production-relevant)

```
P4/
├── app.py                      # Routes, dynamic module load, audit hooks, report static serving
├── config.py                   # Port, report root, feature flags (overridable by env)
├── requirements.txt
├── S0527-Standard-web.py       # ESUN standard: extract → LLM → `generate_html_report`
├── S0527-inn-web.py            # Customer I (anonymized): parallel code path; filename keeps `inn` tag
├── Report-Inn.py               # Customer I (anonymized): monthly aggregation & report templates
├── calllist_dashboard_cache.py
├── reports/                    # Default `REPORTS_DIR`
├── results/                    # Analysis JSON cache (keyed by ticket, etc.)
├── static/                     # Vendor fonts, ApexCharts, etc.
└── lib/                        # vis-network, tom-select, and other front-end deps
```

`temp/` and `backup/` hold experiments and archives; they are **not** part of the deployment artifact set.

---

## Quick Start

### Virtual environment and dependencies

```bash
python -m venv .venv
.venv\Scripts\activate   # Windows
# source .venv/bin/activate  # Unix

pip install -r requirements.txt
```

### Environment variables

| Variable | Semantics |
|----------|-----------|
| `PORT` | Bind port; default `5000` |
| `ESUN_REPORTS_DIR` | Report root; if unset, `<repo>/reports` |
| `USE_WAITRESS` | Truthy → Waitress; else Flask’s built-in server |
| `FLASK_SECRET_KEY` | Session signing secret; **required in production** |
| `DEEPSEEK_API_KEY` | LLM credential (or compatible provider API key) |
| `ACCESS_AUDIT_DISABLE` | Truthy → skip audit file writes |

**Secrets**: do not commit to VCS; inject via CI/host secrets or an external KMS.

### Run

```bash
python app.py
```

Production: place Waitress behind a reverse proxy; terminate TLS, set timeouts, and forward `X-Forwarded-For` / `X-Real-Ip` (audit IP resolution reads these headers).

---

## Architecture Notes

1. **Module loading**: `importlib.util.spec_from_file_location` loads hyphenated script names (e.g. `S0527-Standard-web.py`), avoiding invalid `import` identifiers while keeping single-file scripts maintainable.
2. **Analysis layer**: `S0527-*` exposes `run_analysis`, `generate_html_report`, etc.; I/O and rendering stay out of Flask views.
3. **Reporting layer**: `Report-Inn` serves only the **Customer I (anonymized)** monthly domain; shares the `REPORTS_DIR` contract with single-ticket analysis; filenames are made unique by generation logic.
4. **SSE path**: the monthly job posts progress to a queue from a worker thread; the main generator `yield`s `data: <json>\n\n`; heartbeats mitigate some proxy buffering (`X-Accel-Buffering: no`).

---

## Dependencies

See `requirements.txt`: `flask`, `waitress`, `openai`, `requests`, `beautifulsoup4`, `python-dateutil`.

---

## Extension Conventions

- **New tenant/customer**: add a module whose entry contract matches `S0527-*`, register routes explicitly in `app.py`; avoid piling domain branches into views.
- Static report URLs stay aligned with `report/<path:filename>`; `send_from_directory` constrains paths under the configured root, reducing path-traversal risk.

---

## Author

Zhenyu Li

---

## License

If no `LICENSE` is shipped with the repository, **all rights reserved** by default; add a license and third-party notices before redistribution.
