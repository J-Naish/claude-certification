# Industrial Field Maintenance Agent

Scenario: **AxisWorks**, a manufacturer of industrial CNC machines and robotic production cells, is building a Claude-powered field-maintenance system called **FieldLens**. Field technicians use FieldLens on factory floors to diagnose machine alarms, compare telemetry against maintenance manuals, generate repair plans, check spare-part availability, and prepare post-repair service notes. The system connects to machine telemetry, PLC alarm history, maintenance manuals, spare-parts inventory, technician notes, warranty records, and remote diagnostic procedures through MCP servers and Claude API tools.

This scenario intentionally avoids customer support resolution, code generation, CI review, multi-agent research reports, legal extraction, and privacy/data-governance audit workflows. It focuses on safety-critical tool design, MCP client/server features, field-service orchestration, long-manual context handling, and reliable human review. Standalone CCAF Foundations mock scenario (15 questions). Difficulty: hard. Pass threshold: 11/15 (72%).

---

## Question 1

FieldLens uses client-executed tools for `read_alarm_history`, `get_live_telemetry`, `lookup_part_inventory`, and `schedule_service_visit`. A developer proposes parsing Claude's final prose for the phrase "I need telemetry" to decide when to call `get_live_telemetry`.

What is the correct implementation pattern?

A) Parse assistant prose for tool-intent phrases, execute the inferred tool, and continue until the prose contains a final recommendation.

B) Let Claude emit `tool_use` blocks, execute those tool calls in application code, return `tool_result` blocks, and continue while `stop_reason` is `"tool_use"`.

C) Pre-execute every available tool before the first Claude call so no tool-use loop is necessary.

D) Disable tool use and ask Claude to produce a checklist based only on the technician's description.

Correct Answer: B

Tool use is a structured contract: Claude emits `tool_use` requests, the application executes client-side tools, and tool results are appended to the conversation. Parsing natural language for tool intent is brittle and bypasses the schema that exists precisely to avoid free-form intent extraction.

Reference: docs/Claude API/Build/Tools/How tool use works.md; Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 2

A technician reports intermittent spindle overheating. FieldLens needs to inspect alarm history, vibration telemetry, coolant-system readings, maintenance-manual procedures, and local technician notes. Raw data can exceed 120k tokens.

Which architecture is strongest?

A) Give a single agent all tools and all raw data so it can reason without losing anything.

B) Have each specialist agent write directly to the final service report to avoid coordinator bottlenecks.

C) Use a coordinator that delegates scoped subagents for telemetry, manual procedure lookup, and parts/service history; each returns compact findings with provenance for synthesis.

D) Submit the entire diagnostic task to the Message Batches API because large context is always cheaper in batches.

Correct Answer: C

This is a coordinator-subagent fit: parallel specialist work reduces latency, isolated contexts avoid flooding the coordinator, and structured findings preserve provenance. Direct agent-to-report writing removes centralized error handling and synthesis control; a single raw context invites attention loss.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md; docs/Claude Code/Build with Claude Code/Agents/Create custom subagents.md

---

## Question 3

The maintenance MCP server exposes `run_spindle_calibration(machine_id)` and `unlock_safety_interlock(machine_id)`. Both can change machine state, and misuse could injure technicians or damage equipment.

What is the best safety design?

A) Make tool names scary enough that Claude avoids using them unless absolutely necessary.

B) Keep the tools available but put "ask the technician first" in the system prompt.

C) Require a UI confirmation/human approval path for these operations, and enforce prerequisite checks such as lockout/tagout status before execution.

D) Run the tools automatically but log the action for later incident review.

Correct Answer: C

MCP tool invocations affecting real equipment need human-in-the-loop approval and deterministic prerequisites. Tool descriptions and prompts are useful guidance but not enforcement. Logging after execution is not a safety control for dangerous side effects.

Reference: docs/Model Context Protocol/Specification/Server Features/Tools.md; Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 4

FieldLens includes a local MCP server that reads diagnostic bundles from the technician's laptop. The server should only operate inside directories explicitly selected by the client workspace, and it must update its view when the selected workspace changes.

Which MCP feature is designed for this?

A) Resources, because all filesystem paths should be exposed as browsable documents.

B) Roots, because clients can expose filesystem boundaries and notify servers when the root list changes.

C) Elicitation, because servers should ask the user for every file path in a form.

D) Tool annotations, because annotations enforce filesystem boundaries automatically.

Correct Answer: B

MCP roots let clients expose filesystem boundaries using `file://` URIs, and supporting clients can notify servers with `notifications/roots/list_changed`. Resources expose content; roots define where a server may operate.

Reference: docs/Model Context Protocol/Specification/Client Features/Roots.md

---

## Question 5

The parts-order MCP server needs additional information mid-flow. It may ask for a non-sensitive delivery dock number, but payment authorization must happen through the company's procurement portal and must not pass through the MCP client.

Which elicitation design is correct?

A) Use form mode for both dock number and payment authorization so the server receives all needed data in one structured response.

B) Use URL mode for the dock number and form mode for payment authorization to keep the user experience consistent.

C) Avoid elicitation entirely; ask Claude to infer the dock number and payment authorization from prior context.

D) Use form mode for the dock number and URL mode for the payment authorization flow.

Correct Answer: D

MCP elicitation form mode is appropriate for structured, non-secret information the user can review. The spec says secrets or credentials that grant access or authorize transactions must use URL mode, not form mode.

Reference: docs/Model Context Protocol/Specification/Client Features/Elicitation.md

---

## Question 6

`analyze_vibration_spectrum` can take 4 minutes and streams progress to the UI. A technician cancels the analysis after realizing the wrong machine ID was selected. The request is a normal MCP request, not a task-augmented request.

Which behavior matches the MCP utilities specification?

A) The client sends `notifications/cancelled` referencing the in-progress request ID; the server should stop if possible, free resources, and not send a normal response.

B) The client sends `tools/list_changed` so the server knows the analysis tool is no longer needed.

C) The server must complete the analysis anyway because MCP cancellation is only advisory for UI display.

D) The client sends `tasks/cancel`, because all long-running MCP requests use the task cancellation API.

Correct Answer: A

For non-task requests, MCP cancellation uses `notifications/cancelled` with the request ID and optional reason. Receivers should stop processing and free resources when possible. `tasks/cancel` is specifically for task-augmented requests, not every long-running request.

Reference: docs/Model Context Protocol/Specification/Base Protocol/Utilities/Cancellation.md

---

## Question 7

The same vibration analysis should show progress in the technician UI. The client wants updates only for active operations and needs to correlate each progress event to the correct request.

What should the client include?

A) A `progressToken` in request metadata, unique across active requests.

B) A random `requestId` inside the user prompt, so Claude can copy it into status messages.

C) A `roots/list` request before every progress update.

D) A `tool_choice` value of `"progress"` so Claude is forced to report status.

Correct Answer: A

MCP progress tracking is requested by including a unique `progressToken` in request metadata. Progress notifications reference that token and must stop after completion. Prompt text and `tool_choice` do not provide protocol-level progress correlation.

Reference: docs/Model Context Protocol/Specification/Base Protocol/Utilities/Progress.md

---

## Question 8

FieldLens must answer questions from large maintenance manuals with precise supporting evidence. The team wants Claude to cite manual pages in the answer. They also want the final answer to be a strict JSON object using `output_config.format`.

What is the most accurate design decision?

A) Enable citations and `output_config.format` in the same request; citations become JSON fields automatically.

B) Use prompt instructions asking Claude to include `"page": 12` fields, because prompt-based citations are more reliable than native citations.

C) Do not use documents; paste manual snippets into the system prompt so citations are unnecessary.

D) Avoid combining native citations with structured outputs in one request; split the workflow or choose the feature that matches the downstream requirement.

Correct Answer: D

Claude's citations feature and Structured Outputs are incompatible in the same request. If both evidence reliability and strict JSON are required, the architecture should split steps or make an explicit tradeoff rather than assuming the features compose.

Reference: docs/Claude API/Build/Model capabilities/Citations.md; docs/Claude API/Build/Model capabilities/Structured outputs.md

---

## Question 9

FieldLens receives uploaded service documents of unknown type: alarm export, warranty claim, parts quote, or technician note. The application provides four extraction tools, one schema per document type, and wants Claude to always choose exactly one extraction tool rather than answer in prose.

Which `tool_choice` setting fits best?

A) `tool_choice: "auto"` because Claude should decide whether any extraction is needed.

B) `tool_choice: "any"` because Claude must call one of the available tools but can choose which schema applies.

C) `tool_choice: {"type": "tool", "name": "extract_alarm_export"}` because forcing one tool is safest for unknown documents.

D) No tools; ask for JSON in the prompt and repair invalid JSON downstream.

Correct Answer: B

When multiple extraction schemas exist and the document type is unknown, `tool_choice: "any"` forces a tool call while allowing Claude to select the right extraction schema. Forced selection is appropriate only when the needed tool is known in advance.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md; docs/Claude API/Build/Tools/How tool use works.md

---

## Question 10

AxisWorks wants to process 40,000 historical service notes overnight to identify recurring failure modes. The job does not need live tool calls, and results are due the next morning. A second workflow is an on-site technician waiting for a repair recommendation.

Which API strategy is appropriate?

A) Use Message Batches for the overnight historical analysis and synchronous Messages API calls for the technician-facing workflow.

B) Use Message Batches for both workflows because batches are always faster for large inputs.

C) Use synchronous API calls for both workflows because batches cannot reduce cost.

D) Use Message Batches for the technician workflow and ask the technician to wait for batch completion.

Correct Answer: A

Batch processing fits latency-tolerant, high-volume offline work and offers cost savings. Interactive field diagnostics require synchronous response behavior. The exam guide also warns against batch APIs for blocking or user-waiting workflows.

Reference: docs/Claude API/Build/Model capabilities/Batch processing.md; Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 11

A 600-page service manual contains the torque specification in a middle appendix. FieldLens often cites early troubleshooting flowcharts and misses appendix data. The team cannot rely on one giant prompt.

Which context strategy is strongest?

A) Put the entire manual in the prompt and raise `max_tokens` to make the model read more input.

B) Randomize page order so the middle appendix is sometimes near the beginning.

C) Chunk by subsystem and procedure, retrieve the relevant chunks, put a concise key-facts summary before detailed evidence, and preserve page provenance.

D) Remove page provenance to reduce token usage and make room for more manual text.

Correct Answer: C

Chunking by domain structure, retrieving relevant sections, placing key facts early, and preserving source metadata directly address lost-in-the-middle and provenance risks. `max_tokens` controls output budget, not reliable attention over a huge input.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md; docs/Claude API/Build/Context management/Context windows.md

---

## Question 12

During development, engineers repeatedly ask Claude Code to inspect diagnostic logs and manual snippets before editing repair-plan templates. This exploration floods the main conversation with long logs. The workflow is project-specific and should be shared with the team.

Which Claude Code configuration is the best fit?

A) Add a project-scoped custom subagent under `.claude/agents/` with read-only tools and focused diagnostic-analysis instructions.

B) Add all diagnostic logs permanently to `CLAUDE.md` so every session starts with full context.

C) Use a user-scoped personal subagent only, because project subagents cannot be checked into version control.

D) Disable subagents and rely on `/compact` after every log read.

Correct Answer: A

Project custom subagents are designed for repeated, project-specific side tasks that would otherwise flood the main context. They can be checked into `.claude/agents/`, use focused prompts, and restrict tool access. `CLAUDE.md` should not carry large transient logs.

Reference: docs/Claude Code/Build with Claude Code/Agents/Create custom subagents.md; Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 13

`get_coolant_pressure(machine_id)` sometimes returns no anomalies and sometimes cannot read the sensor because the gateway is offline. The current MCP server returns `{pressure: null}` in both cases.

What should the tool return?

A) A single nullable pressure field is sufficient; Claude can infer whether null means normal or offline.

B) Separate successful empty/normal results from structured errors, including error category, retryability, and what was attempted when the gateway is offline.

C) Always retry until the gateway returns a number, then hide the retries from the coordinator.

D) Throw a generic "operation failed" error without metadata so Claude does not overfit to implementation details.

Correct Answer: B

The agent needs to distinguish valid normal readings from access failures. Structured error metadata such as category and retryability lets the coordinator decide whether to retry, use alternate evidence, or escalate. A nullable field collapses semantically different states.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md; docs/Model Context Protocol/Specification/Server Features/Tools.md

---

## Question 14

FieldLens initially recommends "replace spindle bearings" too often, creating expensive false positives. The team has examples where vibration noise is caused by loose mounting bolts, worn bearings, or coolant pump resonance, and the distinction depends on specific telemetry patterns.

Which prompt-improvement approach is strongest?

A) Add "be conservative" to the prompt and lower temperature to 0.

B) Ask Claude to report only high-confidence diagnoses and suppress everything else.

C) Add targeted few-shot examples showing each ambiguous telemetry pattern, the correct diagnosis, and the reasoning signal that distinguishes it from plausible alternatives.

D) Remove telemetry from the prompt and rely only on the maintenance manual's generic troubleshooting flowchart.

Correct Answer: C

Targeted few-shot examples are best for ambiguous judgment boundaries: they teach the discriminating signals between near-miss diagnoses. Vague conservatism and self-confidence filters do not reliably reduce the specific false-positive pattern.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 15

AxisWorks reports 94% overall diagnostic accuracy across validation cases. However, high-speed spindle failures above 18,000 RPM are only 61% accurate, and those failures are rare but high-cost.

What evaluation change should gate launch?

A) Keep the aggregate metric because it reflects the real-world distribution.

B) Use technician satisfaction surveys only, because field usability matters more than labeled validation.

C) Ask Claude to output confidence and ship if confidence is high on most cases.

D) Track accuracy by machine class and failure segment, create a high-speed spindle eval slice, and require that slice to meet a launch threshold.

Correct Answer: D

Aggregate accuracy can hide poor performance on rare, high-impact segments. The evaluation should measure performance by segment and gate launch on the critical slice, not rely on self-confidence or delayed survey feedback.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md; docs/Claude API/Build/Test and evaluate/Define success criteria and build evaluations.md

