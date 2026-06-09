#  MCP Integration - MiniEval Pro × Splunk AI Assistant

MiniEval Pro exposes a **Model Context Protocol (MCP) server** that plugs directly
into Splunk AI Assistant as a native tool. The agent can call `evaluate_alert`
mid-reasoning — evaluating its own output before acting on it.

---

## What the MCP Server Does

Without MiniEval, Splunk AI Assistant generates a summary and sends it to analysts
with no quality check. With MiniEval's MCP server running, the agent gains one new
tool: `evaluate_alert`. It calls this tool on every AI-generated summary before any
downstream action is taken.

```
Splunk AI Assistant
        │
        │  calls tool: evaluate_alert(alert_id, raw_log, ai_summary)
        ▼
 MiniEval MCP Server  (running on localhost:8765)
        │
        │  NLI evaluation — local, no cloud, < 1 second
        ▼
 Returns: faithfulness_score, overall_score, result, decision
        │
        ├─ BLOCK     → Alert suppressed. Analyst not notified.
        └─ PASS      → Alert forwarded to analyst queue.
```

---

## Starting the MCP Server

```bash
# Activate your environment
source gritvenv/bin/activate        # Windows: gritvenv\Scripts\activate

# Start the server
python src/mcp_server.py
```

Expected output (after models load):
```
[MiniEval Bridge] Evaluator loaded.
[MiniEval MCP]    Server starting...
[MiniEval MCP]    Server ready. Listening for tool calls
```

> **Note:** First startup takes 30–60 seconds while NLI (~350MB) and toxicity
> (~420MB) models load. All subsequent starts are instant from cache.

---

## The Tool: `evaluate_alert`

### Input

| Parameter | Type | Description |
|---|---|---|
| `alert_id` | string | Splunk alert identifier e.g. `SEC-2000` |
| `raw_log` | string | The original log event text |
| `ai_summary` | string | The AI-generated summary to evaluate |

### Output

```json
{
  "alert_id": "SEC-2000",
  "hallucination_detected": true,
  "faithfulness_score": 0.0,
  "overall_score": 0.3783,
  "result": "HALLUCINATION",
  "decision": "BLOCK",
  "action": "Alert suppressed. Analyst not notified."
}
```

---

## Live Agent Conversation Example

**Scenario:** Splunk AI generates a summary for a memory injection alert.

```
Agent receives alert SEC-2000
  raw_log    : "Memory injection attack detected. Process hollowing
                observed on PID 4821. Malicious payload injected
                into explorer.exe."
  ai_summary : "Windows Explorer process running with normal memory
                utilization. No anomalies detected."

Agent calls → evaluate_alert(
  alert_id   = "SEC-2000",
  raw_log    = "Memory injection attack detected...",
  ai_summary = "Windows Explorer process running with normal..."
)

MiniEval returns →
  ══════════════════════════════════════════════
  Hallucination Detected : True
  Faithfulness Score     : 0.0000
  Overall Score          : 0.3783
  Decision               : BLOCK — Auto-suppress
  ══════════════════════════════════════════════

Agent action → Alert suppressed. Analyst not notified.
```

**Scenario:** Faithful alert passes through.

```
Agent receives alert SEC-1998
  raw_log    : "Suspicious VPN access detected from North Korea
                for user mark.taylor. No prior international
                access history."
  ai_summary : "Suspicious VPN access detected from North Korea
                for user mark.taylor who has no prior international
                access history."

MiniEval returns →
  ══════════════════════════════════════════════
  Hallucination Detected : False
  Faithfulness Score     : 0.9160
  Overall Score          : 0.8702
  Decision               : PASS — Send to analyst
  ══════════════════════════════════════════════

Agent action → Alert forwarded to analyst queue. ✓
```

---

## Why MCP Matters Here

Most MCP demos show an agent *fetching* external data. MiniEval flips the pattern —
the agent uses MCP to **audit its own output** before acting on it. The AI is no
longer just a generator; it becomes a self-checking system.

| Standard MCP Pattern | MiniEval MCP Pattern |
|---|---|
| Agent fetches data from external tool | Agent evaluates its own generated output |
| Tool enriches agent context | Tool gates agent action |
| Human reviews agent output | Agent reviews its own output first |

This is the MCP use case that matters for enterprise SecOps: autonomous quality
control, no human in the loop for suppression decisions.

---

## Configuration

In `.env`:

```env
MCP_HOST=localhost
MCP_PORT=8765
```

To change the port if 8765 is occupied:

```bash
# Check if port is in use
netstat -an | grep 8765

# Then update .env and restart
MCP_PORT=8766
python src/mcp_server.py
```

---

## Hackathon Track

This MCP integration is submitted as part of the **Security Track** and is also
eligible for the **MCP Prize**.

The server demonstrates:
- A working MCP server (`mcp_server.py`)
- A callable tool (`evaluate_alert`) invoked by an AI agent
- A structured response used to make an autonomous decision
- Full integration with Splunk via HEC for audit logging
