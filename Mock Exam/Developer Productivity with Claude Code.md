# Developer Productivity with Claude Code

Scenario: Helios Systems — a 200-engineer fintech org rolling out Claude Code across product teams to boost developer productivity. The platform team owns onboarding, repo-level config, MCP integrations with internal services (Jira, design docs), and headless automation for CI utility scripts.

Length: 15 questions. Language: English. Difficulty: standard. Pass threshold: 11 / 15 (72%).

Domain mix (matches the CCAF Foundations weights):
- Domain 1 (Agentic Architecture & Orchestration): 4
- Domain 2 (Tool Design & MCP Integration): 3
- Domain 3 (Claude Code Configuration & Workflows): 3
- Domain 4 (Prompt Engineering & Structured Output): 3
- Domain 5 (Context Management & Reliability): 2

---

## Question 1

Helios Systems wants every engineer in the `payments-service` repo to follow the same coding conventions when working with Claude Code, while also allowing each engineer to keep personal preferences (e.g., preferred test runner output verbosity, preferred commit-message tone). The platform team is deciding where to put each kind of guidance.

Which placement strategy correctly leverages Claude Code's memory file hierarchy?

A) Put both repo conventions and personal preferences in `~/.claude/CLAUDE.md` so every project inherits them; use `./CLAUDE.md` only for repo-specific build commands.

B) Put repo-wide conventions in `./CLAUDE.md` checked into the repo, and personal preferences in `~/.claude/CLAUDE.md`; Claude Code loads both and the project file takes precedence on conflicts.

C) Put repo-wide conventions in `./.claude/settings.local.json` and personal preferences in `./CLAUDE.md`; the `settings.local.json` file is the only one that is gitignored by default.

D) Put all guidance in a single `./CLAUDE.md` and have each engineer fork the repo to maintain personal overrides; Claude Code only loads one memory file per session.

Correct Answer: B
`./CLAUDE.md` is checked into the repo and shared with the team (repo conventions). `~/.claude/CLAUDE.md` is user-global and applies across all projects (personal preferences). Both are auto-loaded; project-level memory takes precedence on conflicts since it's more specific to the current work. A is wrong because putting repo conventions in `~/.claude/CLAUDE.md` defeats team sharing — each engineer's home dir is private. C is wrong because `settings.local.json` is for settings (permissions, env), not prose guidance — CLAUDE.md is the memory file. D is wrong because Claude Code loads multiple memory files (enterprise, project, user, plus imported `@path/file.md`), not just one.

---

## Question 2

A platform engineer at Helios is authoring a Claude Code skill called `db-migration-review` that should inspect Postgres migration files and flag risky operations. The skill must be allowed to read files and run `psql --version` checks, but must NOT be allowed to write files or run arbitrary shell commands beyond a small whitelist. The engineer is writing the skill's frontmatter.

Which configuration correctly enforces this restriction?

A) Add `allowed-tools: Read, Bash(psql --version:*)` to the skill's YAML frontmatter; this restricts the skill's tool surface to only those tools while it is active.

B) Add `permissions: {allow: ["Read", "Bash(psql --version:*)"], deny: ["Write", "Bash"]}` to the skill's frontmatter — skills inherit the same permissions schema as `settings.json`.

C) Skip the frontmatter restriction; instead, rely on prose in the skill body that instructs Claude not to write files or run other commands.

D) Use `tools: ["Read"]` in the frontmatter and call `psql --version` via Read by reading `/dev/stdout`, since skills cannot grant fine-grained Bash permissions.

Correct Answer: A
Skills support `allowed-tools` in YAML frontmatter to restrict the tool surface while the skill is active. The Bash-pattern syntax `Bash(psql --version:*)` allows that exact command without granting full Bash access. B is wrong because skills do not use the `permissions: {allow, deny}` object — that schema belongs to `settings.json`. C is wrong because prose-level "please don't" is probabilistic; tool-level restriction is deterministic. D is wrong because skills can express fine-grained Bash permissions via the same `Bash(pattern:*)` syntax, so there is no need to hack around it.

---

## Question 3

The Helios platform team is building a "release-notes" workflow that, for each merged PR, must: (1) read the PR diff, (2) read the linked Jira ticket, (3) read the last 5 git commits on the affected files, and (4) synthesize a release-notes entry. Each of (1)–(3) requires substantial reading (often >30k tokens of raw material per source), but only a 1-paragraph summary of each is needed for step (4).

Which orchestration design best balances context economy and reliability?

A) Run the entire workflow in a single Claude conversation, loading all raw material into context sequentially so the final synthesis step has full visibility into every detail it might need.

B) Use three subagents in parallel — one each for diff, Jira, and git history — each instructed to return only a 1-paragraph summary; the coordinator then synthesizes from the three summaries.

C) Use three subagents sequentially (diff → Jira → git), each appending its raw findings to a shared scratch file that the coordinator reads at the end to synthesize.

D) Skip subagents entirely; have the coordinator call all three read tools in parallel in one turn and synthesize from the raw outputs.

Correct Answer: B
Subagents run in isolated contexts — the coordinator never sees the 30k-token raw material, only the three condensed summaries. Parallel dispatch also cuts wall-clock latency. This is the canonical context-economy + parallelization pattern for "read a lot, synthesize a little." A is wrong because sequential loading into one context burns budget on raw material the synthesis step does not need, and risks "lost in the middle." C is wrong because a shared scratch file just relocates the bloat — the coordinator still loads the raw findings to synthesize, and it serializes work that could parallelize. D is wrong because parallel tool calls reduce round-trips but the raw outputs still land in the coordinator's context, same bloat as A.

---

## Question 4

Helios has an internal MCP server `helios-jira` that exposes `search_tickets`, `get_ticket`, and `add_comment` tools. The platform team wants every engineer working in the `payments-service` repo to have this MCP server auto-connected when they run Claude Code in that repo, but they do NOT want it auto-connected for engineers working in unrelated repos. The team also wants the configuration to be code-reviewed.

What is the correct way to configure this?

A) Add the server to each engineer's `~/.claude.json` under the user-scoped `mcpServers` key and instruct them to only enable it manually when in the payments repo.

B) Run `claude mcp add helios-jira ...` once on a shared bastion host so the server is globally registered for the org.

C) Commit a `.mcp.json` file at the root of the `payments-service` repo declaring the `helios-jira` server; Claude Code loads project-scoped MCP servers from this file when started in the repo.

D) Add the server config to `./.claude/settings.local.json`; this file is project-scoped and supports the same `mcpServers` schema.

Correct Answer: C
`.mcp.json` is Claude Code's project-scoped MCP configuration file. When `claude` starts in a directory containing `.mcp.json`, those servers are offered to the user (with a trust prompt on first run). Committing it makes the config code-reviewable and team-shared, and it only activates inside that repo. A is wrong because user-scope (`~/.claude.json`) follows the engineer into every repo — the opposite of "only in payments-service" — and is not code-reviewed. B is wrong because there is no global/bastion MCP registration mechanism; MCP servers are configured per user/project/local scope on each developer's machine. D is wrong because `settings.local.json` is gitignored by default (not code-reviewed), and MCP server configuration belongs in `.mcp.json`, not the settings file.

---

## Question 5

A Helios engineer is using the Claude API (not Claude Code) to build a script that classifies merged PR titles into one of four release-note categories (`feature`, `fix`, `chore`, `breaking`) and produces a short summary. The downstream consumer is a Slack bot that parses the response programmatically and will crash on any deviation from the expected shape.

Which approach gives the most reliable structured output?

A) Use a regular `messages.create` call and instruct Claude in the system prompt: "Respond ONLY with JSON in the form `{category, summary}`. Do not include any other text."

B) Use a `messages.create` call with a `tools` parameter defining a single `record_release_note` tool whose `input_schema` declares `category` (enum of the four values) and `summary` (string), and force its use with `tool_choice: {type: "tool", name: "record_release_note"}`.

C) Use the Message Batches API with a JSON schema attached to each request, since batches enforce schema validation at API level.

D) Set `response_format: {"type": "json_object"}` on the `messages.create` call; Claude's JSON mode validates the response against an inferred schema.

Correct Answer: B
Forcing a specific tool with a JSON-schema'd `input_schema` is the canonical Claude API pattern for guaranteed structured output. The model must produce an argument object matching the schema, and `tool_choice` removes the option of free-text replies. A is wrong because prose instructions are probabilistic — Claude may still add a preamble, trailing prose, or markdown fences. C is wrong because the Message Batches API is just an async wrapper over `messages.create` (50% cost, up to 24h latency, no SLA) and does not add schema validation. D is wrong because `response_format: {type: "json_object"}` is OpenAI's parameter, not Claude's — the Claude API does not have a JSON-mode toggle; structured output is achieved through tool use.

---

## Question 6

A Helios engineer has been pair-debugging a flaky test with Claude Code for ~90 minutes. The conversation now contains many tool results from earlier hypotheses that turned out to be dead ends (large file reads, irrelevant log dumps). The current line of investigation is fresh and unrelated to those dead ends, but the engineer wants to keep the findings so far (which hypothesis the team has now landed on, plus 2 reproducer commands) without paying token cost on the dead-end material.

Which Claude Code action best fits this need?

A) Run `/clear` to wipe the conversation entirely and start fresh — the engineer can re-state the current findings in their next prompt.

B) Run `/compact` with an instruction like "preserve the current hypothesis and the two reproducer commands; drop earlier dead-end investigations" — Claude summarizes the conversation according to the instruction.

C) Run `/resume` to fork the conversation at the point where the dead ends began so the rest is dropped.

D) Do nothing — Claude Code's auto-compaction kicks in once context fills, and it preserves the most recent half of the conversation regardless.

Correct Answer: B
`/compact` accepts an optional instruction that tells Claude what to preserve and what to drop when summarizing the conversation. This gives surgical control: keep the current hypothesis and the two reproducer commands, discard the dead-end material — without losing useful state. A is wrong because `/clear` wipes everything, including the findings the engineer wants to keep. C is wrong because `/resume` is for resuming a previously saved session, not for surgically forking out a middle portion of the current conversation. D is wrong because auto-compaction is generic and not instruction-aware — it cannot be told "keep these two specific things."

---

## Question 7

The Helios platform team wants to automatically run `golangci-lint run` on any `.go` file Claude Code modifies and surface lint failures back into the conversation so Claude can fix them in the same turn-set. The check must not block the *Edit/Write tool call itself*, but should run immediately after the file is changed.

Which hook configuration is the right fit?

A) A `PreToolUse` hook on `Edit|Write` matching `*.go` files; it runs before the edit and can block the edit if lint would fail.

B) A `Stop` hook on every assistant turn; it runs `golangci-lint` on the whole repo whenever Claude finishes a turn.

C) A `PostToolUse` hook on `Edit|Write` matching `*.go` files; it runs after the tool succeeds and can return output that Claude sees in the next turn.

D) A `UserPromptSubmit` hook that runs `golangci-lint` whenever the user submits a new prompt, ensuring lint state is fresh at the start of every turn.

Correct Answer: C
`PostToolUse` fires after a matching tool call succeeds and can emit feedback that Claude sees on the next turn. This matches the requirement exactly: don't block the edit, but lint immediately after and surface failures so Claude can fix them in-flight. A is wrong because `PreToolUse` runs before the edit — you can't lint a file that has not been written yet, and it would block the very tool call the team wants to allow. B is wrong because `Stop` fires at turn end on the whole repo, which is coarser, slower, and not scoped to the actual file Claude just touched. D is wrong because `UserPromptSubmit` fires when the user submits input, not when Claude modifies a file — wrong event entirely.

---

## Question 8

A Helios engineer is writing a custom agent loop using the Claude API (not the Agent SDK). The agent has 6 tools defined and runs in a `while True:` loop. The engineer is deciding when the loop should terminate.

Which termination condition is the canonical correct one for a tool-using agent?

A) Terminate when the assistant message contains no `tool_use` content blocks (regardless of `stop_reason`), since absence of a tool call signals the agent has finished.

B) Terminate when the API response's `stop_reason` is `"end_turn"` (or other terminal reasons like `"max_tokens"`, `"stop_sequence"`); continue the loop while `stop_reason` is `"tool_use"`.

C) Terminate after a fixed maximum of 5 iterations, regardless of `stop_reason`, since unbounded loops are unsafe in production.

D) Terminate when the user-supplied callback `is_done(messages)` returns true; the API does not provide a built-in signal for agentic completion.

Correct Answer: B
`stop_reason` is the authoritative API-level signal for agent-loop control. `"tool_use"` means "I need you to run these tools and feed back results"; `"end_turn"` means "I'm done." This is the canonical agentic loop pattern in the Claude API docs. A is wrong because checking content blocks instead of `stop_reason` is fragile — `stop_reason` is the explicit, documented signal. C is wrong because iteration caps are useful as a safety net (defense in depth), but they are not the canonical termination condition — they prematurely cut off legitimate multi-step work. D is wrong because the API does provide a built-in signal (`stop_reason`); a user-defined `is_done` callback is unnecessary boilerplate.

---

## Question 9

A Helios engineer is designing a new MCP tool `query_metrics` for the internal observability MCP server. The tool accepts a free-form PromQL query string and returns time-series data. In an early eval, Claude often called `query_metrics` with vague inputs (e.g., `query="latency"`) instead of valid PromQL, then complained when the server returned an error.

Which change to the tool's *definition* is most likely to improve correct usage, with no model retraining?

A) Make the tool's description longer and more conversational — describe the tool's history and the team that owns it so Claude treats it as authoritative.

B) Tighten the tool description and `input_schema`: specify that `query` MUST be a valid PromQL expression, give 2–3 inline examples of valid queries, and add a `description` on the `query` parameter explaining the syntax.

C) Remove the `query` parameter and let Claude pass arbitrary natural-language strings — the server can translate to PromQL itself, hiding the syntax from Claude.

D) Replace the tool with 30 fine-grained tools, one per common metric (e.g., `get_p99_latency`, `get_error_rate`), so Claude never has to construct PromQL.

Correct Answer: B
Tool descriptions and `input_schema` annotations are the primary mechanism Claude uses to decide how to call a tool. Making the contract explicit ("must be valid PromQL"), describing the parameter, and showing examples directly addresses the failure mode — wrong input shape — with no model changes. A is wrong because padding the description with team history and ownership is noise; descriptions should be functional. C is wrong because pushing NL→PromQL translation into the server hides the problem rather than solving it, and adds a brittle translation layer Claude can't see or reason about. D is wrong because tool sprawl (30 narrow tools) increases selection difficulty, balloons the tool listing in context, and loses the flexibility PromQL was meant to provide.

---

## Question 10

A Helios engineer is asking Claude Code to refactor a large, intertwined module that touches authentication, logging, and database access. The engineer is not yet sure which refactor approach is best and wants Claude to think through trade-offs and present a plan *before* making any edits.

Which Claude Code feature best fits this need?

A) Run `claude --dangerously-skip-permissions` so Claude can iterate without prompts, edit, and revert if the approach turns out to be wrong.

B) Use plan mode (`Shift+Tab` to toggle, or `claude --permission-mode plan`) — Claude analyzes and proposes a plan without making file changes until the user approves.

C) Use the `-p` headless flag so Claude returns a single text response describing the plan, then run a second `claude` invocation with the approved plan to perform the edits.

D) Run `/clear` and re-prompt with "Do not write code yet — first describe your plan." This is the only way to prevent Claude from editing immediately.

Correct Answer: B
Plan mode is purpose-built for "analyze and propose before editing." It blocks file-modifying tools until the user explicitly approves the plan, so Claude can think through trade-offs without accidentally committing to one approach. A is wrong because `--dangerously-skip-permissions` is the opposite stance — full autonomy with no gating. C is wrong because `-p` is for headless/CI scripts that need a single response — using it for interactive refactor planning loses the iterative back-and-forth. D is wrong because prose instructions ("don't write code yet") are probabilistic; plan mode is the deterministic, tool-level guarantee.

---

## Question 11

The Helios platform team wants a nightly cron job that runs Claude Code in CI to audit dependency licenses across all repos and produce a JSON report. The job must run unattended, must not prompt for anything, and the output must be machine-parseable.

Which invocation pattern is correct?

A) `claude --interactive "audit dependency licenses"` with a long timeout so the job has time to respond to permission prompts as they arise.

B) `claude -p "audit dependency licenses and emit a JSON report matching <schema>" --output-format json` running in a sandboxed environment with permissions pre-configured via `settings.json`.

C) `claude --headless` (no prompt argument) — `--headless` is the canonical flag for non-interactive runs, and Claude infers the task from the CI environment.

D) `claude --batch` to submit the run to the Message Batches API; CI then polls for completion and pulls the result.

Correct Answer: B
`-p` (`--print`) is Claude Code's non-interactive mode — single prompt in, single response out, no TTY required. `--output-format json` produces machine-parseable output. Permissions configured upfront in `settings.json` (allowlists, etc.) prevent any mid-run prompts that would deadlock an unattended CI job. A is wrong because `--interactive` keeps the TTY-based UX active; a cron job cannot respond to prompts. C is wrong because there is no `--headless` flag in Claude Code — the headless mode is `-p` / `--print`. D is wrong because `--batch` is not a Claude Code flag, and the Message Batches API is an API-side feature for the Messages API, not Claude Code.

---

## Question 12

A Helios engineer is iterating on a long Claude Code session refactoring an authentication module. They notice that the most recent ~10 turns produce surprisingly low-quality suggestions: Claude has started missing constraints that were clearly stated 50 turns ago and citing files it has not opened in this session.

Which is the most likely diagnosis, and the right corrective action?

A) Claude's model weights drift over the course of a long session; running `/model` to switch and switch back resets the weights.

B) The conversation has grown long enough that key earlier constraints are buried mid-context ("lost in the middle"); compact with an instruction to preserve those constraints, or re-state them explicitly in the next prompt.

C) Claude Code disables tool use after a certain number of turns; opening a fresh session is the only fix.

D) The agent has exhausted its tool-use quota for the session; wait for the per-hour quota to reset.

Correct Answer: B
LLMs (including Claude) reliably attend to information at the beginning and end of context, but attention degrades in the middle. After many turns, constraints stated 50 turns ago sit in that low-attention region. The remedies: `/compact "preserve constraints X, Y, Z"` to bring them forward as a summary, or simply re-state the constraints in the current prompt so they sit near the model's focus. A is wrong because model weights are immutable; `/model` switches which model you're talking to, not "resets" anything. C is wrong because there is no documented "tool use disabled after N turns" behavior in Claude Code. D is wrong because there is no per-session tool-use quota mechanic that degrades output quality.

---

## Question 13

The Helios platform team is building a "code-review-bot" that reviews PRs in CI. They want it to behave deterministically: for the same PR diff, the bot should always produce the same set of review comments (idempotent runs, repeatable in audit). Currently it uses one Claude agent with a single comprehensive system prompt; runs vary noticeably between invocations.

Which architectural change most directly addresses the variability, *without* abandoning Claude as the reasoning engine?

A) Switch from Claude to a deterministic rule-based linter; LLM-based review is fundamentally non-deterministic and cannot be stabilized.

B) Decompose the single agent into multiple specialized subagents (e.g., "security", "style", "tests"), each with a narrow, well-scoped prompt; orchestrate them via a coordinator that aggregates findings.

C) Increase `max_tokens` and `temperature` so the model has more room to think; more thinking yields more consistent outputs.

D) Run the same agent five times in parallel and majority-vote the comments; voting eliminates non-determinism in LLM outputs.

Correct Answer: B
A single "do everything" prompt gives the model many degrees of freedom — what to prioritize, what depth to go to, what to ignore — and each axis is a source of variance. Narrow, specialized subagents (security / style / tests, each with a tight scope) reduce ambiguity, which reduces output variance. A coordinator aggregates findings into a deterministic structure. A is wrong because it abandons Claude entirely, violating the constraint. C is wrong because increasing `temperature` increases randomness, not consistency, and `max_tokens` is unrelated to determinism. D is wrong because majority-voting over free-text comments is hard (the comments themselves vary in wording), costs 5× tokens, and at best reduces variance — it does not eliminate it.

---

## Question 14

A Helios engineer is configuring the `helios-jira` MCP server. The server requires an OAuth token to call the Jira API. The team wants:
- The token to never appear in `.mcp.json` (which is checked into the repo).
- Each engineer to use their own OAuth token (so Jira audit logs attribute actions to the right person).

What is the right pattern?

A) Hardcode a shared service-account token in `.mcp.json`; the audit-log requirement can be met later with a Jira proxy.

B) Reference an environment variable in `.mcp.json` (e.g., `"args": ["--token", "${JIRA_TOKEN}"]`); each engineer sets `JIRA_TOKEN` in their shell environment.

C) Store the token in a `secrets:` field that `.mcp.json` supports natively; Claude Code encrypts the value at rest.

D) Put the token in `~/.claude/CLAUDE.md` so it loads at session start and Claude Code injects it into the MCP server stdin.

Correct Answer: B
`.mcp.json` supports environment-variable expansion in `command`, `args`, and `env` fields via `${VAR}` syntax. This keeps the checked-in file secret-free, and because each engineer has their own `JIRA_TOKEN`, Jira's audit log attributes actions correctly. A is wrong because a shared service-account token violates the per-engineer audit-log requirement and commits a secret. C is wrong because there is no `secrets:` field in `.mcp.json` — that is not part of the schema. D is wrong because CLAUDE.md is a memory file whose contents are sent to the model on every turn — putting a secret there leaks it into prompts, and there is no "inject into MCP stdin" mechanism behind it.

---

## Question 15

A Helios engineer is writing a Claude API prompt that classifies internal support tickets into one of `bug | feature_request | question | spam`. The classifier currently makes ~15% mistakes on tickets that are *both* a bug report *and* contain a feature suggestion ("It crashes when X — can you add Y?"). The engineer must keep the output as a single category (downstream system can't accept multi-label), and cannot change the schema.

Which prompt-engineering change most directly reduces this specific failure mode?

A) Add a system-prompt rule that mixed tickets should be classified as `bug` (with a brief rationale: "bugs block users; feature suggestions can be filed separately"), and include 2–3 few-shot examples of mixed tickets correctly labeled `bug`.

B) Lower the temperature to 0 so Claude always returns the most-likely category for any input.

C) Ask Claude to "think carefully" in the system prompt; chain-of-thought reasoning always improves classification accuracy.

D) Switch from single-shot classification to a self-critique loop where Claude classifies, then a second pass critiques the classification and may revise it.

Correct Answer: A
The failure mode is precisely ambiguity — tickets that legitimately match two categories. The fix is to remove the ambiguity in the prompt: (1) state a tie-breaker rule for that exact situation, and (2) show 2–3 worked examples of mixed tickets labeled the right way. Few-shot examples targeting the ambiguous boundary are the canonical pattern for nudging classifier behavior on edge cases. B is wrong because `temperature=0` makes outputs consistent given a prompt, but does not tell Claude which label to pick when two genuinely apply — deterministic on a 50/50 input still misclassifies, just consistently. C is wrong because "think carefully" is vague and not reliably useful for simple classification; CoT can even hurt if it introduces irrelevant reasoning. D is wrong because self-critique in the same context suffers from confirmation bias (the same model defends its first answer), doubles latency, and still does not resolve the underlying "which one when both apply?" question.
