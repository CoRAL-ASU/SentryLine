# SentryLine: Evidence-Grounded Question Answering over Evolving Documents in Oncology Care

Grounded, verified oncology guideline QA. See `EMNLP_System_Description.md`
for full architecture details.

## Requirements

- Python 3.10+
- A Google Gemini API key -- https://console.cloud.google.com/agent-platform/studio/settings/api-keys
  (any valid key works -- not account-restricted)
- **PageIndex API key.** see "Indexing under your own account"
  below.

## Setup

```bash
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

## Run

```bash
python run_demo.py --google-api-key YOUR_GOOGLE_KEY --pageindex-api-key 
```

- Keys can also be supplied via `GOOGLE_API_KEY` / `PAGEINDEX_API_KEY` environment
  variables, or you'll be prompted for them if you omit the flags.
- The demo opens automatically at `http://127.0.0.1:8081` in your default browser.
  Use `--port` to change it, `--no-browser` to skip auto-opening a tab.

### Indexing under your own account

To use your own PageIndex account:

```bash
rm data/guidelines/doc_ids.json
python run_demo.py --google-api-key YOUR_GOOGLE_KEY --pageindex-api-key YOUR_OWN_PAGEINDEX_KEY
```

This re-indexes all 56 PDFs under your account -- takes a few minutes and uses your
own PageIndex API credits. Every run after that starts immediately, same as above.

## What's included

- `app.py` / `pipeline/` -- the FastAPI backend and full 7-step pipeline
  (concept extraction, PageIndex retrieval, context merge, role-routed
  generation, factual + temporal verification, citation grounding, evidence
  extraction)
- `static/` -- the chat frontend, including the embedded PDF.js viewer used
  for paragraph-level evidence highlighting
- `data/guidelines/` -- 56 ASCO breast and prostate cancer guideline PDFs,
  pre-indexed (`doc_ids.json` included)
- `upload_guidelines.py` -- guideline indexing script (needed to index under your own PageIndex account)
- `main.py` -- optional terminal-only chat client, if you'd rather not use the
  web UI (`python main.py`)


## Troubleshooting

- **Port already in use:** `python run_demo.py ... --port 8090`
- **Re-index guidelines from scratch:** delete `data/guidelines/doc_ids.json` and run
- **Reset the answer cache / conversation history:** delete the corresponding
  file(s) under `data/cache/` or `data/conversations/` -- both are recreated
  automatically
