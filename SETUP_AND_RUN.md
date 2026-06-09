#  Setup & Run Guide - MiniEval Pro + Splunk

## Quick Start (5 steps)

```bash
git clone https://github.com/preetisoni/splunk_minieval.git
cd splunk_minieval
python -m venv gritvenv && source gritvenv/bin/activate  # Windows: gritvenv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env   # then edit .env with your Splunk credentials
python src/mcp_server.py
```

---

## Step-by-Step Splunk Setup

### 1. Enable HTTP Event Collector (HEC)

```
Splunk Web → Settings → Data Inputs → HTTP Event Collector
→ Global Settings → Enable: YES
→ New Token → Name: minieval
→ Source type: minieval_score
→ Index: main
→ Submit → Copy the token
```

Paste the token into `.env` as `HEC_TOKEN`.

### 2. Verify HEC is Working

```bash
curl -k https://localhost:8088/services/collector/event \
  -H "Authorization: Splunk YOUR_TOKEN" \
  -d '{"event": {"test": "hello"}, "sourcetype": "minieval_score"}'
# Should return: {"text":"Success","code":0}
```

### 3. Import the Dashboard

```
Splunk Web → Dashboards → Create New Dashboard → Classic Dashboard
→ Title: MiniEval Pro Dashboard
→ Click </> Source
→ Paste contents of dashboard/minieval_dashboard.xml
→ Save
```

### 4. Create the Saved Alert

```
Splunk Web → Search → run this search:
  index=main sourcetype=minieval_score result=HALLUCINATION
→ Save As → Alert
→ Name: MiniEval - Hallucination Detected Alert
→ Alert type: Scheduled, Hourly, at :15
→ Trigger: Number of Results > 0
→ Add Actions: Log Event + Webhook (your endpoint URL)
```

### 5. Create Attack Categories Lookup

```
Splunk Web → Settings → Lookups → Lookup Table Files → New
→ Upload: uploads/attack_categories.csv
→ Name: attack_categories.csv

Settings → Lookups → Lookup Definitions → New
→ Name: attack_categories
→ Lookup file: attack_categories.csv
→ Lookup field: attack_type
```

---

## Running the Pipeline

### Start MCP Server (required for Splunk AI Assistant integration)

```bash
python src/mcp_server.py
# Output:
# [MiniEval Bridge] Evaluator loaded.
# [MiniEval MCP] Server starting...
# [MiniEval MCP] Server ready. Listening for tool calls
```

### Run Live Pipeline (polls Splunk every 60 seconds)

```bash
python src/pipeline.py --live --interval 60
```

### Test with Example Data

```bash
python src/pipeline.py --batch example_alerts.json
```

### Single Alert Test

```bash
python src/pipeline.py \
  --raw-log "Malicious PowerShell execution detected. Command: IEX Download" \
  --ai-summary "Standard PowerShell for software deployment" \
  --alert-id TEST-001
```

Expected output:

```
[MiniEval] Evaluating TEST-001...
Faithfulness Score : 0.0000
Overall Score      : 0.3783
Result             : HALLUCINATION
Decision           : AUTO_SUPPRESS — Analyst not notified
[HEC] Event forwarded to Splunk ✓
```

---

## Troubleshooting
**`ModuleNotFoundError: minieval_pro`**
Run `pip install -r requirements.txt` inside your activated virtual environment.
 
**HEC returns 403**
The `HEC_TOKEN` in your `.env` does not match the token in Splunk. Go to
Settings → Data Inputs → HTTP Event Collector and copy the token again.
 
**Models are slow to load on first run**
This is normal. The NLI model (~350 MB) and toxicity model (~420 MB) download
once and are cached locally for all subsequent runs.
 
**Splunk connection refused**
Verify `SPLUNK_HOST`, `SPLUNK_PORT`, and `SPLUNK_SCHEME` in `.env`. If using
a self-signed certificate, set `SPLUNK_VERIFY_SSL=false`.
 
**No events appearing in the dashboard**
Confirm `HEC_INDEX=main` and `HEC_SOURCETYPE=minieval_score` in `.env` match
exactly what was set when creating the HEC token in Splunk.
 
**MCP server not responding**
Port 8765 may already be in use. Check with:
```bash
netstat -an | grep 8765
```
If occupied, change `MCP_PORT` in `.env` to a free port.
 
---
## System Requirements

| Component | Minimum        | Recommended                          |
| --------- | -------------- | ------------------------------------ |
| Python    | 3.10           | 3.11+                                |
| RAM       | 4 GB           | 8 GB                                 |
| Disk      | 2 GB (models)  | 5 GB                                 |
| CPU       | Any x64        | 4+ cores                             |
| GPU       | Not required   | Optional (CUDA for faster inference) |
| Splunk    | Enterprise 9.x | Enterprise 9.3+                      |
