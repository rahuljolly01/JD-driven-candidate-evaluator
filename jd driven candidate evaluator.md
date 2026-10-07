# Workflow 2: Serial Agent
## Sequential Execution Pattern for Task Chaining

**Version:** 1.0 | **Last Updated:** October 2026 | **Status:** Production-Ready

---

## Overview

The Serial Agent implements a **sequential execution pattern** where tasks execute one after another in a defined order. Each task's output feeds directly into the next task's input, enabling dependency-based workflows where later steps require earlier completion.

### Use Cases

- **Multi-stage content processing:** Research → Analysis → Summarization → Formatting
- **Approval workflows:** Submit → Review → Revise → Approve
- **Data enrichment pipelines:** Extract → Transform → Validate → Store
- **Document workflows:** Generate → Review → Finalize → Distribute

### Key Characteristics

| Aspect | Detail |
|--------|--------|
| **Execution Model** | Sequential, single-threaded |
| **Parallelization** | None; each node waits for predecessor |
| **Error Handling** | Stops at first failure (unless configured otherwise) |
| **Latency** | Cumulative across all stages |
| **Data Dependencies** | Strict; later stages require earlier completion |

---

## Architecture

```
Input Data
    ↓
[Task 1: Process] → Output A
    ↓
[Task 2: Analyze] → Output B (depends on A)
    ↓
[Task 3: Summarize] → Output C (depends on B)
    ↓
[Task 4: Format] → Final Output (depends on C)
```

### Components

1. **Input Trigger:** Initiates the workflow via manual trigger, HTTP webhook, or scheduled event
2. **Gemini Nodes:** LLM-powered transformation steps using Google's Gemini API
3. **Google Sheets Integration:** Logs intermediate and final results for audit trail
4. **Output Node:** Delivers results to destination (email, webhook, or storage)

---

## Prerequisites

### Required Accounts
- **n8n Cloud** (free tier eligible): [n8n.io](https://n8n.io)
- **Google Account:** Any Gmail address for Gemini API and Sheets integration
- **Gemini API Key:** Obtain from [Google AI Studio](https://aistudio.google.com)

### Required Setup
- Google Sheets document (shared with n8n service account during credential setup)
- Gemini API credential registered in n8n (shared across all workflows)

---

## Installation & Configuration

### Step 1: Import Workflow JSON
```bash
# In n8n Dashboard
1. Click menu (≡) → Workflows → Import from file
2. Select Agent_2_Serial.json
3. Click Import (do NOT save yet)
```

### Step 2: Configure Gemini Credential
**If this is your first workflow, set up the Gemini credential:**

1. Left sidebar → **Credentials** → **Add Credential**
2. Search: `Google Gemini` → Select `Google Gemini (PaLM) API`
3. Paste your API Key from [aistudio.google.com](https://aistudio.google.com)
4. Name: `Gemini Workshop Key` → **Save**
5. Verify: Green dot next to credential name in Credentials list

**If you've already configured Gemini, skip to Step 3.**

### Step 3: Configure Workflow Nodes

#### Node: Google Sheets (Input Log)
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing `Gemini Workshop Key` | If missing, return to Step 2 |
| **Sheet ID** | Your Google Sheet ID | URL: `sheets.google.com/spreadsheets/d/**[ID]**` |
| **Sheet Name** | Name of target tab (e.g., `Agent2_Input`) | Create tab if it doesn't exist |

#### Node: Gemini - Stage 1 (Process)
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | Must match credential set in Step 2 |
| **Model** | `gemini-1.5-flash` | Recommended for cost-efficiency |
| **Prompt** | Define your processing instruction | E.g., "Extract entities from: {{$json.input_text}}" |

#### Node: Gemini - Stage 2 (Analyze)
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | |
| **Model** | `gemini-1.5-flash` | Same as Stage 1 |
| **Prompt** | Reference Stage 1 output: `{{$json.stage1_result}}` | Ensure input uses correct reference path |

#### Node: Gemini - Stage 3 (Summarize)
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | |
| **Model** | `gemini-1.5-flash` | |
| **Prompt** | Summarize: `{{$json.stage2_result}}` | Limit tokens if needed for cost control |

#### Node: Gemini - Stage 4 (Format)
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | |
| **Model** | `gemini-1.5-flash` | |
| **Prompt** | "Format as markdown:\n{{$json.stage3_result}}" | Specify output format explicitly |

#### Node: Google Sheets (Output Log)
| Field | Value | Notes |
|-------|-------|-------|
| **Sheet ID** | Same as input log | |
| **Sheet Name** | Name of output tab (e.g., `Agent2_Output`) | Create tab if it doesn't exist |
| **Append** | Row with: `[timestamp, input, final_output, status]` | Ensures audit trail |

### Step 4: Test & Deploy
```bash
1. Click "Execute Workflow" (play button)
2. Verify all nodes execute sequentially (check execution timeline)
3. Confirm Google Sheets are populated with input and output
4. If errors occur, check API quota (Gemini API has rate limits)
5. Click "Save" to persist configuration
```

---

## Data Flow & Context Passing

Each stage must explicitly reference the previous stage's output. The workflow uses **n8n expression syntax** for data binding:

```javascript
// Stage 1 Output
{{ $json.stage1_result }}

// Stage 2 Input (references Stage 1)
Analyze the following processed data: {{ $json.stage1_result }}

// Stage 3 Input (references Stage 2)
Summarize this analysis: {{ $json.stage2_result }}

// Stage 4 Input (references Stage 3)
Format this summary: {{ $json.stage3_result }}
```

**Critical:** If a reference path is incorrect, the node will fail silently or pass undefined values.

---

## Error Handling & Troubleshooting

### Common Issues & Resolutions

| Error | Cause | Resolution |
|-------|-------|-----------|
| **"Authentication failed"** | Gemini API key invalid or expired | Regenerate key at [aistudio.google.com](https://aistudio.google.com) → Update credential |
| **"Sheet not found"** | Invalid Sheet ID or insufficient permissions | Verify Sheet ID in URL; ensure Google account has access |
| **Null or undefined output** | Incorrect node reference in prompt | Check variable syntax: `{{ $json.fieldName }}` (exact case) |
| **Rate limit exceeded** | Too many Gemini API calls in short time | Add delay node (2-3 seconds) between stages; upgrade Gemini API plan if sustained |
| **Workflow stops at Stage N** | Previous stage failed (check execution log) | Expand stage N in execution view; review error message; check input data validity |

### Debug Strategy

1. **Check Execution Timeline:** Click workflow execution → Expand each node → View input/output
2. **Verify Data Types:** Ensure outputs are strings (not arrays/objects) before passing to next stage
3. **Test Prompts Independently:** Run single Gemini node with hardcoded input to isolate LLM issues
4. **Monitor API Usage:** Visit [Google Cloud Console](https://console.cloud.google.com) → Check Gemini API quota

---

## Performance Considerations

### Latency Profile
- **Per Gemini Call:** 1-3 seconds (average)
- **Full Workflow (4 stages):** 4-12 seconds end-to-end
- **Bottleneck:** Gemini API response time (not n8n itself)

### Cost Optimization
- **Model Choice:** `gemini-1.5-flash` is 1/4 the cost of `gemini-1.5-pro`
- **Token Limits:** Specify max tokens per prompt (e.g., 300-500) to control spend
- **Batch Processing:** If processing multiple items, consider Parallel Agent (Workflow 3) instead

### Scaling Limits
- **Gemini API Rate Limit:** ~60 requests/minute on free tier
- **Google Sheets Quota:** 500 requests/minute per Sheet
- For higher throughput, upgrade to Gemini API paid plan and distribute Sheet writes

---

## Best Practices

### Workflow Design
1. **Clear Stage Objectives:** Each Gemini node should have one well-defined job (e.g., "extract" not "extract and analyze")
2. **Explicit Prompts:** Include context and format expectations in every prompt
3. **Intermediate Logging:** Store results at each stage in separate columns for debugging
4. **Error Escalation:** Add error handler nodes (catch failures before silent failure)

### Data Handling
1. **Input Validation:** Verify input structure before Stage 1 (check for required fields)
2. **Output Consistency:** Document expected output format from each stage
3. **Context Preservation:** Pass full context forward, not just the last node's output
4. **Audit Trail:** Always log to Google Sheets for compliance and troubleshooting

### Maintenance
1. **Version Control:** Store workflow JSON in Git; tag releases
2. **Monitoring:** Set up n8n alerts for failed executions
3. **Credential Rotation:** Regenerate Gemini API keys quarterly
4. **Testing:** Maintain test cases covering happy path and error scenarios

---

## Example: Content Enrichment Workflow

**Scenario:** Take raw research notes → Extract key findings → Analyze implications → Generate summary → Format for presentation

```
Input: "AI adoption in enterprise increases ROI. Survey of 500 companies shows 35% efficiency gains. Main barriers: talent, integration costs, change management."

↓ Stage 1 (Extract)
Output: "Key Findings: 1) 35% efficiency gains 2) Survey size: 500 companies 3) Barriers: talent, integration, change management"

↓ Stage 2 (Analyze)
Output: "Impact Analysis: Efficiency gains justify investment. Barriers are organizational, not technical. Medium complexity adoption path required."

↓ Stage 3 (Summarize)
Output: "Enterprise AI ROI is proven (35% gains) but requires organizational capability building. Talent and change management are critical success factors."

↓ Stage 4 (Format)
Output: (Markdown)
"## AI Enterprise ROI Summary
**Key Finding:** 35% efficiency gains documented
**Barriers:** Talent gaps, integration complexity, organizational change
**Recommendation:** Invest in capability building before deployment"
```

---

## Monitoring & Analytics

### Key Metrics to Track
- **Execution Success Rate:** Target ≥95%
- **Average Execution Time:** Baseline and track regressions
- **API Cost per Execution:** Monitor for unexpected spikes
- **Stage-Wise Pass Rate:** Identify where failures cluster

### n8n Monitoring Dashboard
```
Executions → Filter by workflow "Agent 2 - Serial"
↓
- Success count vs. failure count
- Average duration per execution
- Error type breakdown
- Daily/weekly trend analysis
```

---

## Advanced Variations

### Add Error Recovery
Insert a **retry node** after any Gemini call to handle transient failures:
```
If Gemini fails → Wait 3 seconds → Retry (max 2 attempts) → Then fail
```

### Add Conditional Branching
Use **IF nodes** to skip stages based on input:
```
If input.type == "urgent" → Skip analysis, go straight to formatting
```

### Async Results Delivery
Instead of synchronous output, send results via email/Slack:
```
Stage 4 Complete → Send email to stakeholder → Log completion → Return execution ID
```

---

## Support & Resources

- **n8n Docs:** [docs.n8n.io](https://docs.n8n.io)
- **Gemini API Guide:** [ai.google.dev](https://ai.google.dev)
- **n8n Community:** [community.n8n.io](https://community.n8n.io)
- **Issue Reporting:** Include execution log (JSON) and exact error message

---

## License & Attribution

This workflow was built as part of the **BITSoM × Masai "Product Management with Generative & Agentic AI"** workshop series.

**Status:** Production-Ready | **Last Tested:** October 2026

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Oct 2026 | Initial release; serial execution pattern documented |

