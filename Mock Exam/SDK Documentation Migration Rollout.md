# SDK Documentation Migration Rollout

Scenario: **Northstar SDKs**, a developer-platform company, is migrating 1,200 pages of SDK documentation from a legacy docs stack to a new docs-as-code repository. The repository contains API reference pages, tutorials, generated examples, migration guides, and language-specific snippets for TypeScript, Python, Go, and Java. The docs platform team is rolling out Claude Code to technical writers and SDK engineers so they can safely rewrite pages, normalize examples, run link checks, and produce migration summaries.

This scenario is **Domain 3-focused** because prior scenarios already covered agent orchestration, MCP tool design, structured extraction, privacy operations, and field-maintenance workflows. It targets Claude Code configuration and workflows: CLAUDE.md hierarchy, path-scoped rules, skills, hooks, permissions, plan mode, iterative refinement, checkpointing, and CI/headless usage. Difficulty: hard. Standalone CCAF Foundations mock scenario (15 questions). Pass threshold: 11/15 (72%).

Domain mix:
- Domain 3: Claude Code Configuration & Workflows — 11
- Domain 2: Tool Design & MCP Integration — 1
- Domain 4: Prompt Engineering & Structured Output — 2
- Domain 5: Context Management & Reliability — 1

---

## Question 1

Northstar already maintains an `AGENTS.md` file for several coding agents. The docs platform team wants Claude Code to use those shared instructions too, while adding a few Claude-specific notes about the docs build command and preferred review workflow.

What is the best configuration?

A) Rename `AGENTS.md` to `CLAUDE.md`; other coding agents should be updated to read the new filename.

B) Create `CLAUDE.md` that imports `@AGENTS.md`, then add Claude-specific notes below the import.

C) Put the contents of `AGENTS.md` into `~/.claude/CLAUDE.md` for each writer so it loads in every repository.

D) Put a pointer to `AGENTS.md` in `.claude/settings.json`; settings files are where Claude reads prose instructions.

Correct Answer: B

Claude Code reads `CLAUDE.md`, not `AGENTS.md`. The docs recommend creating a `CLAUDE.md` that imports `@AGENTS.md` when a repo already uses AGENTS.md, then appending Claude-specific instructions. User-level memory would not be shared via version control, and settings files are enforced configuration, not prose memory.

Reference: docs/Claude Code/Getting started/Use Claude Code/Store instructions and memories.md

---

## Question 2

The repo has docs under `docs/tutorials/**`, generated API references under `docs/reference/**`, and runnable examples under `examples/**`. The team currently has a 900-line root `CLAUDE.md` containing rules for all three areas. Claude often spends context on irrelevant instructions.

Which organization best reduces always-on context while preserving the right rules?

A) Move area-specific rules into `.claude/rules/*.md` files with `paths:` frontmatter matching each area.

B) Keep the 900-line root `CLAUDE.md`; longer instructions improve adherence for large docs migrations.

C) Move all rules into `CLAUDE.local.md` so each writer can customize them.

D) Put every rule into a single project skill and rely on Claude to invoke it before every file edit.

Correct Answer: A

Path-scoped rules load only when Claude works with matching files, reducing irrelevant context and token usage. A huge root `CLAUDE.md` is always loaded; `CLAUDE.local.md` is personal and not team-shared; a skill is task-triggered, not the right mechanism for conditional file-type conventions that should activate by path.

Reference: docs/Claude Code/Getting started/Use Claude Code/Store instructions and memories.md; Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 3

The team has a repeatable "convert legacy tutorial" workflow: read a legacy Markdown file, map old frontmatter to new frontmatter, update code fences, run a local validator, and report unresolved questions. The workflow is long and only needed when explicitly converting a tutorial.

Where should this live?

A) In root `CLAUDE.md`, because every session should preload the full conversion playbook.

B) In `.claude/settings.json`, because settings are the correct place for workflow instructions.

C) In a project skill such as `.claude/skills/convert-legacy-tutorial/SKILL.md`, optionally with `argument-hint`, `allowed-tools`, and supporting files.

D) In each writer's `~/.claude/skills/` directory, so each person can evolve a different version.

Correct Answer: C

Skills are designed for reusable task playbooks that should load only when invoked or relevant. Project scope shares the workflow through version control. Root `CLAUDE.md` would impose the playbook on every session, and user-scope skills would drift across writers.

Reference: docs/Claude Code/Build with Claude Code/Tools and plugins/Extend Claude with skills.md; Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 4

The `convert-legacy-tutorial` skill generates verbose mapping notes while reading old pages. The team wants that verbose exploration isolated from the main conversation, but the final summary should return to the main session. The skill should be able to read files and run `npm run docs:validate <path>`, but not edit unrelated files.

Which frontmatter direction best fits?

A) Use `context: fork` and restrict `allowed-tools` to the needed read and validation tools.

B) Use `disable-model-invocation: true` only; that automatically runs the skill in a separate context with limited tools.

C) Put "do not pollute the main conversation" in the skill body; skills cannot isolate context.

D) Use `user-invocable: false`; hidden skills run in a sandboxed subagent by default.

Correct Answer: A

`context: fork` runs a skill in a forked subagent context, which keeps verbose work out of the main conversation. `allowed-tools` narrows the tool surface while the skill runs. Invocation-control fields do not imply context isolation or tool restriction.

Reference: docs/Claude Code/Build with Claude Code/Tools and plugins/Extend Claude with skills.md; Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 5

A writer asks Claude Code to reorganize the entire information architecture for the Python SDK docs: split tutorials, change navigation hierarchy, move examples, and update cross-links. There are multiple plausible approaches, and mistakes would create broad link breakage.

Which Claude Code workflow is most appropriate first?

A) Direct execution; ask Claude to start moving files and fix broken links as it discovers them.

B) Plan mode; have Claude inspect the docs structure and propose a plan before any file modifications.

C) `bypassPermissions`; broad docs moves require fewer prompts to be efficient.

D) `/clear`; a fresh session prevents Claude from making edits until the user asks again.

Correct Answer: B

Plan mode is intended for complex tasks with architectural implications, multiple valid approaches, and multi-file changes. It lets Claude explore and propose before modifying files. Direct execution is better for small scoped edits; bypassing permissions is not a planning mechanism; `/clear` only clears conversation state.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md; docs/Claude Code/Reference/Reference/CLI reference.md

---

## Question 6

Claude keeps converting legacy frontmatter inconsistently. For example, `product_area: "payments"` should become `tags: ["payments"]`, `beta: true` should become `availability: "beta"`, and missing `owner` should produce a review note instead of guessing.

What is the strongest iterative-refinement input?

A) Tell Claude "be consistent and do not hallucinate missing fields."

B) Lower the model temperature and retry the same natural-language instruction.

C) Provide 2-3 concrete input/output examples covering the frontmatter transformations and edge cases, then iterate using validator failures.

D) Move the transformation rules into a longer paragraph in `CLAUDE.md`.

Correct Answer: C

Concrete input/output examples are the strongest way to communicate expected transformations when prose is interpreted inconsistently. Validator failures give specific feedback for progressive improvement. Vague consistency instructions and lower temperature do not teach the exact mapping.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md; docs/Claude Code/Getting started/Use Claude Code/Common workflows.md

---

## Question 7

The team wants every Markdown file edited by Claude Code to be formatted by `prettier --write` immediately after the edit succeeds. The formatting must run regardless of whether Claude remembers the instruction.

Which mechanism is appropriate?

A) A `PostToolUse` hook matching `Edit|Write` for Markdown files.

B) A root `CLAUDE.md` rule saying "always run Prettier after edits."

C) A project skill named `/format-md` that writers manually invoke when they remember.

D) A `UserPromptSubmit` hook, because formatting should happen before Claude starts working.

Correct Answer: A

Hooks provide deterministic behavior at lifecycle events. `PostToolUse` runs after an edit/write succeeds, which is exactly when formatting can operate on the changed file. CLAUDE.md and skills are model-mediated or manually invoked, not guaranteed event enforcement.

Reference: docs/Claude Code/Build with Claude Code/Automation/Automate with hooks.md

---

## Question 8

Northstar's docs repo includes generated files under `docs/reference/generated/**`. Writers want Claude Code to read these files but never edit them. They also want a hard block even if Claude is in an otherwise permissive workflow.

Which configuration best expresses this?

A) Add a prose warning in `docs/reference/CLAUDE.md` that generated files are read-only.

B) Add `Edit(/docs/reference/generated/**)` to `permissions.allow` so Claude recognizes the path as special.

C) Add `Edit(/docs/reference/generated/**)` to `permissions.deny` in project settings.

D) Put the generated directory in `.gitignore`; Claude Code will infer not to edit ignored files.

Correct Answer: C

Permission deny rules enforce tool access and take precedence over ask/allow rules. A prose warning is not deterministic; allow is the opposite of a block; `.gitignore` is not a Claude Code edit policy.

Reference: docs/Claude Code/ Configuration/Settings and permissions/Permissions.md

---

## Question 9

A writer asks Claude to use `devbox run npm test`. The team previously allowed `Bash(devbox run *)` because they wanted test commands to run without prompts. Security review flags this as dangerous.

Why is the rule risky, and what is the safer pattern?

A) `devbox` is stripped as a recognized wrapper, so the rule will never match; use `Bash(npm test *)` instead.

B) Bash wildcard rules cannot match commands with spaces; use `Bash` to allow all Bash and rely on prompts.

C) The risk is only cosmetic; Claude Code separately validates the inner command inside all environment runners.

D) Environment runners are not stripped like simple wrappers, so `Bash(devbox run *)` can cover arbitrary inner commands; allow specific runner-plus-inner-command patterns such as `Bash(devbox run npm test)`.

Correct Answer: D

The permissions docs warn that development environment runners such as `devbox run` are not stripped. A broad rule like `Bash(devbox run *)` can authorize dangerous inner commands. The safer approach is specific allow rules for each intended inner command.

Reference: docs/Claude Code/ Configuration/Settings and permissions/Permissions.md

---

## Question 10

The docs team wants a nightly CI job to run Claude Code over changed docs and emit a machine-parseable report with fields `file`, `issue_type`, `severity`, and `suggested_fix`. The job must not wait for interactive input.

Which invocation pattern is best?

A) `claude "review changed docs"` in CI; interactive mode can infer it is running unattended.

B) `claude -p "review changed docs" --output-format json --json-schema '<schema>'` with permissions/settings preconfigured for CI.

C) `claude --batch "review changed docs"` because Claude Code has a native CI batch mode.

D) `claude -p "review changed docs"` and ask in the prompt for valid JSON; no CLI support exists for schema validation.

Correct Answer: B

The CLI reference documents `-p`/`--print` for non-interactive runs, `--output-format json`, and `--json-schema` for validated JSON output in print mode. Interactive sessions can hang in CI; prompt-only JSON lacks schema enforcement.

Reference: docs/Claude Code/Reference/Reference/CLI reference.md; Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 11

During a large docs migration, Claude makes a series of poor edits across 12 files. The writer wants to undo the file changes but keep the conversation so they can ask Claude what went wrong and try a different approach.

Which checkpointing action fits?

A) `/clear`, because it reverts file edits and clears the conversation.

B) `/compact`, because compaction rewrites files back to the prior state while preserving a summary.

C) `claude --resume <session>`; resuming a session automatically rolls back code.

D) `/rewind` then choose "Restore code" at the checkpoint before the bad edits.

Correct Answer: D

Checkpointing supports restoring code while keeping the conversation. `/clear` does not restore code; `/compact` manages context, not file state; resuming a session does not automatically undo changes.

Reference: docs/Claude Code/Reference/Reference/Checkpointing.md

---

## Question 12

The team has a shared style-guide repository outside the docs repo. A writer starts Claude Code in the docs repo with `--add-dir ../style-guide` so Claude can read the style guide. They expect `../style-guide/CLAUDE.md` and its `.claude/rules/` to load automatically, but Claude does not follow those instructions.

What explains this behavior?

A) `--add-dir` only works for MCP servers, not filesystem access.

B) Claude Code never reads files outside the current working directory, even with `--add-dir`.

C) Added directories override the main repo's CLAUDE.md, so the issue must be conflicting instructions.

D) `--add-dir` grants file access, but most `.claude/` configuration and CLAUDE.md files from added directories are not loaded by default.

Correct Answer: D

`--add-dir` grants access to additional directories, but most configuration discovery does not happen from those directories by default. The docs note exceptions and an environment variable for loading CLAUDE.md from additional directories, but access is not the same as configuration loading.

Reference: docs/Claude Code/Reference/Reference/CLI reference.md; docs/Claude Code/Getting started/Use Claude Code/Store instructions and memories.md

---

## Question 13

The docs team wants Claude Code to access a docs issue tracker through an MCP server only in the docs repository, with credentials coming from each writer's environment. The configuration should be code-reviewed and shared with the team.

Which setup is correct?

A) Commit `.mcp.json` at the docs repo root, using environment variable expansion such as `${DOCS_ISSUE_TOKEN}` for credentials.

B) Put the MCP server in every writer's `~/.claude.json`, including the shared token value.

C) Put the MCP server in `.claude/settings.local.json`; local settings are committed and reviewed by default.

D) Add the issue tracker URL to `CLAUDE.md`; Claude Code converts URLs in memory files into MCP tools automatically.

Correct Answer: A

Project-scoped `.mcp.json` is the right place for shared MCP server configuration, and environment variable expansion keeps secrets out of the repo. User config drifts and is not code-reviewed; local settings are personal/gitignored; CLAUDE.md does not create MCP tools.

Reference: docs/Claude Code/Getting started/Core concepts/Explore the .claude directory.md; Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 14

A writer asks Claude to update one broken link in a single Markdown page. The target URL and replacement text are already known, and only one file needs a small edit.

Which workflow is most appropriate?

A) Plan mode, because any documentation change should be planned before editing.

B) Direct execution, because the change is well-scoped and low-risk.

C) Create a new project skill first, because every repeated docs operation should be a skill before it is performed once.

D) Spawn an Explore subagent to inspect the full docs tree before making the one-line change.

Correct Answer: B

Direct execution is appropriate for a simple, well-scoped edit with a clear desired change. Plan mode and broad exploration are useful for architectural or multi-file uncertainty, but they add overhead here.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md; docs/Claude Code/Getting started/Use Claude Code/Common workflows.md

---

## Question 15

After migration, the team evaluates Claude-generated docs reviews. Writers dismiss many findings labeled "style issue" because they are subjective, but they rarely dismiss broken-link, invalid-code-fence, or missing-frontmatter findings.

Which prompt/eval improvement is strongest?

A) Tell Claude to be more confident before reporting style issues.

B) Lower temperature so the same style findings recur consistently across runs.

C) Temporarily disable or tighten the high-false-positive style category, add explicit criteria and examples, and track dismissal patterns by `detected_pattern`.

D) Remove structured fields from findings so writers can interpret comments more flexibly.

Correct Answer: C

The exam guide emphasizes explicit criteria over vague confidence instructions and tracking false-positive patterns with structured fields. A high false-positive category undermines trust, so tightening or temporarily disabling it while improving criteria is the strongest intervention.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md
