# Scenario 3: Code Generation with Claude Code

Scenario: **Lumen Health**, a 120-engineer health-tech company, is rolling out Claude Code to its product engineering teams. The codebase is a polyglot monorepo: TypeScript frontends, a Go service mesh, Python ML services, plus shared infrastructure. The platform team is building Claude Code into the daily code-authoring workflow — engineers use Claude Code interactively to write features, refactor modules, and onboard new hires. The team is configuring CLAUDE.md, skills, hooks, and rules to standardize how Claude Code generates code across stacks.

Part of a full-length CCAF Foundations mock exam (4 scenarios × 15 questions = 60). Pass threshold: 43/60 (72%).

---

## Question 1

Lumen's monorepo has the following structure:

```
lumen-monorepo/
├── CLAUDE.md              # repo conventions
├── frontend/
│   ├── CLAUDE.md          # TypeScript/React conventions
│   └── components/
│       └── PatientCard.tsx
├── services/
│   ├── CLAUDE.md          # Go service conventions
│   └── billing/
└── ml/
    ├── CLAUDE.md          # Python ML conventions
    └── inference.py
```

An engineer runs `claude` from the monorepo root and asks Claude to "implement a new patient card component." Claude reads `frontend/components/PatientCard.tsx` to understand the existing structure.

Which CLAUDE.md files are in Claude's context at the moment it's reading `PatientCard.tsx`?

A) Only `lumen-monorepo/CLAUDE.md` — Claude Code loads exactly one CLAUDE.md per session (the one at the working directory).

B) `lumen-monorepo/CLAUDE.md` (loaded at launch), plus `frontend/CLAUDE.md` (loaded on-demand when Claude reads a file under `frontend/`).

C) All four CLAUDE.md files in the repo — Claude Code loads every CLAUDE.md it can find recursively at launch.

D) `lumen-monorepo/CLAUDE.md` and `frontend/CLAUDE.md` and `services/CLAUDE.md` — Claude loads all subdirectory CLAUDE.md files at launch regardless of which the engineer touches.

Correct Answer: B
The documented behavior: parent direction from cwd is loaded in full at launch; subdirectories below cwd are loaded on-demand when Claude reads files in those directories. The root CLAUDE.md was loaded at session start; `frontend/CLAUDE.md` is injected the moment Claude reads `frontend/components/PatientCard.tsx`. `services/CLAUDE.md` and `ml/CLAUDE.md` stay dormant until Claude reads files under those areas. A is wrong because Claude Code loads multiple CLAUDE.md files. C is wrong because subdirectory CLAUDE.md files are not pre-loaded at launch. D is wrong for the same reason — subdirectories don't get blanket pre-loading.

Reference: docs/Claude Code/Getting started/Use Claude Code/Store instructions and memories.md:63, :141

---

## Question 2

Lumen's lead engineer is investigating a context-exhaustion problem. They have a long-running Claude Code session where:
- The repo's `CLAUDE.md` is 800 lines (a few teams added sections over time)
- 12 path-scoped rules under `.claude/rules/` have all activated (~150 lines total)
- ~40 file reads have happened
- The conversation has 60+ turns

The engineer wants to free context budget without losing the most valuable persistent context. The single highest-leverage action is:

A) Disable all `.claude/rules/` by deleting the directory; rules are a duplicate of CLAUDE.md.

B) Run `/compact` repeatedly during the session to keep the conversation short; CLAUDE.md size is irrelevant.

C) Switch to a model with a larger context window; the session was always going to exhaust eventually.

D) Move large reference sections out of `CLAUDE.md` into `.claude/rules/*.md` with `paths:` frontmatter so they only load when Claude touches matching files; this directly cuts the always-on token cost.

Correct Answer: D
The 800-line CLAUDE.md is the dominant cost — it loads on every launch and again after every `/compact`. Docs explicitly recommend keeping CLAUDE.md under 200 lines and offloading the rest into path-scoped rules that load only when matching files are touched. This converts always-on token cost into on-demand cost. A is wrong because `.claude/rules/` aren't duplicates of CLAUDE.md; they serve a complementary path-scoped role. C is wrong because larger context windows don't fix waste. B is wrong because `/compact` shrinks the conversation history but CLAUDE.md is re-injected after compact.

Reference: docs/Claude Code/Getting started/Use Claude Code/Store instructions and memories.md:81, :393

---

## Question 3

Lumen wants a `/deploy` slash command that runs the team's deploy workflow — pre-flight checks, version bumping, changelog update, then triggers the actual deploy script. The team wants the command to be available to every engineer who clones the repo.

What's the correct place for this?

A) `~/.claude/commands/deploy.md` on every engineer's machine — slash commands always live in user scope.

B) `.claude/commands/deploy.md` checked into the repo with the workflow body — project-scoped slash commands ship with the repo and become available to everyone who clones it.

C) Hardcode the workflow into the repo's `CLAUDE.md` under a section "## Deploy workflow" so Claude follows it whenever the engineer mentions deploy.

D) `.mcp.json` declaring a `deploy` MCP server — slash commands are an MCP feature.

Correct Answer: B
Project-scoped slash commands live in `.claude/commands/<name>.md`. When the file is checked into the repo, every engineer who clones it automatically gets the `/deploy` command. Subdirectories namespace the command (`.claude/commands/frontend/component.md` becomes `/frontend:component`). A is wrong because slash commands have both user scope and project scope; project is correct for shared-with-team. C is wrong because CLAUDE.md is for persistent context, not invocable commands. D is wrong because slash commands are not an MCP feature.

Reference: Official Exam Guide Sample Question 4, docs/Claude Code/Getting started/Getting started/Changelog.md (0.2.31)

---

## Question 4

Lumen's platform team wants Claude Code to **automatically run `gofmt -w` on any `.go` file Claude edits**, with no engineer intervention. The action must happen reliably regardless of the engineer's prompt phrasing or which tool Claude used to modify the file.

What is the right mechanism?

A) Add an instruction to the root `CLAUDE.md`: "After editing any `.go` file, always run `gofmt -w <file>`."

B) Create a Claude Code skill named `gofmt` that runs `gofmt -w` and tell engineers to invoke it after every Go edit.

C) Configure a `PostToolUse` hook matching `Edit|Write` on `*.go` files; the hook runs `gofmt -w <file>` deterministically after the tool succeeds.

D) Set a path-scoped rule in `.claude/rules/gofmt.md` with `paths: ["**/*.go"]` containing "always run gofmt -w after editing"; rules enforce behavior deterministically.

Correct Answer: C
`PostToolUse` fires after the matching tool succeeds and runs deterministically (not LLM-mediated), which is what "regardless of prompt phrasing" requires. A is wrong because CLAUDE.md instructions are probabilistic. B is wrong because skills require invocation (manual or auto-load); they don't fire automatically on a tool event. D is wrong because rules are context loaded into Claude's prompt, not enforcement mechanisms — Claude must still decide to run `gofmt` and may not.

Reference: docs/Claude Code/Build with Claude Code/Automation/Automate with hooks.md, docs/Claude Code/Reference/Reference/Hooks reference.md

---

## Question 5

Lumen wants to give Claude Code access to their internal **patient-records database** so engineers can ask things like "what's the schema for the `consents` table?" or "show me a sample row." The database has **read-only** access via a service account. Multiple engineers will use this from their laptops.

Which MCP setup is the cleanest fit?

A) Implement an MCP server `lumen-pr-db` exposing `describe_table(name)`, `sample_rows(table, limit)`, and `list_tables()` tools, configured per-repo via `.mcp.json` with the connection string referenced as `${LUMEN_PR_DB_URL}` so each engineer sets their own env var.

B) Have each engineer install a generic `mcp-server-postgres` and configure it manually with the production database URL hardcoded in their personal config.

C) Skip MCP and have engineers run `psql` via Claude Code's Bash tool with the raw connection string in CLAUDE.md.

D) Use a `WebFetch` allow rule pointing at the internal DB's HTTP admin UI; Claude Code can scrape it for schema info.

Correct Answer: A
Three good design choices stacked: tailored intent-aligned tools (read-only by design), project-scoped `.mcp.json` (auto-available only in the relevant repo and code-reviewed), and env-var-referenced credentials (`${LUMEN_PR_DB_URL}` keeps secrets out of the checked-in file). B is wrong because hardcoding the production DB URL in each engineer's config is unsafe and creates configuration drift. C is wrong because connection strings in CLAUDE.md leak secrets into every prompt and give Claude raw SQL access. D is wrong because web-scraping the admin UI is brittle and a misuse of WebFetch.

Reference: docs/Claude Code/Build with Claude Code/Tools and plugins/Model Context Protocol (MCP).md

---

## Question 6

A Lumen engineer is starting a non-trivial refactor: extracting the patient-consent logic from a monolithic service into a new microservice. They want Claude to think through service boundaries, data ownership, migration steps, and rollout strategy *before* touching any code.

Which Claude Code feature is purpose-built for this?

A) `/clear` to start a fresh session, then prompt "describe a plan, do not edit yet" — fresh sessions force Claude to plan first.

B) **Plan mode** (`Shift+Tab` to toggle, or `--permission-mode plan`) — Claude reads, analyzes, and proposes a plan without making any file modifications until the user approves.

C) `--dangerously-skip-permissions` so Claude can freely explore the codebase to design the right plan.

D) `claude -p` (headless mode) — single-shot prompts produce more focused planning than interactive sessions.

Correct Answer: B
Plan mode is the deterministic, tool-level guarantee: file-modifying tools are blocked until the user explicitly approves the plan. Claude can read, search, analyze freely, and produces a plan as output. A is wrong because `/clear` + prose instructions is probabilistic. C is wrong because `--dangerously-skip-permissions` is the opposite stance — full unattended action. D is wrong because `-p` is for headless/CI scripts and loses the interactive refinement that planning needs.

Reference: docs/Claude Code/Getting started/Use Claude Code/Plan mode.md

---

## Question 7

A Lumen engineer is using Claude Code to investigate a complex bug across the frontend, the billing service, and the ML inference service. Claude needs to read ~50 files across the three areas and produce a root-cause analysis. The engineer is concerned about context pollution in their main session.

What is the cleanest pattern?

A) Run three completely separate Claude Code sessions (one per area) and manually paste findings between them; manual integration is the most reliable.

B) Read all 50 files in the main session and trust Claude to handle the context size; modern models have long context windows.

C) Spawn three **subagents** (one per area: frontend, billing, ML), each tasked with reading and summarizing findings from its area; the main agent receives only the summaries and synthesizes the root-cause analysis from them.

D) Run `/compact` between each area so the main session forgets the previous area before starting the next; this isolates context naturally.

Correct Answer: C
Each subagent works in its own context window. The main agent never sees the raw 50 files — only the three condensed summaries it needs to synthesize. This is the canonical context-isolation + parallelism pattern. A is wrong because manual paste between sessions defeats the agentic loop, loses parallelism, and is slow + error-prone. B is wrong because 50 files is a lot and the "lost in the middle" effect degrades attention even when it fits. D is wrong because `/compact` summarizes lossy — if synthesis needs details from area 1 while reasoning about area 3, those details are gone.

Reference: docs/Claude Code/Build with Claude Code/Agents/Subagents.md

---

## Question 8

Lumen's auto-memory feature is enabled. Over weeks of use, Claude has built up notes in `~/.claude/projects/lumen-monorepo/memory/`. An engineer notices that auto memory has captured an outdated build command (the team migrated from `npm` to `pnpm` three weeks ago, but the memory still says `npm test`).

What is the appropriate corrective action?

A) Disable auto-memory entirely; once memory is wrong it cannot be trusted, and the engineer should manage all memory manually via CLAUDE.md.

B) Run `/memory` to open the memory directory, locate the stale note, and edit or delete it; Claude reads/writes memory files during sessions and will use the updated content. Adding a CLAUDE.md entry stating "Use `pnpm`, not `npm`" further reinforces the correction.

C) Run `/clear` to wipe the session; auto-memory will rebuild correct entries from scratch.

D) Auto-memory cannot be edited by hand; it's a derived artifact, and the only way to update it is to wait for Claude to overwrite it during a future session.

Correct Answer: B
Auto-memory files are plain markdown the user can edit or delete at any time. `/memory` provides browse-and-open UI within a session. A CLAUDE.md entry adds defense in depth — a project-owned signal that any future auto-memory drift can be checked against. A is wrong; disabling a feature because it once held a stale note is overcorrection. C is wrong; `/clear` wipes the conversation, not auto-memory files on disk. D is wrong; directly contradicted by docs — auto-memory IS plain markdown.

Reference: docs/Claude Code/Getting started/Use Claude Code/Store instructions and memories.md:354-362

---

## Question 9

A Lumen engineer is building a small Python script using the Claude Agent SDK to automate "given a Jira ticket ID, draft a PR description by reading the linked code." The agent uses 3 tools: `get_jira_ticket`, `read_file`, `git_diff`. The engineer is unsure how to structure the loop.

Which Agent-SDK-aligned approach is correct?

A) Write the entire workflow as a single deterministic Python script that calls the Claude API once per step (one for "fetch ticket," one for "read code," one for "write description"); avoid loops.

B) Use a fixed sequence of 3 API calls with `tool_choice` forcing each tool in order; don't let Claude decide the order.

C) Skip the SDK; the Claude API doesn't natively support multi-tool agents and requires manual orchestration that's better written from scratch.

D) Run an agentic loop: `client.messages.create(...)` → if `stop_reason == "tool_use"` execute the tools and append results → repeat until `stop_reason` is terminal. Let Claude decide which tools to call and when to stop. This is the standard agentic loop pattern.

Correct Answer: D
The canonical pattern: send messages with `tools=` and let Claude decide; if `stop_reason == "tool_use"`, execute the tools and append results as `role: "user"` with `tool_result` content; loop until `stop_reason` becomes terminal. Letting Claude decide adapts to the situation (one ticket might need `git_diff` twice, another might need none). A is wrong because a fixed pipeline isn't an agent — it's a script with LLM-shaped calls. B is wrong because forcing every tool in predetermined order strips Claude of decision-making. C is wrong because the Claude API natively supports tool-using agents.

Reference: docs/Claude API/Build/Agents and tools/Building agents with the Claude API.md

---

## Question 10

Lumen's engineers use Claude Code to generate unit tests. The team noticed Claude often *duplicates* existing test scenarios — e.g., if `Button.test.tsx` already tests "renders with default props," Claude's newly generated tests include another "renders with default props" case with slightly different wording.

Which intervention is most effective?

A) Add a system-prompt instruction "do not duplicate existing tests" and trust Claude to follow.

B) Lower the temperature to 0 so Claude generates the same tests every time and duplicates can be deduplicated.

C) Run a post-processing script that diff-matches test names and deletes duplicates.

D) Have Claude read the existing test file *before* generating new tests, and include explicit instruction: "First list the test scenarios already covered; then generate ONLY scenarios not already in the list." Providing the existing tests in context as a reference is what the exam guide calls out for test generation.

Correct Answer: D
The exam guide highlights this pattern: "Providing existing test files in context so test generation avoids suggesting duplicate scenarios already covered by the test suite." The explicit "list covered scenarios, then generate only new" structure makes deduplication a step Claude reasons through, not a vague hope. A is wrong because without seeing existing tests, Claude has nothing to compare against. B is wrong because `temperature=0` makes output consistent, not correct — duplicates would just be the same duplicates every run. C is wrong because name-based diff catches exact duplicates but the failure mode is semantic duplicates with "slightly different wording."

Reference: Official Exam Guide Domain 4 (test generation with existing test context)

---

## Question 11

Lumen's platform team has set a deny rule on `Bash(rm -rf:*)` in the monorepo's `.claude/settings.json` (project-scope). A senior engineer doing a cleanup task adds `"allow": ["Bash(rm -rf:*)"]` to their personal `~/.claude/settings.json` to skip the approval prompt for their local workflow. Despite the change, Claude Code still refuses the command.

Why is the user-scope `allow` not taking effect, and what is the correct way to override the project rule on this engineer's machine?

A) Settings precedence is **Managed > CLI args > Local > Project > User**. A project-scope rule overrides any user-scope rule for the same permission, so the engineer's `allow` in `~/.claude/settings.json` is ignored. To override locally, the engineer must place the `allow` in a higher-precedence scope: e.g., `.claude/settings.local.json` (gitignored, local-scope), or pass `--settings <permissive-config>` on the command line.

B) `deny` rules are immutable in Claude Code once defined in any scope; no scope can override a `deny`. The engineer must ask the platform team to remove the project-level deny.

C) The user's `allow` is being silently dropped because user-scope settings cannot reference `Bash(...)` patterns — only managed and project scopes can. The engineer should move the rule to `.claude/settings.local.json`, where Bash patterns are supported.

D) The settings cascade requires `--permission-mode acceptEdits` to merge user-scope rules into the active session. Without that flag, only project rules are loaded.

Correct Answer: A
The documented precedence is Managed > CLI args > Local > Project > User. The official doc spells this exact conflict out: "if a permission is allowed in user settings but denied in project settings, the project setting takes precedence and the permission is blocked." To get the engineer's allow to win, it has to come from a scope higher than Project: `.claude/settings.local.json` (Local) or `--settings <file>` (CLI). B is wrong because `deny` is not immutable — a higher-precedence scope can override. C is wrong because Bash patterns work in all scopes, including user. D is wrong because `--permission-mode acceptEdits` controls prompt behavior, not settings cascade.

Reference: docs/Claude Code/ Configuration/Settings and permissions/Settings.md:52-60

---

## Question 12

A Lumen engineer is writing a small Claude API script that summarizes a meeting transcript. The transcript is ~30,000 tokens. The script doesn't use any tools — just a single `messages.create` call.

The engineer is debugging why the summary occasionally misses topics that appear in the **middle** of the transcript. The script runs the same transcript through Claude 3 times and gets 3 different summaries.

Which two factors are the most plausible explanations?

A) **(i)** "Lost in the middle" attention bias — middle content gets less attention than start/end content. **(ii)** Default sampling temperature (>0) introduces variance across runs.

B) The Claude API silently drops content after the first 10k tokens for non-streaming requests; the script should stream.

C) Anthropic's API caches summaries for repeat inputs; the script is getting different cached entries.

D) The Claude API requires `tool_choice` to be set even when no tools are used; without it, the model produces non-deterministic outputs.

Correct Answer: A
Two independent failure modes stacked: systematic miss of middle content → attention bias; cross-run variance → temperature-driven sampling (default temperature is 1.0). The fixes are also independent: structural mitigation for position effects (section headers, summarize-per-chunk) and `temperature: 0` (or low) for reproducibility. B is fabricated. C is fabricated — prompt caching is opt-in via `cache_control` and doesn't return different cached entries. D is fabricated.

Reference: Official Exam Guide Domain 5 (lost in the middle), docs/Claude API/API Reference/Messages/Create a Message.md (default temperature 1.0)

---

## Question 13

Lumen has an internal `lumen-payments` MCP server with tools `list_invoices`, `get_invoice`, `process_refund`. The `process_refund` tool is **destructive** — once called, the refund is irreversible. The team is worried Claude will call it speculatively (e.g., "let me try refunding to see if the customer is eligible").

Which combination of tool-level + Claude Code-level controls best mitigates this risk?

A) Tighten the `process_refund` tool's description to "DO NOT call this speculatively" and trust Claude to follow.

B) Remove `process_refund` from the MCP server entirely; refunds must be performed in the admin UI by humans.

C) Keep `process_refund` in the MCP server, but: (i) write its tool description to emphasize irreversibility and require an explicit `confirmation_token` argument the user must supply; (ii) set `permissions.ask: ["mcp__lumen-payments__process_refund"]` in `.claude/settings.json` so every call requires explicit user approval. Defense in depth: prompt + permission gate + parameter contract.

D) Use a `PreToolUse` hook that always denies `process_refund` regardless of context; if a refund is genuinely needed, the engineer must edit the hook.

Correct Answer: C
Three independent layers catching different failure modes: tool description (soft signal making Claude less likely to call speculatively), `confirmation_token` parameter (structural enforcement — Claude can't decide on its own), and `permissions.ask` gate (every invocation surfaces a UI approval; human is the final gate). Even if one layer fails, the next catches it. A is wrong because prose alone is probabilistic. B is overreach — the operation is legitimate; the issue is control, not removal. D is wrong because always-deny + hook-editing is a bypassable safety net that trains engineers to disable safety gates locally.

Reference: docs/Claude API/Build/Tools/, docs/Claude Code/ Configuration/Settings and permissions/Permissions.md, exam guide Domain 2

---

## Question 14

A Lumen engineer is using the Claude API directly (no Claude Code) to extract structured fields from clinician notes. The notes are messy free-text, and the output must conform exactly to:

```
{
  "patient_id": "string",
  "icd10_codes": ["string", ...],
  "medications": [{"name": "string", "dosage": "string"}],
  "follow_up_date": "YYYY-MM-DD or null"
}
```

The engineer wrote a system prompt: "Extract these fields and respond as JSON." Output is mostly correct but occasionally Claude adds a preamble ("Here's the extracted data:") or wraps the JSON in a code fence.

What's the most reliable fix?

A) Define a tool `record_extraction` with `input_schema` matching the exact shape (including `icd10_codes` as an array of strings, `medications` as an array of objects, `follow_up_date` as nullable string), and force `tool_choice: {type: "tool", name: "record_extraction"}`. The tool's `input` argument will conform to the schema with no preamble or fence.

B) Add to the system prompt: "Respond with raw JSON only. Do NOT include any preamble, markdown, or code fences." Re-instructing is the standard pattern.

C) Run the extraction twice and keep only output that successfully `json.loads`. Retry-on-parse is cheaper than schema enforcement.

D) Use `response_format: {"type": "json_object"}` on the Messages API; Claude's JSON mode strips preambles and fences automatically.

Correct Answer: A
Tool input is structurally guaranteed to match the `input_schema` — there's no opportunity for preamble or code fence because the output isn't free text. With `tool_choice: {type: "tool", name: ...}` the model must produce a conforming argument object. B is wrong because re-instructing is probabilistic. C is wrong because retry-on-parse pays for the same work twice. D is wrong because `response_format: {type: "json_object"}` is OpenAI, not Claude — the Claude API doesn't have a JSON-mode toggle.

Reference: docs/Claude API/Build/Tools/, docs/Claude API/Build/Structured outputs.md

---

## Question 15

A Lumen engineer has been working with Claude Code on a refactor for ~3 hours. They notice that recent Claude responses sometimes contradict decisions made earlier in the conversation (e.g., Claude reverts to using a class name they explicitly renamed 90 turns ago). The engineer doesn't want to lose the conversation entirely.

What is the most appropriate action?

A) Restart the session (`claude` exit + re-launch) and re-explain everything from scratch.

B) Run `/clear` to wipe history; the rename was made in conversation so it can be re-stated.

C) Run `/compact` with an explicit instruction: "preserve the renaming decisions and the final agreed-upon naming convention." `/compact` summarizes the conversation according to the instruction, freeing context budget while keeping load-bearing decisions, and re-injects CLAUDE.md from disk. The engineer can also add the convention to `CLAUDE.md` for persistence across future sessions.

D) Tell Claude to "remember the renames from earlier" — instruction repetition is the documented remedy for state forgetting.

Correct Answer: C
`/compact <instruction>` is the surgical tool for "the conversation has drifted; preserve these specific decisions and drop the rest." It frees context budget while keeping load-bearing state, and re-injects CLAUDE.md from disk after compaction. Persisting the convention to CLAUDE.md ensures it survives across future sessions. A is wrong because restart loses the in-flight refactor state. B is wrong because `/clear` wipes everything including the renames the engineer wants to keep. D is wrong because prompt-level reminders are probabilistic and don't address the structural cause (the rename decision sits in a low-attention region of a long context).

Reference: docs/Claude Code/Getting started/Use Claude Code/Store instructions and memories.md:397, exam guide Domain 5
