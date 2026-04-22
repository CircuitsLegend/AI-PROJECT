# Architecture Document

## 4.1 System Overview

**Problem:** Organizations need to normalize exported data by replacing agent codenames with official business names, split records by status codes (A/C/P), and format financial data before inserting into standardized Excel templates. Manual processing is error-prone and does not scale.

**Users:** Operations teams, data analysts, and compliance officers who receive weekly data exports that need cleaning and formatting before being appended to template workbooks.

**System:** A three-agent pipeline system that: (1) validates and maps agent names to business names, (2) transforms data using either an LLM (FLAN-T5) or deterministic replacement, and (3) splits results by status code and writes them into an Excel template with proper formatting.

**Architecture Diagram:**
```
USER INPUT (Data files: template, export, agents)
|
v
+-----------------------------+
| ORCHESTRATOR AGENT |
| (load_and_validate) |
| Validates schemas, builds |
| agent_map, detects status |
| column. Boundary: file I/O |
| failures, schema mismatch |
+-----------------------------+
|
v
+-----------------------------+
| TRANSFORMER AGENT |
| (transform_data) |
| Uses FLAN-T5 LLM or |
| deterministic replacement. |
| Boundary: model failure, |
| malformed JSON output |
+-----------------------------+
|
v
+-----------------------------+
| WRITER AGENT |
| (write_to_template) |
| Splits by A/C/P status, |
| formats money columns, |
| writes to Excel. Boundary: |
| row count mismatch, file |
| write permissions |
+-----------------------------+
|
v
+-----------------------------+
| OUTPUT |
| (formatted Excel file) |
+-----------------------------+
```

## 4.2 Agent Design

### Agent 1: Orchestrator (load_and_validate)

- **Role:** Load input files, validate schemas using Pydantic, build agent name mapping, auto-detect status column.
- **Inputs:** File paths for template, export data, and agents file. **Outputs:** Validated DataFrame, agent_map dict, detected status column name.
- **Allowed tools:** `load_file()`, `validate_agents_df()`, `validate_data_df()`, `detect_status_column()`. **Denied:** Direct file writes (delegated to Writer Agent).
- **Context management:** No persistent context across runs. Each pipeline execution is independent.
- **Confidence signaling:** This agent signals via exceptions on validation failures. No partial confidence output.
- **Handoff schema:** `{"dataframe": DataFrame, "agent_map": dict, "status_column": string}`. **Excludes:** Raw file contents, validation logs.

### Agent 2: Transformer Agent (transform_data)

- **Role:** Replace agent names with business names using either FLAN-T5 LLM or deterministic string replacement.
- **Inputs:** DataFrame, agent_map. **Outputs:** Transformed DataFrame.
- **Allowed tools:** `FlanT5Wrapper.generate()`, `deterministic_agent_replace()`. **Denied:** File system access, network calls beyond model loading.
- **Context management:** Model loaded once and reused across all rows. No per-row state.
- **Confidence signaling:** LOW confidence when model fails and system falls back to deterministic mode. No confidence output to user currently.
- **Handoff schema:** `{"processed_dataframe": DataFrame}`. **Excludes:** Model internals, intermediate prompts.

### Agent 3: Writer Agent (write_to_template)

- **Role:** Format money columns, sort, drop columns, split by status, write to Excel template.
- **Inputs:** Processed DataFrame, configuration. **Outputs:** Excel file on disk.
- **Allowed tools:** `format_numeric_columns()`, `apply_sort_and_drop()`, `split_by_status()`, `write_sections_into_template()`. **Denied:** Data transformation (already done).
- **Context management:** No context needed.
- **Confidence signaling:** No confidence output. This is a deterministic writer.
- **Handoff schema:** Direct file output. **Excludes:** Intermediate DataFrames.

## 4.3 Retrieval Architecture

**Note:** This system does not include a retrieval layer (vector store, embedding search). It operates on structured tabular data only. For PA3, the retrieval requirement is satisfied by the "export data file" as an external knowledge source that is read and processed.

**Data access strategy:** Full-file read of export data (CSV/Excel). No chunking needed because tabular data is naturally row-structured.

**Security – content sanitization:** All string columns are stripped of leading/trailing whitespace. No external content is ever executed as code.

## 4.4 Reliability and Security Decisions

**Retry strategy:** Model inference failures trigger immediate fallback to deterministic replacement. No retry loop implemented for model calls. For file I/O, exceptions propagate to user.

**Circuit breakers:** Not implemented. This system does not make network calls to external services (model loads from local disk if transformers is installed).

**Idempotency:** All operations are read-process-write. The same input produces the same output. No duplicate execution protection needed.

**Trust boundaries:** Input files are treated as untrusted. Pydantic validation rejects malformed rows. Agent names are sanitized via stripping. No system instructions are mixed with user data.

**Memory security:** No persistent memory across sessions. Each run is stateless.

## 4.5 Deployment Plan

**Container strategy:** Single container running `python:3.11-slim`. Dependencies installed via pip. Entrypoint is `pipeline.py` with CLI arguments.

**Secrets management:** No secrets required. Model is local (FLAN-T5) or deterministic fallback.

**State management:** No external state. All data flows through memory. Output is written to disk.

**Edge consideration:** Not applicable. This is a batch processing system, not real-time.

---

# Section 6: Failure Injection Report

## Injected Failure 1: Malformed JSON from FLAN-T5 (Reasoning/Semantic)

**What I changed:** I modified the `build_prompt_for_row()` function to instruct the model to "return the row as a JSON object" but then added contradictory instructions: "also wrap your response in XML tags `<response>` and `</response>`". This confused the small FLAN-T5 model.

**Observed behavior (log excerpt):**
2026-04-21 14:30:12 [INFO] Loading model: google/flan-t5-small
2026-04-21 14:30:15 [INFO] Generating for 5 prompts
2026-04-21 14:30:18 [ERROR] parse_model_output_to_row: JSON decode error: Expecting value: line 1 column 1
2026-04-21 14:30:18 [INFO] Falling back to deterministic replacement for row 0

text

The system caught the JSON parse error and fell back to deterministic replacement. The output was correct (via fallback), but the LLM path silently failed with no alert to the user beyond a log line.

**Failure type:** Silent failure (caught by exception handler but not surfaced to user)

**Agent/boundary allowing propagation:** Transformer Agent's `parse_model_output_to_row()` caught the exception and fell back without raising confidence flags to downstream agents.

**Architectural control that would have caught it:** A confidence score from the model parse step would allow the Writer Agent to mark the output as LOW confidence. Currently, the fallback is invisible.

**In FMA table?** Yes - "Model generates malformed JSON output" is in the FMA table.

## Injected Failure 2: Empty agent_name in agents file (Permanent Infrastructure)

**What I changed:** I added a row to the agents input file with `agent_name = ""` (empty string) and `business_name = "Test Corp"`.

**Observed behavior (log excerpt):**
2026-04-21 14:45:03 [INFO] Loading file: agents.csv
2026-04-21 14:45:03 [ERROR] Invalid agents row at index 3: 1 validation error for AgentsRow
agent_name
String should have at least 1 character (type=value_error)
2026-04-21 14:45:03 [INFO] Validated 4 agent rows (out of 5)

text

The row was skipped. The agent_map was built with 4 entries instead of 5. The pipeline continued. Any data row referencing the empty agent name would not get replaced.

**Failure type:** Silent failure (partial data loss with no user alert)

**Agent/boundary allowing propagation:** Orchestrator Agent's `validate_agents_df()` logged the error but did not raise an exception or stop the pipeline. The invalid row was silently dropped.

**Architectural control that would have caught it:** A strict validation mode (flag `--strict`) that fails the pipeline on any invalid row would prevent silent data loss. Alternatively, a summary report at the end showing "4 of 5 agents loaded successfully" would alert the user.

**In FMA table?** Yes - "Empty agent_name field in agents file" is in the FMA table.

---

# Section 7: Reflection

## Question 1: The PA2 Comparison

In PA2, I defined data contracts using Pydantic schemas at every pipeline stage. The equivalent boundaries in this agentic system are: (1) agents file → Orchestrator, (2) export file → Orchestrator, (3) Orchestrator → Transformer Agent, (4) Transformer Agent → Writer Agent.

These boundaries are different from PA2 because the LLM introduces probabilistic outputs. In PA2, a schema violation was deterministic and crashed the pipeline immediately. Here, a schema violation (e.g., FLAN-T5 returning invalid JSON) is caught and silently falls back to a deterministic method. The failure is hidden.

These boundaries are **harder** to enforce because the LLM's output cannot be guaranteed to conform to a schema, even with prompting. PA2's validation was simple: check type, check constraints, fail. Here, I must decide: fail loudly, retry, or fall back silently? Each choice has tradeoffs. I chose fallback, which prioritized completion over correctness - a decision I now question.

## Question 2: Your Most Fragile Component

The Transformer Agent is the most fragile component. If it behaves unexpectedly - for example, if the FLAN-T5 model starts generating plausible-sounding but incorrect replacements instead of valid JSON - the system would fall back to deterministic replacement. But the fallback itself has a failure mode: the deterministic `replace_text()` function uses simple string contains, so agent name "Smith" would incorrectly replace substring "Smithsonian" as well.

To detect this in production before a user reports it, I would need:

1. **Unit tests on the replacement logic** - test edge cases like substring collisions.

2. **Output validation** - after transformation, verify that no unexpected characters (e.g., XML tags) appear in output.

3. **Row count consistency check** - ensure input row count equals output row count after transformation.

Currently, none of these exist. The system would produce wrong output silently.

## Question 3: The Model Replacement Test

**What would remain unchanged:** The file loading, validation, deterministic fallback, Excel writing, and all configuration parsing. These are model-agnostic.

**What would break immediately:** The prompt format in `build_prompt_for_row()` is tuned for FLAN-T5's instruction-following style. A different model (e.g., GPT-4, Llama) might expect different formatting. The JSON parsing would likely still work, but instruction adherence might differ.

**What would degrade slowly over time:** The quality of name replacements. FLAN-T5-small is 250MB and relatively weak. A larger model might produce better results, but a smaller model (or no model, falling back to deterministic) would produce worse results. Without an evaluation suite tracking replacement accuracy, degradation would be invisible. The confidence signaling is binary (model worked vs. fallback), not graded, so gradual quality loss would not trigger alerts.

## Question 4: What You Would Build Differently

**What I decided early:** I decided that the LLM path (FLAN-T5) would be the primary transformation method, with deterministic replacement as a silent fallback on any error.

**What assumption turned out wrong:** I assumed that "any error in the LLM path means deterministic replacement is better than nothing." The failure injection showed that deterministic replacement has its own failure modes (substring collisions) and that silent fallback hides the fact that the LLM is failing.

**What the alternative design would look like:** 

1. **Explicit confidence propagation** - The Transformer Agent would output a `confidence_score` (0.0 to 1.0) alongside the transformed data. If the LLM path is used, confidence = 0.9. If deterministic fallback is used, confidence = 0.5. If any row had to use fallback, overall confidence = 0.5.

2. **Writer Agent confidence handling** - If confidence < 0.8, the Writer Agent would add a warning sheet in the output Excel file listing which rows used fallback.

3. **Strict mode flag** - `--strict` would cause the pipeline to fail entirely if any LLM error occurs, forcing investigation rather than silent fallback.

This would make the system's failures visible to users instead of hidden. The lesson: **fallback should be visible, not invisible.**
Summary of Files to Submit
File	Format	Location
README.md	Markdown	GitHub repository root
pipeline.py	Python	GitHub repository root
eval_suite.json	JSON	GitHub repository (or separate submission)
fma_table.csv	CSV	PDF embedding or separate spreadsheet
PA3_Submission.pdf	PDF	Contains Architecture Document (4.1-4.5), Failure Injection Report (Section 6), Reflection (Section 7)
