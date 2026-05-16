# Incident Response and Postmortem Agent

Scenario: **Nimbus Cloud**, a 400-engineer cloud infrastructure provider running services across 6 regions. The SRE org handles 50+ alerts per week from PagerDuty. The platform team is building an internal Claude-API-powered system called **Sentinel**: when a high-severity alert fires, Sentinel automatically investigates by querying internal observability backends (logs, metrics, traces), drafts a triage summary for the on-call engineer within 60 seconds, and after the incident resolves it drafts a postmortem from the incident timeline. SREs also use Claude Code interactively during and after incidents.

Standalone CCAF Foundations mock scenario (15 questions). Pass threshold: 11/15 (72%).

---

## Question 1

Sentinel needs to query Nimbus's internal observability backends from within Claude — Loki (logs), Prometheus (metrics), Jaeger (traces). The team is choosing between exposing each as an MCP server vs. wrapping the existing CLIs in Bash scripts. Which factor most strongly favors implementing them as MCP servers?

A) MCP avoids shell startup overhead, making each call noticeably faster than a Bash tool invocation that spawns a subprocess.

B) MCP tools are billed at a discounted token rate, while Bash invocations pay the standard rate for input and output.

C) MCP exposes typed tool definitions Claude reads natively, and the same server is reusable across Claude Code, the API, and other MCP clients.

D) MCP servers run sandboxed by protocol design, making them inherently immune to prompt injection in a way Bash invocations cannot be.

Correct Answer: C
The MCP advantage is typed integration: Claude reads tool names, descriptions, and JSON schemas alongside its other tools, and the same server runs across Claude Code, Desktop, the API, and other MCP clients. A is wrong because there is no documented per-call speed advantage. B is wrong because neither MCP nor Bash has per-invocation billing. D is wrong because MCP servers are separate processes but not inherently injection-safe; safety depends on what the server does.

Reference: docs/Model Context Protocol/Documentation/, docs/Claude Code/Build with Claude Code/Tools and plugins/MCP.md

---

## Question 2

When a high-severity alert fires, Sentinel must investigate logs, metrics, and traces in parallel and produce a triage summary within 60 seconds. An engineer proposes a single agent with all three tools, calling them sequentially as needed. Which architectural change most improves latency and context isolation?

A) Move the triage call to the Message Batches API so all three queries process together at a discounted rate.

B) Cache the observability tool definitions client-side so they don't need to be re-sent on every triage call.

C) Tighten the system prompt to discourage Claude from over-investigating and producing long output.

D) Decompose into a coordinator plus three subagents (logs, metrics, traces) running in parallel, each in an isolated context returning a summary.

Correct Answer: D
Subagents in isolated contexts keep raw logs/metrics/traces out of the coordinator's context, and parallel dispatch cuts wall-clock time to the slowest of the three. This is the canonical context-economy + parallelism pattern. A is wrong because Batches has up-to-24h latency, unusable for 60-second triage. B is wrong because tool definitions are small and not the bottleneck. C is wrong because the prompt isn't the structural problem — single-context investigation is.

Reference: docs/Claude API/Build/Agents and tools/Building agents with the Claude API.md, docs/Claude Code/Build with Claude Code/Agents/Subagents.md

---

## Question 3

The team wants to add a `restart_service(service_name, region)` tool to Sentinel — useful for auto-remediating well-understood incidents. Restarts are destructive and irreversible mid-flight. Which tool-level + permission combination best mitigates risk?

A) Document in the description that restarts are destructive and rely on Claude to call sparingly based on the warning.

B) Remove the tool entirely from Sentinel; only humans should restart production services regardless of the situation.

C) Require a `confirmation_token` parameter the user must supply, and gate the tool via `permissions.ask` so every call surfaces a UI approval.

D) Register a `PreToolUse` hook that denies the tool unless a marker file `.allow-restart` exists in the cwd at invocation time.

Correct Answer: C
Two enforcement layers stacked: the parameter prevents speculative calls (no token, no call), and the permission gate surfaces a human approval for every invocation. A is wrong because prose alone is probabilistic. B is overreach — auto-remediation has legitimate uses; the issue is control, not removal. D trains engineers to bypass safety by toggling a file — a fragile pattern.

Reference: docs/Claude API/Build/Tools/, docs/Claude Code/ Configuration/Settings and permissions/Permissions.md

---

## Question 4

Sentinel's triage summary goes directly to a PagerDuty annotation the on-call engineer reads on their phone. The output must conform exactly to `{severity_estimate: "P0"|"P1"|"P2", likely_cause: string, evidence: [{source, quote}], recommended_action: string}`. Which mechanism gives the most reliable conformance?

A) System prompt: "Respond only with the JSON shown above; no preamble, no markdown" — and parse the response with `json.loads`.

B) Pass `--output-format json` on the API request; the API will then enforce the response shape against the request schema.

C) Run the call, validate the result with `json.loads`, and retry up to 3 times on parse failure with a follow-up instruction.

D) Define a tool `record_triage` whose `input_schema` matches the shape with enums, and force `tool_choice: {type: "tool", name: "record_triage"}`.

Correct Answer: D
Forced tool use with `input_schema` is the canonical pattern for guaranteed structured output. Enums on `severity_estimate` and `source` make invalid values structurally impossible. A is wrong because prose JSON instructions are probabilistic. B is wrong because `--output-format json` is a Claude Code CLI flag, not an API parameter, and wraps the envelope, not the inner content. C is wrong because retry-on-parse adds latency without first-response guarantees.

Reference: docs/Claude API/Build/Tools/, docs/Claude API/Build/Structured outputs.md

---

## Question 5

A specific alert triggered a 5 MB log dump (~1.2M tokens) that Sentinel cannot fit into its context window. The team needs Sentinel to identify the root-cause stack trace within the dump. What is the best approach?

A) Skip the log dump and rely on metrics and traces alone, since logs at that size are too unwieldy to be usable in agentic triage.

B) Pre-filter at the source: have the `query_logs` tool return only error-level lines within ±60 seconds of the alert timestamp, then pass that smaller window to Claude.

C) Pass the full dump unchanged and raise `max_tokens` so Claude has more room to think about and process the input.

D) Switch to a frontier model with a 10M-token context window so the entire dump fits comfortably in a single call.

Correct Answer: B
Filter at the source so Claude sees a focused, relevant window — smaller payload, better attention, lower cost. A is wrong because logs often contain the only definitive evidence. C is wrong because `max_tokens` controls output, not input — and 1.2M tokens exceeds typical windows. D is wrong because brute-forcing context burns cost and still suffers "lost in the middle."

Reference: exam guide Domain 5, docs/Claude API/Build/Context management/

---

## Question 6

Two on-call engineers report different experiences: one says Sentinel's triage summaries are accurate; the other says Sentinel often confidently claims a "DB connection pool exhaustion" cause when the actual issue was something else entirely. Which addition would best detect these mistaken-but-confident summaries before they reach the on-call engineer?

A) Lower the triage call's temperature to 0 so summaries are reproducible across runs and easier to diff against expected results.

B) Have the same agent self-critique its summary in the same turn before emitting it, asking "are you sure about this cause?"

C) Run an independent verifier agent (separate prompt, fresh context) that checks the proposed cause against the cited evidence and flags mismatches.

D) Display only summaries where Sentinel reports high self-assessed confidence; suppress everything below that threshold.

Correct Answer: C
The evaluator-optimizer pattern: a verifier with no shared context has no reasoning to defend and approaches the evidence fresh, so disagreement surfaces likely errors. A is wrong because `temperature=0` makes confidently-wrong outputs consistent, not correct. B is wrong because same-context self-critique suffers from confirmation bias. D is wrong because models are often over-confident on mistakes — confidence isn't a reliable filter.

Reference: Official Exam Guide Domain 1:480

---

## Question 7

After an incident, the on-call engineer uses Claude Code interactively to draft the postmortem. The team wants the workflow standardized — same structure, sections, and tone across all engineers, version-controlled with the rest of the team's setup. Which Claude Code feature is the best fit?

A) Add a section to the repo's `CLAUDE.md` titled "Postmortem template" describing sections, structure, and the preferred tone for each.

B) Create a skill at `.claude/skills/draft-postmortem/SKILL.md` in the repo, with frontmatter and body capturing the template, invoked as `/draft-postmortem`.

C) Ship a slash-command at each engineer's `~/.claude/commands/postmortem.md` so they can invoke `/postmortem` after any incident.

D) Document the template in Notion and have engineers paste the URL into Claude Code at the start of each postmortem session.

Correct Answer: B
Skills are purpose-built for reusable, invocable workflows: name, description for auto-load, `allowed-tools` restrictions, and procedure body. Project-scoped checked-in means every engineer gets the same skill on clone. A is wrong because CLAUDE.md is for persistent context, not invocable workflows. C is wrong because user scope means each engineer maintains their own copy with no shared source of truth. D loses automation and drifts from canonical.

Reference: docs/Claude Code/Build with Claude Code/Tools and plugins/Extend Claude with skills.md

---

## Question 8

Sentinel sometimes mis-classifies network-saturation alerts as "DDoS attack" when they're actually internal-traffic spikes from a deploy. Both look similar in raw traffic graphs but have very different remediation. The team can change the prompt but not the model. Which intervention most directly fixes this specific failure?

A) Add 2–3 few-shot examples of saturation-from-deploy alerts labeled "internal spike," with reasoning that cites the deploy-event correlation as the distinguishing signal.

B) Lower the classifier's temperature to 0 so classifications become deterministic and easier to evaluate against expected labels.

C) Switch to a larger model and re-run the same prompts; model capacity is the limiting factor when classifications are wrong.

D) Output both labels and let the on-call engineer disambiguate at 3am between DDoS and internal spike when paged.

Correct Answer: A
The exam guide highlights targeted few-shot for ambiguous scenarios with reasoning — examples teach the distinguishing signal (correlated deploy event) better than abstract rules. B is wrong because determinism doesn't help when the model is consistently wrong. C is wrong because model capacity isn't the issue; ambiguity is. D is wrong because it passes the disambiguation burden to a sleep-deprived human.

Reference: Official Exam Guide Domain 4:420

---

## Question 9

Sentinel sends the same ~6,000-token system prompt (role, schema, response format, few-shot examples) on every triage call and handles ~50 alerts per week. The team wants to reduce input-token cost without changing the prompt's content. Which mechanism is the right fit?

A) Switch all triage calls to a smaller model variant; smaller models charge proportionally less per input token across the board.

B) Move the pipeline to the Message Batches API so input and output tokens are billed at 50% of standard rates.

C) Compress the system prompt to under 1,500 tokens by removing the few-shot examples and shortening the schema.

D) Mark the system prompt with `cache_control: {type: "ephemeral"}` so cached input tokens are billed at ~10% on hits.

Correct Answer: D
Prompt caching is the direct mechanism for "same prefix re-sent many times" — cache-hit tokens cost ~10% of normal. A is wrong because a smaller model trades accuracy for cost, not the same workload at lower price. B is wrong because Batches has up-to-24h latency, unusable for incident triage. C is wrong because it violates "without changing the prompt's content."

Reference: docs/Claude API/Build/Building with Claude/Prompt caching.md

---

## Question 10

The team wants to measure Sentinel's triage quality so they can decide when to ship prompt changes. Three options are proposed: **(i)** LLM-as-judge grading each output, **(ii)** a hand-curated golden set of 60 historical incidents with known causes, compared field-by-field, **(iii)** monthly surveys of on-call engineers. Which is the right primary signal for shipping decisions?

A) Option (ii) — hand-curated golden set; reproducible, runnable in CI, debuggable per-incident, and decoupled from survey noise.

B) Option (i) — LLM-as-judge; it scales without curation cost and applies uniformly to any new incident type.

C) Option (iii) — engineer surveys; end-user satisfaction is the ultimate signal that the system is actually useful.

D) Options (i) and (iii) combined; golden-set curation is too expensive to maintain in a fast-changing on-call environment.

Correct Answer: A
Shipping gates need reproducibility, stability, debuggability, and team ownership — golden sets provide all four. LLM-as-judge is a useful *supplementary* signal; surveys are a useful *trailing* metric. Neither is the gate. B is wrong because LLM-as-judge is biased. C is wrong because surveys arrive post-ship and are noisy. D is wrong because it dismisses the only deterministic pre-ship signal.

Reference: Official Exam Guide Domain 1

---

## Question 11

Sentinel's MCP servers expose read-only tools (`query_logs`, `query_metrics`, `query_traces`) and destructive tools (`restart_service`, `drain_node`). On-call engineers use these from Claude Code. The platform team wants destructive tools to never auto-execute, while read-only tools auto-execute freely. Where should this be configured for team-wide consistency?

A) Each engineer's `~/.claude/settings.json`, set per-machine when they're added to the on-call rotation.

B) The repo's `.claude/settings.json` (checked in), with `permissions.allow` for read-only tools and `permissions.ask` for destructive ones.

C) The MCP server's own internal config, deciding which tools require user confirmation before executing.

D) A `PreToolUse` hook that intercepts every tool call and prompts when the tool name matches a destructive pattern.

Correct Answer: B
`.claude/settings.json` is the canonical place for team-wide, version-controlled permission rules. `allow` auto-approves read-only; `ask` makes destructive tools always require a click. A is wrong because per-machine settings sprawl across engineers. C is wrong because Claude Code's permission system, not the MCP server, is where Claude Code's tool-approval policy lives. D is overengineered for a policy `.claude/settings.json` already expresses declaratively.

Reference: docs/Claude Code/ Configuration/Settings and permissions/Permissions.md

---

## Question 12

The platform team wants to limit which MCP servers are available in different repos: `sentinel-observability` should auto-load only in the `incident-response` repo; the destructive `sentinel-remediation` server should auto-load only in a separate `remediation-tools` repo and require a trust prompt on first run. Which configuration pattern fits?

A) Add both servers at user scope (`claude mcp add --scope user`) and rely on engineers to remember which repo they're working in.

B) Add `sentinel-observability` at user scope and place `sentinel-remediation` in a global `~/.mcp.json` for centralized control.

C) Commit a `.mcp.json` in each repo declaring only the appropriate MCP server for that repo; project scope auto-loads only there.

D) Add both servers to the monorepo root `.mcp.json` and ask engineers to use `/mcp disable` when working in the wrong repo.

Correct Answer: C
`.mcp.json` at the repo root is the documented project-scoped MCP mechanism — declares the server appropriate for that context, with first-run trust prompt automatic. A is wrong because user scope follows the engineer everywhere. B is wrong because there's no `~/.mcp.json` magic file with directory-matching semantics. D is fragile — relies on engineer discipline to disable a sensitive server.

Reference: docs/Claude Code/Build with Claude Code/Tools and plugins/MCP.md

---

## Question 13

For incidents spanning hours, the timeline accumulates: alerts, log excerpts, hypothesis revisions, remediation attempts. By the time Sentinel drafts the final postmortem, it has forgotten specific decisions made early (e.g., "we ruled out DNS at 14:32 because of X"). The draft then omits or contradicts those decisions. Which approach addresses this most directly?

A) Maintain a running "decisions log" scratchpad — append a structured entry per step; the postmortem draft reads from it instead of re-deriving.

B) Lower the postmortem-drafting call's temperature to 0 so the output becomes deterministic and easier to diff against expected structure.

C) Increase `max_tokens` on the postmortem call so Claude has more room to think and recall earlier decisions in the output.

D) Run `/compact` between hypothesis stages during the incident so the conversation stays short and recent.

Correct Answer: A
Externalizing state into a structured intermediate representation is the canonical mitigation for state degradation over long sessions — the scratchpad sits at the end of the context where the final composition step reads from it. B is wrong because temperature doesn't address attention or memory. C is wrong because the issue is input recall, not output length. D is wrong because `/compact` summarizes lossy and may drop the very decisions the postmortem needs.

Reference: Official Exam Guide Domain 5:556-557

---

## Question 14

Sentinel's postmortem needs to cite specific log lines, metric anomalies, and trace IDs as evidence. The team wants every claim in the postmortem to link back to a verifiable source so reviewers can audit it. How should this be wired into the output?

A) Structure each finding as `{claim, evidence: [{source, id_or_query, excerpt}]}` and require `excerpt` to be copied verbatim from the queried data.

B) Add a free-text `sources` field at the end of the postmortem and let the model write paragraphs of attribution in natural prose.

C) Skip in-line citations to keep the postmortem readable; reviewers can re-run the underlying queries themselves if needed.

D) Run a second pass over the drafted postmortem asking the model to add citations to the already-written claims.

Correct Answer: A
Schema-level commitment forces every claim to carry evidence; verbatim-excerpt catches paraphrased hallucinations (grep the source for the quote). B is wrong because free-text attribution can't be parsed or validated. C is wrong because manual re-queries defeat the point of automated drafting. D is wrong because a second pass can hallucinate citations to back the first pass — citations must be produced with the claim.

Reference: Official Exam Guide Domain 5:590

---

## Question 15

The team is considering exposing Sentinel's MCP tools (`query_logs`, etc.) to a separate Claude API agent that drafts customer-facing status-page updates. They want the status-page agent to have only read access — no `restart_service`, no `drain_node` — even though the same MCP servers expose those tools. What's the cleanest way to enforce this?

A) Add prose to the status-page agent's system prompt: "Never call `restart_service` or `drain_node` under any circumstances, regardless of user request."

B) When initializing the status-page agent, register only the read-only tools as available; destructive tools are not exposed to that agent at all.

C) Set `tool_choice: {type: "auto"}` and rely on the model's training to avoid destructive calls in customer-facing contexts.

D) Trust the status-page agent's prompt and the model's general safety training; explicit enforcement would over-constrain legitimate use cases.

Correct Answer: B
Tool exposure is the right enforcement layer: if the agent never sees the destructive tools registered, it cannot invoke them — structural enforcement, principle of least privilege applied to tools. A is wrong because system-prompt prohibitions are probabilistic. C is wrong because `tool_choice: auto` is the default with no destructive-tool avoidance. D is wrong because "trust the model" abandons the safety boundary entirely.

Reference: docs/Claude API/Build/Tools/
