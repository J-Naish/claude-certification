# Scenario 1: Developer Productivity with Claude

Scenario: **Tessera Analytics** is a 600-person data-analytics SaaS company rolling out Claude across engineering. The platform team is building **Athena**, an internal Claude-API-powered "codebase Q&A" agent that engineers can chat with to ask questions like "where does the billing webhook handle disputes?" or "explain the retry logic in the ingestion worker." Athena has access to tools: `search_code(query)`, `read_file(path, range)`, `git_log(path, limit)`, `find_definition(symbol)`, `list_tests(path)`. Separately, the team is standardizing Claude Code configuration for all engineering teams.

Part of a full-length CCAF Foundations mock exam (4 scenarios × 15 questions = 60). Pass threshold: 43/60 (72%).

---

## Question 1

The platform team is implementing the Athena agent loop in Python against the Claude API. An engineer drafts the loop as:

```python
response = client.messages.create(model=..., tools=TOOLS, messages=msgs)
while response.content and any(b.type == "tool_use" for b in response.content):
    tool_results = run_tools(response.content)
    msgs.append({"role": "assistant", "content": response.content})
    msgs.append({"role": "user", "content": tool_results})
    response = client.messages.create(model=..., tools=TOOLS, messages=msgs)
return response
```

A senior reviewer flags the loop condition as fragile. Which is the most accurate critique?

A) The loop should append the tool results to `msgs` as a `role: "tool"` message, not `role: "user"`; using `role: "user"` will be rejected by the API for tool-result content.

B) The loop should check `response.stop_reason == "tool_use"` instead of scanning content blocks; `stop_reason` is the explicit API-level signal for "I need tool results to continue."

C) The loop should call `client.messages.create` with `tool_choice: {type: "auto"}` explicitly; without it, Claude will never emit `tool_use` blocks.

D) The loop should re-send the full original system prompt on every iteration via the `system` parameter; otherwise context from the system prompt is dropped after the first request.

Correct Answer: B
`stop_reason` is the documented, authoritative loop signal. Scanning content blocks is indirect — it works most of the time but can desync with model behavior, and it does not distinguish "I'm done" from "I hit max_tokens mid-toolcall." A is wrong because the Claude API does NOT use `role: "tool"`; tool results are passed back as `role: "user"` messages whose `content` is a list of `tool_result` blocks. C is wrong because `tool_choice` defaults to `"auto"` when `tools` are provided. D is wrong because while you do need to re-send `system` each call (the API is stateless), that is a separate concern and not the loop-condition critique the reviewer is flagging.

---

## Question 2

Athena's logs show a recurring failure: when an engineer asks "explain how the ingestion worker handles backpressure," Athena calls `search_code("backpressure")`, gets back 47 matches, then immediately calls `read_file` on the *first* result without inspecting the rest. The first result is often a stale test fixture or a config file, leading to incorrect answers. The 30+ relevant code matches are ignored.

What is the most effective single change to fix this?

A) Replace `search_code` with a vector-similarity search so the first result is always semantically most relevant; ranking quality is the root cause.

B) Update Athena's system prompt to instruct it to scan all `search_code` results, identify the 2–4 most likely relevant files based on filename/path heuristics, and read those *in parallel* before forming an answer.

C) Wrap `search_code` so it only ever returns the top 1 result; reducing optionality forces Athena to use what it has.

D) Add a `PreToolUse` hook that rejects `read_file` calls until Athena has spent at least 5 seconds reasoning about the search results.

Correct Answer: B
The failure mode is behavioral, not infrastructural — Athena is skipping the triage step. A prompt change that prescribes "scan → triage → parallel read" addresses the root cause and additionally exploits parallel tool calling to reduce round-trips. A is wrong because better ranking helps but does not fix Athena's habit of taking only result #1. C is wrong because capping to 1 result is a regression — it eliminates the ability to cross-reference, which is exactly what is needed for nuanced questions. D is wrong because `PreToolUse` hooks decide allow/deny based on the call's contents, not on elapsed time.

---

## Question 3

The platform team wants Athena to handle two qualitatively different question types:

1. **Quick lookup** ("where is `compute_invoice_total` defined?") — should answer in 1–2 tool calls.
2. **Deep investigation** ("trace how a single ingested event flows from S3 to the warehouse, including retry behavior") — may require 30+ tool calls across many files.

A junior engineer proposes building two completely separate agents (`AthenaQuick` and `AthenaDeep`) with different system prompts, different tool sets, and a manual router that classifies the incoming question.

What is the strongest argument *against* this two-agent split?

A) Building two separate agents doubles maintenance cost and introduces classification error at the router; a single agent with a well-designed system prompt and full tool set can adapt its depth of investigation to the question naturally.

B) Two separate agents will produce inconsistent answer styles, confusing engineers; uniformity matters more than depth-appropriateness.

C) The Claude API does not allow more than one agent per API key, so the architecture is technically infeasible.

D) Deep-investigation agents always require the Message Batches API for cost reasons, and the Batches API cannot interactively answer "quick lookup" questions in real time, so they cannot share infrastructure anyway.

Correct Answer: A
This is the canonical "don't over-decompose" lesson. A capable LLM with a good system prompt can scale its investigation depth to the question — that is the point of an agentic loop. Splitting forces a classification step (which is itself error-prone) and bifurcates the codebase, prompt, evals, and tool definitions. B is wrong because style inconsistency is a minor concern, not the strongest objection. C is wrong because there is no "one agent per API key" limit. D is wrong because deep investigations run fine in real time with the regular Messages API; Batches is for non-blocking, latency-tolerant workloads.

---

## Question 4

The platform team is adding a new tool to Athena: `run_sql(query)` against a read-only analytics replica so Athena can answer questions like "how many tenants had >10 ingestion errors last week?". A reviewer raises concerns about safety and reliability of the tool definition.

Which of these is the most important property for the `run_sql` tool definition to enforce *at the tool level* (not via prompt instruction)?

A) The tool's `input_schema` should mark `query` as required, with a description that lists allowed SQL verbs (SELECT only) and notes the read-only replica.

B) The tool's server-side implementation should validate that the parsed SQL is read-only (e.g., reject anything that is not a single `SELECT` statement) and run against a read-only connection; relying on description text alone is insufficient.

C) The tool's description should be made shorter and more terse so Claude does not get distracted by safety caveats and can focus on the query construction.

D) The tool's `input_schema` should accept the query as a base64-encoded blob to discourage casual misuse and force Claude to think harder before submitting.

Correct Answer: B
Safety properties must be enforced where they cannot be bypassed. Description text is a prompt-level hint — Claude usually follows it, but a single misinterpretation could issue a `DELETE`. A parser-level check plus a connection that physically cannot write is deterministic. A is partially correct (documenting "SELECT only" helps correct usage) but does not enforce safety on its own. C is wrong; terseness does not help safety. D is wrong; base64-wrapping is security-by-obscurity.

---

## Question 5

Tessera's platform team wants to expose internal company tools to Claude Code on every engineer's laptop — specifically:
- A `tessera-jira` MCP server (read/write Jira tickets)
- A `tessera-docs` MCP server (search internal Notion/Confluence)
- A `tessera-deploy` MCP server (trigger deploys; sensitive)

The team wants `tessera-jira` and `tessera-docs` to be available in every repo automatically, but `tessera-deploy` must only be available in the `deploy-tooling` repo and require an extra trust step.

Which configuration strategy correctly implements this?

A) Add all three servers to user scope (`claude mcp add --scope user ...`); rely on each engineer to manually disable `tessera-deploy` outside the deploy-tooling repo.

B) Add `tessera-jira` and `tessera-docs` at user scope (`--scope user` so they follow the engineer everywhere); commit a `.mcp.json` to the `deploy-tooling` repo declaring only `tessera-deploy` (project-scope), so it auto-loads only there and triggers Claude Code's first-run trust prompt.

C) Add all three servers to a global `~/.mcp.json` file; Claude Code reads this on every startup and the per-repo restriction is handled automatically by directory name matching.

D) Add `tessera-deploy` at user scope and `tessera-jira` + `tessera-docs` at project scope in every repo's `.mcp.json`; project scope is faster than user scope so the frequently-used servers should be there.

Correct Answer: B
User scope (`--scope user`) follows the engineer into every directory — right for "always available." Project-scope `.mcp.json` is repo-bound and only activates when Claude Code is launched inside that repo, and the first-run trust prompt for project servers gives the extra confirmation step deploy operations deserve. A is wrong because relying on engineers to manually disable a sensitive server is fragile. C is wrong because there is no `~/.mcp.json` magic file with directory-matching semantics. D is wrong because project vs user scope has nothing to do with performance — it is a visibility/sharing setting.

---

## Question 6

The Athena team is reviewing the JSON descriptions of its 5 tools. A new hire suggests "compressing" each tool's description to under 30 characters to save context window and let Claude focus on user prompts.

Why is this the wrong instinct, in priority order?

A) Tool description tokens are billed at 2× the rate of message tokens, so compressing them is the most effective cost optimization; the new hire is correct on cost grounds but the team should still keep them ~80 chars for clarity.

B) The API rejects tool descriptions shorter than 64 characters with a validation error; the team would have to lengthen them to meet the schema regardless.

C) The model needs detailed tool descriptions to choose between tools correctly and to construct valid arguments; tool descriptions are a primary lever for reducing wrong-tool calls. Saved tokens are tiny relative to the cost of one botched tool call requiring a re-roll.

D) Tool descriptions under 30 characters get cached and never refreshed, so the team would have to wait 24 hours after each change to see updated behavior.

Correct Answer: C
The new hire's instinct optimizes the wrong axis. Tool description tokens are a tiny share of context, and they have outsized leverage on correctness: a well-written description prevents wrong-tool selection, malformed arguments, and follow-up clarification turns. One avoided botched tool call saves orders of magnitude more tokens than tightening every description. A is wrong; there is no 2× billing rate for tool descriptions. B is wrong; no 64-char minimum exists. D is wrong; tool descriptions are not subject to special caching.

---

## Question 7

Tessera's platform team is rolling out Claude Code to all engineers. They want **organization-wide defaults** that individual engineers cannot override (e.g., deny rule on `Bash(rm -rf /:*)`, mandatory `WebFetch` deny rules for competitor domains), plus team-level defaults per repo, plus the ability for each engineer to add personal allowances.

What is the correct precedence order Claude Code uses, from highest to lowest, that lets this work?

A) Managed (enterprise) settings → Local project settings → Project shared settings → User settings.

B) User settings → Local project settings → Project shared settings → Managed settings.

C) Local project settings → Project shared settings → User settings → Managed settings.

D) Project shared settings → User settings → Local project settings → Managed settings.

Correct Answer: A
The documented precedence is: **Managed → Command-line arguments → Local (`.claude/settings.local.json`) → Project shared (`.claude/settings.json`) → User (`~/.claude/settings.json`)**. Managed sits at the top so organizations can enforce policies that individual engineers cannot override; Local sits above Project shared because Local is for personal, gitignored overrides specific to one engineer's checkout. B, C, and D all place Managed in a position where it can be overridden, which defeats enterprise enforcement.

Reference: `docs/Claude Code/ Configuration/Settings and permissions/Settings.md`

---

## Question 8

Tessera's Data Science team has a repo `dsx-experiments` with notebooks, raw data exports, and dozens of throwaway scripts. The team wants Claude Code to:
- Auto-allow reading any file in the repo
- Auto-allow running `uv run pytest` and `uv run python` against `.py` files
- *Never* auto-allow `Bash(rm:*)` or `Bash(git push:*)` — these always need explicit approval

Where should these team-wide rules live?

A) The repo's `.claude/settings.json` (checked into the repo), with `permissions: {allow: [...], ask: [...]}`; this is the canonical place for team-wide, version-controlled permission rules.

B) Each engineer's `~/.claude/settings.json`, since they will be the ones approving prompts anyway.

C) The repo's `.claude/settings.local.json`, since `local` is more authoritative than `project`.

D) A `permissions:` block in the repo's root `CLAUDE.md`; Claude Code parses CLAUDE.md for permission rules at session start.

Correct Answer: A
`.claude/settings.json` is the canonical home for team-wide, version-controlled permission rules. Being checked in means it is code-reviewed and consistent across the team. Note that "never auto-allow but always need explicit approval" maps to the `ask` rule (not `deny`, which would block entirely): the three buckets are `allow` (auto-approve), `ask` (always prompt), and `deny` (block). B is wrong because team rules in a private home dir are not shared. C is wrong because `settings.local.json` is gitignored by default — personal overrides, not team config. D is wrong because CLAUDE.md is a memory file, not a settings file.

---

## Question 9

Athena occasionally goes off the rails: an engineer asks a simple question and Athena reads 40+ files, hits the model's `max_tokens`, and produces a bloated response. The platform team wants a safety net that *bounds runaway behavior* without preventing legitimate deep investigations on questions that warrant them.

Which defense is the best fit?

A) Cap the agent loop at a fixed maximum iteration count (e.g., 25 tool-use rounds), and on cap-hit, ask the agent to summarize what it has found so far and answer. Combine with an explicit prompt rule: "for simple lookups, answer in 1–2 tool calls."

B) Set `max_tokens` very low (e.g., 500) on every request so Athena cannot produce long responses; long responses are the problem.

C) Have an external monitor process kill the agent after 60 seconds of wall-clock time regardless of progress.

D) Disable parallel tool calls; serializing tool calls forces Athena to think between each one, reducing runaway behavior.

Correct Answer: A
The right pattern combines an upper bound (defense in depth) with a graceful degradation (still produce something useful when the cap fires) and a prompt-level nudge for the common case. This protects without hard-capping legitimate deep work. B is wrong because a 500-token cap cripples legitimate long answers and targets output length, not the behavioral root cause. C is wrong because a wall-clock kill is coarse and produces no useful output. D is wrong because parallel tool calls are a positive feature; disabling them does not bound depth, just slows runaways.

---

## Question 10

Athena's product manager wants Athena to return its answer in a consistent structured shape so a frontend can render it cleanly:

```
{
  "summary": "<one-paragraph answer>",
  "files_referenced": [{"path": "...", "lines": "..."}],
  "confidence": "high" | "medium" | "low"
}
```

The platform engineer is choosing how to enforce this shape against the Claude API.

Which approach is most reliable?

A) Add a final assistant turn before the user's question that says: "I will always respond in this JSON schema: ..." and include the schema verbatim; Claude will follow assistant-turn instructions more reliably than system-prompt instructions.

B) Switch to the Message Batches API, which validates structured responses against an attached JSON schema before returning them to the caller.

C) Instruct Claude in the system prompt to emit JSON and then run `json.loads` on the response in a `try/except`; on failure, retry up to 3 times with a "your previous response was invalid JSON" follow-up.

D) Define a tool called `answer` with `input_schema` matching the desired shape, and on the final response turn (when Athena has gathered enough info), force `tool_choice: {type: "tool", name: "answer"}`; the structured output appears as the tool's `input` argument.

Correct Answer: D
Forced tool use with `input_schema` matching the shape is the canonical Claude API pattern for guaranteed structured output. The model must produce arguments matching the schema; there is no free-text fallback. For an agentic loop, a common pattern is `tool_choice: auto` on regular turns and `tool_choice: {type: "tool", name: "answer"}` on the final turn. A is wrong; there is no documented "assistant-turn instructions are more reliable" property. C is wrong because a retry loop adds latency/cost without first-response guarantees. B is wrong because the Batches API does not validate against schemas.

---

## Question 11

Tessera's engineering manager wants to measure whether Athena is actually helpful. The team is debating between three eval strategies:

| Strategy | Description |
|---|---|
| **A. Self-judge** | After each Athena answer, have a separate Claude API call judge the answer (using the same model) against the question; aggregate pass-rate. |
| **B. Golden set** | Hand-curate 50 questions with known-correct expected answers (with citations); run Athena against the set nightly; measure exact-match on citations + LLM-judged answer quality. |
| **C. User thumbs** | Add 👍/👎 buttons in the UI; treat thumbs-up rate as the eval signal. |

The manager wants the *primary* eval to drive shipping decisions. Which is most appropriate, and why?

A) Strategy A — self-judge. It scales arbitrarily, requires no human curation, and using the same model ensures consistent grading rubric.

B) Strategy A and C combined — automated breadth (A) plus real-user signal (C); strategy B is not worth the curation cost in a fast-moving environment.

C) Strategy C — user thumbs. End-user satisfaction is the only thing that matters; the other strategies measure intermediate proxies.

D) Strategy B — golden set. Known-correct expected answers + citation matching give a stable, reproducible signal independent of users and judge bias; LLM-judged quality is a supplementary signal on top.

Correct Answer: D
For shipping decisions you need an eval that is reproducible, stable, debuggable, and team-owned. Hand-curated golden sets satisfy all four — they are the canonical shipping-gate eval. A is wrong as a primary signal because a model judging the same model's output has well-documented bias (it tends to defend its own reasoning style); useful as a supplementary signal, not a gate. C is wrong as a primary signal because of selection bias (most users do not click), post-shipping latency (you cannot use thumbs to decide whether to ship), noisy attribution, and non-reproducibility in CI. B is wrong because it dismisses the only deterministic pre-ship signal.

---

## Question 12

The Athena team is reviewing a proposed new tool, `cite_pr(pr_number)`, that returns the title, description, and merged commits of a given PR. An eval engineer notes that Athena currently has `git_log(path, limit)` and proposes that `cite_pr` should be removed because `git_log` can reach the same data.

Which is the *strongest* reason to keep `cite_pr` despite the overlap?

A) The Claude API charges per tool invocation; having two tools spreads invocations and reduces cost.

B) Removing `cite_pr` would cause backwards-incompatibility with previously cached responses.

C) `git_log` returns plain text and `cite_pr` returns JSON; JSON is always preferred for structured downstream rendering.

D) `cite_pr` has a name that more clearly signals intent to Claude (citing a PR) than `git_log`, reducing wrong-tool selection on questions like "what changed in PR #4521?".

Correct Answer: D
When two tools overlap in capability, the name and description are what Claude uses to pick the right one. `cite_pr` reads as "the right tool when someone asks about a PR"; reaching the same data through `git_log` requires extra reasoning steps and is more error-prone. D well-scoped, well-named tool is worth carrying even if it is strictly a subset of another tool's capabilities. B is wrong; no such cached-response coupling exists. C is wrong; return format is implementation, not a fundamental difference. A is wrong; the API does not charge per tool invocation.

---

## Question 13

Tessera's mobile team uses a custom React Native framework with non-standard conventions. They want Claude Code, when working in `mobile/` subdirectory of the monorepo, to follow those conventions, but when working in `services/` (Go backend) to follow Go conventions. The monorepo has a root `CLAUDE.md` for org-wide guidance.

Which approach achieves directory-aware conventions cleanly?

A) Put a `CLAUDE.md` in `mobile/` and another in `services/`; Claude Code automatically loads the CLAUDE.md closest to the files being worked on, in addition to the root CLAUDE.md.

B) Put all conventions in the root `CLAUDE.md` under headers like "## When working in mobile/" and "## When working in services/", and rely on Claude to apply the right section based on file paths it sees.

C) Use `.claude/rules/` directory with separate rule files that have `paths:` frontmatter (e.g., `mobile-rules.md` with `paths: ["mobile/**/*"]`); Claude Code loads matching rule files based on the files in context.

D) Both A and C are valid and complementary: nested CLAUDE.md for prose conventions, `.claude/rules/` for path-targeted rules that activate on specific file patterns.

Correct Answer: C
`.claude/rules/` is Claude Code's purpose-built mechanism for path-based conditional rule application. Each rule file has YAML frontmatter with a `paths:` glob; when files matching that pattern are read, the rule activates. This is deterministic, path-aware, and works regardless of where the engineer's cwd is in the monorepo. The official exam guide's Sample Question 6 confirms this as the canonical pattern. A is plausible but directory-bound and less robust when files are spread across directories or when engineers work from the monorepo root. B is wrong because it relies on Claude inferring "which section applies." D is a plausible distractor but duplicative; `.claude/rules/` handles both directory- and pattern-based cases.

Reference: `Official Exam Guide/Foundations Certification Exam Guide.md` (Sample Question 6)

---

## Question 14

The platform team observes that Athena's `read_file` tool is sometimes called with `range` parameters that exceed the file's length (e.g., `lines: "1-2000"` on a 200-line file). The tool currently returns an error string `"Error: lines 800-2000 are out of range; file has 200 lines"`. Athena, on seeing the error, often re-issues the same call with the same out-of-range parameters.

What is the most effective fix at the *tool-implementation* level?

A) Suppress the error and return an empty string when the range is out of bounds; this prevents the error from re-entering Athena's context and causing the loop.

B) Raise an exception instead of returning an error string; the API surfaces exceptions more visibly than tool-result text.

C) Change the tool implementation to clamp the requested range to the file's actual length and return the in-range content, along with a structured note indicating the clamp (e.g., `{"content": "...", "clamped_to": "1-200", "requested": "1-2000"}`); the structured response gives Claude the information needed to adjust without ambiguity.

D) Add a `max_lines` parameter to the tool with a default of 100, forcing Athena to request smaller chunks; out-of-range becomes impossible.

Correct Answer: C
The right principle for tool design under LLM callers: be forgiving and informative. Do what the caller probably meant (return the lines that exist), and surface what happened in a structured form Claude can act on. This converts a failure into a useful, self-describing result. A is wrong because silent failures are worse than errors; Claude has no signal to course-correct. B is wrong because exceptions and error strings both arrive as `tool_result` blocks with `is_error: true`; the format does not change Claude's behavior. D is wrong because it forces tiny chunks (excessive round trips) and treats the symptom rather than the cause.

---

## Question 15

Athena answers questions about the codebase, often spanning many file reads. A user asks: "trace how a single event flows from ingestion to the warehouse, listing every transformation." Athena does 22 tool calls reading source files. By the time it composes the answer, it has forgotten that the user specifically asked it to list every transformation step in order and instead produces a narrative summary that skips three transformations and reorders two.

Which intervention is most likely to fix this *without* changing the model or the agent's tool set?

A) Lower the temperature to 0 so Athena's output becomes deterministic.

B) Increase `max_tokens` so Athena has more room to think out loud in the final answer.

C) Have Athena keep a running "transformation list" scratchpad: after each file read, append any transformations it found to a structured `findings` block in its own assistant turn; the final answer is composed by reading from the structured findings, not by re-deriving from memory of 22 tool calls.

D) Add a system-prompt instruction "remember the user's original question" — instruction repetition is the standard remedy for forgetting.

Correct Answer: C
Externalizing findings into an explicit, structured form as the agent works lets the final composition step read from explicit state rather than re-deriving from a long, low-attention context. The structured findings sit at the end of the context window (high-attention region) right before the final answer. This is the canonical mitigation for state degradation across many tool calls — the exam guide calls it out under Domain 5: "Having agents maintain scratchpad files recording key findings, referencing them for subsequent questions to counteract context degradation." A is wrong; temperature controls randomness, not memory. B is wrong; `max_tokens` controls output length, not attention. D is wrong; prompt-level repetition is probabilistic and does not address the structural issue.

Reference: `Official Exam Guide/Foundations Certification Exam Guide.md` Domain 5 task statements
