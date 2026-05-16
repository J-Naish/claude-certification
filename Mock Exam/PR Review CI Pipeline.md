# Scenario 2: Claude Code for Continuous Integration

Scenario: **Vector Logistics**, a 250-engineer freight-tech company, is rolling out Claude Code into its CI/CD pipeline. The platform team is wiring Claude Code (via the headless mode) into GitHub Actions workflows to: (a) review pull requests automatically and post review comments, (b) generate release notes from merged PRs, (c) audit security-sensitive code paths on every PR, and (d) triage flaky test failures. The CI runs on self-hosted GitHub Actions runners with limited credentials.

Part of a full-length CCAF Foundations mock exam (4 scenarios × 15 questions = 60). Pass threshold: 43/60 (72%).

---

## Question 1

Vector's platform engineer is writing the GitHub Actions step that invokes Claude Code to review a PR diff and post comments. The first attempt uses:

```yaml
- name: Review PR
  run: claude "Review the changes in this PR and post comments via gh pr review"
```

The job hangs indefinitely. Logs show Claude Code is waiting for input. What is the minimum correct change?

A) Add `--interactive=false` to force a non-interactive session.

B) Use `claude -p "Review the changes in this PR and post comments via gh pr review"` — the `-p` (`--print`) flag is Claude Code's headless / non-interactive mode for CI usage.

C) Switch to the Message Batches API — Claude Code's interactive mode cannot run in CI runners due to TTY requirements.

D) Wrap the call in `expect` to auto-respond to the input prompts; this is the standard pattern for running interactive CLI tools in CI.

Correct Answer: B
`-p` (`--print`) makes Claude Code take a single prompt, run to completion, print the response, and exit. No TTY, no waiting for input. This is the canonical CI invocation. A is wrong because there is no `--interactive=false` flag; the non-interactive mode is `-p`/`--print`. C is wrong because the Message Batches API is an API-side feature with up-to-24h latency, which would break PR review workflows. D is wrong because wrapping with `expect` is a hack that fights the tool — the tool already has a proper headless mode.

Reference: `docs/Claude Code/Getting started/Use Claude Code/Headless mode.md`

---

## Question 2

Vector wants the PR-review job to emit a JSON payload that a downstream Slack notifier can parse, rather than free-form prose. The current invocation is:

```yaml
- run: claude -p "Review the diff and report findings"
```

Which change correctly produces machine-parseable JSON output without writing a parsing script?

A) Add `--json` and instruct Claude in the prompt to "Output JSON only, no markdown"; both flags are needed.

B) Add `--output-format json` to the invocation; Claude Code wraps the response in a JSON envelope with the result text and metadata fields the notifier can consume.

C) Pipe stdout through `jq '. | {result: .}'` — Claude Code does not natively emit structured output, so post-processing is required.

D) Wrap the prompt in `<output-format>json</output-format>` XML tags inside the prompt string; Claude Code reads these tags and adjusts its output mode.

Correct Answer: B
Claude Code's headless mode supports `--output-format json` (and `stream-json` for streaming) to wrap the response in a JSON envelope with the result text, usage, cost, and other metadata fields. The Slack notifier can `jq` into it without any custom parsing layer. A is wrong because there is no `--json` flag; the correct flag is `--output-format json`. C is wrong because Claude Code natively emits structured output via `--output-format`. D is wrong because there is no XML-tag input convention that changes Claude Code's output mode.

---

## Question 3

Vector's PR-review job currently produces too many false positives: it flags stylistic preferences as "issues" and re-flags the same patterns it flagged on the previous run. The team wants to reduce noise.

Which prompt-level change is most likely to reduce false positives without re-training or switching models?

A) Switch to a smaller, cheaper model; smaller models are more conservative and flag less.

B) Increase the model's `temperature` so it considers more options before flagging; current low-variance outputs over-flag.

C) Provide 3–5 *few-shot examples* in the system prompt: each shows a code snippet, what to flag (or not flag) for it, and a one-sentence reasoning. Target the examples at the ambiguous cases the team has seen flagged incorrectly.

D) Run the review three times and require all three runs to agree before posting a comment; consensus filtering eliminates false positives.

Correct Answer: C
Few-shot examples are the canonical fix for "this classifier/judge is mis-calibrated on a specific class of inputs." Showing what to *not* flag (with reasoning) gives the model a concrete contrast to learn from, far more reliably than abstract instructions like "be less aggressive." B is wrong because higher `temperature` introduces more randomness, not better calibration. A is wrong because a smaller model has different failure modes, not uniformly fewer flags. D is wrong because 3× consensus voting triples cost and latency, and runs sharing the same mis-calibration will flag the same false positives — consensus reduces *variance*, not systematic bias.

Reference: Official Exam Guide Domain 4 task statements (few-shot for ambiguous scenarios)

---

## Question 4

Vector's release-notes generator (a separate CI job) reads a list of merged PRs and produces a Markdown summary. It currently outputs a one-paragraph summary per PR, but the team wants:
- A short title (≤8 words)
- A category from `feature | fix | chore | breaking`
- A one-sentence summary
- A list of impacted services (subset of: `api`, `web`, `worker`, `etl`)

…and the output must be valid JSON the consumer can parse.

Which approach gives the most reliable per-PR structured output?

A) System prompt: "Output JSON only. Schema: `{title, category, summary, impacted_services}`." Then parse the response as JSON.

B) Use `--output-format json` on the Claude Code invocation; Claude Code automatically infers the schema from the prompt.

C) Define a tool `record_pr_summary` with `input_schema` matching the shape (title: string maxLength constraint; category: enum; summary: string; impacted_services: array of enum), and force `tool_choice: {type: "tool", name: "record_pr_summary"}` on each PR.

D) Ask Claude to fill out a Markdown template with placeholders; convert the filled template to JSON in a post-processing script.

Correct Answer: C
Forced tool use with `input_schema` matching the shape is the canonical Claude API pattern for guaranteed structured output. The model must produce arguments matching the schema; no free-text fallback. With enum constraints on `category` and `impacted_services`, the model literally cannot produce invalid values. A is wrong because prose JSON instructions are probabilistic. B is wrong because `--output-format json` wraps the Claude Code response envelope but does not enforce a schema on the inner content. D is wrong because Markdown templates + post-processing is brittle.

---

## Question 5

Vector's PR-review job runs `claude -p` on each opened PR and posts review comments. On a large PR (300+ files changed across services), the job:
- Takes 8+ minutes
- Posts contradictory comments (flags a pattern in `api/handlers.ts` then approves the identical pattern in `worker/handlers.ts`)
- Misses bugs that are obvious to a human reviewer in the changed files

The team is debating restructuring the review. Which is the most effective change?

A) Break the review into a *multi-pass architecture*: first pass identifies which files are most risky and need careful review, second pass reviews those files in depth (one subagent per file or per service), and a coordinator aggregates findings; this prevents context bloat and inconsistency from a single-pass 300-file review.

B) Switch to a more powerful model — the failures are caused by the current model's reasoning capacity.

C) Set `temperature: 0` so the review is deterministic and contradictions disappear.

D) Compress the diff before sending — request only the +/- lines without surrounding context.

Correct Answer: A
B single-context 300-file review causes the symptoms: contradictions (the model can't reliably hold judgments across a long context), missed bugs (attention dilutes), and latency (serial reasoning). Multi-pass + subagents fixes all three: triage isolates risk, per-file subagents work in isolated focused contexts (no cross-contradictions), the coordinator aggregates short summaries, and parallelism reduces wall-clock latency. B is wrong because model swap does not fix architectural problems. C is wrong because `temperature=0` makes contradictions *consistent across runs*, not absent within a run. D is wrong because stripping surrounding context makes review worse.

Reference: Practice Exam Q12 (CI scenario), exam guide Domain 5

---

## Question 6

Vector's security-audit CI job needs to call an internal vulnerability-scanning service (HTTP API) from within Claude Code. The platform team has two options:

- **(i)** Add the API as a custom `Bash` script the platform team maintains, called like `bash scripts/vuln_scan.sh <file>`, with the script handling auth, retries, and response formatting.
- **(ii)** Implement an MCP server `vector-vuln-scanner` exposing a `scan_file(path)` tool, configured per-repo via `.mcp.json`.

The team wants to choose based on which is more appropriate for *Claude Code's invocation model and reusability across multiple tools*.

Which factor most strongly favors the MCP server approach (option ii)?

A) MCP servers run in a sandboxed process Claude Code cannot see into, so they cannot be exploited by prompt injection; Bash scripts have full filesystem access and are unsafe.

B) MCP servers are billed at half the rate of Bash tool invocations in the Claude Code pricing model.

C) MCP servers automatically version their tools so backwards compatibility is preserved across Claude Code releases; Bash scripts must be manually versioned.

D) Claude Code natively understands MCP tool definitions (name, description, JSON schema for parameters) and can reason about when to invoke them; a Bash script appears as an opaque shell command that Claude must guess the calling convention for. Additionally, MCP servers are reusable across Claude Code, Claude Desktop, the API, and other MCP clients.

Correct Answer: D
The MCP advantage is fundamentally about typed integration: each MCP tool has a name, description, and JSON-schema'd parameters that Claude reads alongside its other tools. Claude can reason about *when* to call it instead of guessing shell-command syntax. The same server works in Claude Code, Claude Desktop, the API directly, and any other MCP client. A is wrong because MCP servers aren't inherently more prompt-injection-safe than Bash; safety depends on what the server does, not the protocol. C is wrong because there is no automatic-versioning mechanism in MCP. B is wrong because no per-invocation pricing model exists for either.

---

## Question 7

The `vector-vuln-scanner` MCP server exposes one tool: `scan_file(path: string) -> {issues: [{severity, cwe, line, description}]}`. In testing, Claude occasionally calls `scan_file("**/*.py")` (a glob) instead of an actual path, and the server returns an unhelpful `"file not found"` error.

Which tool-definition change most directly prevents this misuse pattern?

A) Add a second tool `scan_glob(pattern)` so Claude has both options; let it choose at runtime.

B) Allow glob patterns in `path`; the server expands the glob and scans all matching files. Convenience for Claude is more important than strict input semantics.

C) Rename the tool to `scan_single_file` to communicate the constraint via the name alone, leaving the description unchanged.

D) Document in the tool *description* that `path` must be a single file path, not a glob, and provide one inline example: `scan_file("services/api/auth.py")`. The description is what Claude reads when choosing how to call the tool.

Correct Answer: D
Tool descriptions are Claude's primary signal for how to call a tool. Adding "must be a single file path, not a glob" plus a concrete example directly addresses the misuse pattern. Examples in tool descriptions are especially effective — they show, rather than tell, the expected shape. B is wrong because silently expanding globs hides intent and causes surprising results. C is wrong because rename alone is a weaker signal than description + example. A is wrong because adding tools to handle every variation is tool sprawl and creates a new selection problem.

---

## Question 8

Vector's release-notes job classifies each merged PR title into `feature | fix | chore | breaking`. Some PR titles are obviously breaking (e.g., "Remove deprecated `/v1/orders` endpoint"), but the classifier currently labels them as `chore` because the title also mentions removing code. The team wants to fix this *via the prompt only*.

Which is the most effective intervention?

A) Train the team to write better PR titles; the classifier is downstream of human writing quality.

B) Switch the schema so the model returns *multiple labels* and let downstream consumers decide; single-label classification is fundamentally lossy.

C) Add a rule to the system prompt: "Any PR that removes a public API endpoint or changes its contract is `breaking`, even if the title sounds like cleanup." Include 2 few-shot examples of removal-titled PRs labeled `breaking` with the reasoning.

D) Add a follow-up clarifying question turn: "Is this breaking? (yes/no)" — Claude answers more accurately on yes/no than on multi-category classification.

Correct Answer: C
The failure mode is ambiguity — a PR title can plausibly match two categories. The fix is to make the boundary explicit: a rule naming the exact distinguishing criterion plus worked examples on the ambiguous boundary. This is the canonical few-shot pattern. B is wrong because it sidesteps the team's constraint (prompt-only) and offloads decision-making to downstream consumers. A is wrong because it is a longer-term process change, not a fix for the classifier now. D is wrong because if Claude is mis-judging "breaking-ness," it will mis-answer the yes/no the same way — the fix has to be in the criterion.

---

## Question 9

Vector's flaky-test-triage job uses a small agent: it reads a failing test's output, hypothesizes a cause (real bug, race condition, environment issue, dependency on test order), and labels the failure. The team observes that the agent often labels timeouts as "race condition" without checking whether the test even *uses* concurrency primitives.

Which architectural change is the strongest fix?

A) Increase the agent's reasoning budget via `max_tokens`; the agent labels too quickly because it's truncated mid-thought.

B) Give the agent a `read_test_source(path)` tool and explicitly instruct it to read the test source before labeling a race condition; require the agent to cite specific concurrency primitives observed in the source.

C) Switch to a smaller model with faster inference so the agent can re-run the flaky test 10 times and label based on observed variance.

D) Replace the agent with a regex over test output; agentic labeling is fundamentally unreliable for this task.

Correct Answer: B
The failure mode is unevidenced labeling — the agent reaches for a category that fits the symptom (timeout) without verifying the prerequisite. The fix grounds the label in evidence: the tool gives means to check, and the instruction + citation requirement enforces the check and makes the reasoning auditable. A is wrong because `max_tokens` controls output length, not reasoning quality. C is wrong because re-running 10× is expensive and does not address why the label was wrong. D is wrong because regex cannot distinguish "race condition" from "deadlock" or "slow downstream service."

---

## Question 10

Vector's CI runner has restricted credentials — it can read the repo, post PR comments, but cannot push to main or modify settings. The platform team wants to make sure Claude Code, running in CI, *cannot* attempt destructive operations even by mistake (no `rm`, no force-push, no settings modifications).

Which is the cleanest enforcement mechanism?

A) Set `permissions.deny` in `.claude/settings.json` (or via the CI-specific `--settings` argument) for `Bash(rm:*)`, `Bash(git push:*)`, `Edit(.claude/**)`, etc. Combined with CI credentials this is defense-in-depth: Claude Code blocks the call before it runs, and OS-level controls catch anything that slips through.

B) Trust the CI runner's OS-level credential restrictions; if Claude tries `git push --force`, GitHub will reject it. No Claude Code-level config needed.

C) Use plan mode (`--permission-mode plan`) so Claude can never modify files — plan mode is the right mode for CI.

D) Wrap the `claude` invocation in `sudo -u nobody` so any destructive operation fails with a permission error.

Correct Answer: A
The right model is "block at the highest layer that can recognize the intent." Claude Code permissions intercept the call before it runs; OS credentials are the backstop. Pairing both yields defense in depth. B is wrong because trusting only OS-level is fragile — Claude will try, fail, see the error, and may retry, wasting tokens. C is wrong because plan mode is too restrictive — it blocks `gh pr review`, the very operation the CI job needs. D is wrong because `sudo -u nobody` is a sledgehammer that strips legitimate capabilities along with destructive ones.

Reference: docs/Claude Code/ Configuration/Settings and permissions/Permissions.md

---

## Question 11

Vector's PR-review job (after the multi-pass restructure) has a coordinator that spawns one review-subagent per modified service. Some PRs span 6+ services. The coordinator runs each subagent sequentially because the engineer wasn't sure parallel subagent spawning was safe.

Which is the correct understanding for this scenario?

A) Subagents can be spawned in parallel — each has its own context window and there is no shared state to corrupt. Parallel dispatch is the standard pattern for independent work units like per-service review; it cuts wall-clock latency proportionally without changing correctness.

B) Subagents must always run sequentially because the Claude API does not support concurrent requests from the same API key.

C) Parallel subagents are safe but consume context window space on the coordinator while running; the coordinator may exceed its context budget if more than 3 run concurrently.

D) Parallel subagents are unsafe because their tool calls can collide on the filesystem; only sequential subagent execution avoids race conditions.

Correct Answer: A
Each subagent runs in its own context window; the coordinator only sees the final summary each subagent returns. Parallel dispatch for independent units is the canonical context-economy + latency-reduction pattern. B is wrong because there is no "one concurrent request per API key" rule. C is wrong because the subagent's context is separate from the coordinator's. D is wrong because review tools are read-only; subagent tool calls are independent processes and don't collide.

---

## Question 12

Vector's platform team wants to run the security audit, the PR review, and the release notes generation as **three separate Claude Code jobs** in CI rather than one giant prompt. Each runs with its own headless invocation, its own system prompt, and reports independently.

What is the strongest *architectural* argument for this separation?

A) Each job has a distinct objective, distinct success criteria, and benefits from a focused system prompt; bundling them dilutes Claude's attention and makes failures harder to attribute. Separation enables independent iteration, independent failure handling, and per-job evals.

B) Running them as one combined invocation would exceed the Claude API's per-request token limit; separation is a workaround for that limit.

C) The Claude API charges a fixed per-invocation fee, but the headless mode in Claude Code is exempt; running three headless jobs is cheaper than one combined one.

D) Combined invocations cannot use `--output-format json`; only single-purpose invocations support structured output.

Correct Answer: A
Focused prompts give Claude a clear objective; a combined prompt covering security + dependencies + release notes spreads attention thinly. Separation also enables independent iteration, independent failure handling, and per-job evals. This is the complement to "don't over-decompose" — do split unrelated objectives, don't split coherent tasks. B is wrong because token limits aren't the bottleneck. C is wrong because no per-invocation fee exists. D is wrong because `--output-format json` works regardless of bundling.

---

## Question 13

The flaky-test-triage agent sometimes labels a failure as "real bug" and Vector's auto-router opens a ticket. When engineers investigate, ~30% of these tickets turn out to be environmental issues (e.g., a stale DNS cache on the CI runner). The team wants to reduce false-positive tickets without losing real-bug detection.

Which pattern is the most appropriate?

A) Lower the agent's confidence threshold for "real bug" so it labels conservatively; fewer tickets is better than wrong tickets.

B) Add a verification step: after the agent labels "real bug," spawn an independent second agent (different prompt, no shared context) that re-examines the failure and casts an independent vote. Only open a ticket if both agree. This is the evaluator-optimizer / second-opinion pattern.

C) Replace the agent with a human triage rotation; LLM-based triage cannot reach the precision needed.

D) Have the agent self-critique its label in the same conversation turn — "Are you sure?" prompts reliably increase precision.

Correct Answer: B
An independent second agent with its own context and prompt approaches the failure fresh — no shared reasoning to defend, no anchoring on the first label. This is the canonical evaluator-optimizer pattern and eliminates the confirmation bias that plagues same-context self-critique. A is wrong because lowering the confidence threshold is vague and likely loses real bugs. C is wrong because human-only is expensive and pursued only when LLM precision genuinely cannot be reached. D is wrong because same-context self-critique suffers from confirmation bias — the exam guide explicitly calls this out under Domain 1.

Reference: Official Exam Guide Domain 1 task statements (evaluator-optimizer pattern)

---

## Question 14

The release-notes job needs to query the GitHub API to fetch the title, body, and author of each merged PR. The platform engineer is choosing between:

- (i) Letting Claude Code call `gh pr view` via the Bash tool with appropriate allow rules
- (ii) Adding the `github` MCP server (Anthropic-provided) which exposes typed tools like `get_pull_request`

For this *one-time CI workflow* that fetches PR data, which is the better choice and why?

A) Option (i) — Bash is *always* faster than MCP because MCP servers have higher per-call overhead.

B) Option (ii) — MCP is *always* preferable to Bash for any external service call; MCP servers are the canonical Claude Code integration mechanism.

C) Option (i) — `gh` via Bash. It avoids adding an MCP dependency for a job that calls one well-defined CLI; the `gh` CLI's output is already well-structured (with `--json`); MCP introduces unnecessary infrastructure for a single-purpose CI script.

D) Option (ii) — Bash tool invocations are billed at a premium rate per minute of execution time; MCP tools are free.

Correct Answer: C
The right choice depends on context. MCP shines for typed integration, cross-client reusability, and connecting to systems Claude cannot otherwise touch. For a one-off CI workflow calling one well-defined CLI that already emits structured JSON (`gh pr view --json ...`), Bash with a narrow allow rule (`Bash(gh pr view:*)`) is the minimal appropriate integration. MCP vs. Bash is a tool-selection decision, not a dogma. B is wrong because "always preferable" is the wrong framing — MCP has setup cost. A is wrong because there is no consistent speed advantage. D is wrong because no such per-minute billing model exists.

---

## Question 15

Vector's release-notes job runs at the end of each week. It reads ~40 merged PRs and produces release notes. The team notices that for weeks with many PRs, the release notes emphasize PRs from earlier in the week and only briefly mention PRs from later in the week — even though the prompt asks for balanced coverage.

What is the most likely cause, and the appropriate fix?

A) The prompt is too long; Claude truncates at the input limit. Fix: reduce per-PR context.

B) Claude's release-notes mode samples PRs proportionally to recency; this is configurable via the `recency_weight` parameter on `messages.create`.

C) The model's context-degradation always favors the *end* of the input; the team should reverse the PR order so important ones come last.

D) Position-based attention bias — Claude over-attends to early items because they're processed first into the running summary. Fix: present the PRs in a structured list (with explicit numbering and section headers), and instruct Claude to draft a one-line summary for each PR *before* composing the final notes; the per-PR summary acts as a normalized representation that the final composition reads from.

Correct Answer: D
The mechanism is position-based attention bias. The fix is to normalize the representation: force a one-line per-PR summary first (numbered list, headers), then compose the final notes by reading from the normalized list rather than re-deriving from raw PR descriptions. Each PR gets equal attention bandwidth, and the final composition reads from a uniform, recent (end-of-context) structure. A is wrong because 40 PRs is well within input limits. C is wrong because "always favors the end" is oversimplified; reordering just shifts which items get bad coverage. B is wrong because no `recency_weight` parameter exists.

Reference: Official Exam Guide Domain 5 ("lost in the middle" + position-mitigation patterns)
