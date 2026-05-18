# Clinical Trial Operations Agent

Scenario: **Veridian CRO** is building a Claude-powered clinical-trial operations assistant called **TrialBridge**. Study managers use TrialBridge to check site-startup readiness, reconcile protocol amendments against eTMF documents, draft investigator-site follow-up actions, query CTMS/EDC records, and prepare audit-ready summaries. TrialBridge connects to clinical systems through MCP servers, uses Claude Code and the Agent SDK for internal engineering workflows, and must handle regulated data, long-running document checks, and human approvals for operational actions.

This scenario intentionally avoids customer support, generic CI review, legal contract extraction, industrial field maintenance, privacy audit, and SDK documentation migration workflows. It is hard and focuses on prior weak areas: MCP Roots vs Resources, Elicitation URL/Form modes, cancellation/progress, Claude Code permissions/hooks/skills/settings, citations vs structured outputs, prompt caching, and reliable agent-loop behavior. Standalone CCAF Foundations mock scenario (15 questions). Difficulty: hard. Pass threshold: 11/15 (72%).

Domain mix: Domain 1 = 3, Domain 2 = 4, Domain 3 = 4, Domain 4 = 3, Domain 5 = 1.

Correct-answer distribution: A = 4, B = 4, C = 4, D = 3.

---

## Question 1

TrialBridge must review a protocol amendment, compare affected procedures against site manuals, check missing training attestations in CTMS, and draft site-specific follow-up tasks. The inputs are large and spread across separate systems.

Which architecture is most appropriate?

A) Load every protocol, manual, and CTMS export into one long prompt and ask for a final site-readiness decision.

B) Use a coordinator that delegates scoped document, CTMS, and training checks to specialist agents, then synthesizes compact findings with provenance and human-review gates.

C) Let each MCP server independently call Claude and write directly to the final audit summary.

D) Convert all source systems into a single SQL query interface and disable multi-step agent orchestration.

Correct Answer: B

This is a strong coordinator-specialist case: the coordinator owns synthesis and review, while specialists operate on narrow contexts and return concise findings. A single monolithic context increases attention risk, and letting tools or servers write the final answer bypasses orchestration and review control.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md

---

## Question 2

TrialBridge has two separate MCP integrations. One eTMF server should expose protocol PDFs and site-startup checklists as selectable context. A local validation server should only operate within workspace folders explicitly selected by the client.

Which MCP concepts match those needs?

A) Roots expose eTMF documents as selectable context; Resources authorize local filesystem access.

B) Resources expose server-provided context; Roots define client-provided filesystem boundaries.

C) Elicitation exposes documents; Progress defines local filesystem access.

D) Tool annotations expose documents; Sampling defines local filesystem boundaries.

Correct Answer: B

Resources are server-exposed context identified by URIs and read through `resources/read`. Roots are client-exposed filesystem boundaries, using `file://` URIs, that servers can request and respect.

Reference: docs/Model Context Protocol/Specification/Server Features/Resources.md; docs/Model Context Protocol/Specification/Client Features/Roots.md

---

## Question 3

TrialBridge must produce an audit summary with precise citations to protocol PDFs. Downstream systems also require a strict JSON object with fixed fields: `site_id`, `missing_items`, `risk_level`, and `citations`.

What is the most accurate design?

A) Split the workflow or choose one output contract, because native citations cannot be used in the same request as `output_config.format`.

B) Enable citations and structured outputs together; citations are automatically converted into JSON arrays.

C) Use prompt-only page-number instructions because native citations are less reliable than generated citation text.

D) Put the PDFs in the system prompt and skip citations so the JSON schema can remain strict.

Correct Answer: A

Claude API native citations are incompatible with Structured Outputs in the same request. If both audit evidence and strict JSON are required, use a staged workflow or make an explicit tradeoff for the consumer that matters most.

Reference: docs/Claude API/Build/Model capabilities/Citations.md; docs/Claude API/Build/Model capabilities/Structured outputs.md

---

## Question 4

The CTMS MCP server needs the user to authorize access to an external EDC vendor. The authorization requires an OAuth consent screen and must not expose third-party tokens or credentials to the MCP client.

Which approach matches MCP elicitation guidance?

A) Use URL mode elicitation to send the user to a server-controlled authorization flow, bind the result to the verified user, and keep third-party credentials on the MCP server.

B) Use form mode elicitation and ask the user to paste the OAuth authorization code into the MCP client.

C) Reuse the MCP client's bearer token as the EDC token so no second authorization flow is needed.

D) Ask Claude to infer whether the user has EDC access from prior chat history and retry the tool call.

Correct Answer: A

URL mode is for sensitive or third-party authorization flows that must not pass through the MCP client. The MCP server must maintain state, bind the authorization to the verified user, and must not use URL elicitation as a substitute for MCP client-to-server authorization.

Reference: docs/Model Context Protocol/Specification/Client Features/Elicitation.md

---

## Question 5

The engineering team adds this project setting:

```json
{
  "permissions": {
    "allow": ["Bash(cat *)"],
    "deny": ["Read(./.env)", "Read(./secrets/**)"]
  }
}
```

They believe this prevents Claude Code from reading secrets while still allowing harmless shell inspection.

What is the main issue?

A) Deny rules are evaluated after allow rules, so `Bash(cat *)` overrides the read denies.

B) `Read(./.env)` is invalid because relative paths are never supported in permission rules.

C) `Bash(cat *)` is ignored because Bash commands cannot be granted with wildcards.

D) Read deny rules block built-in read tools but do not stop Bash subprocesses from reading the same files.

Correct Answer: D

Claude Code docs warn that Read/Edit deny rules apply to built-in file tools, not Bash subprocesses. If Bash can run `cat`, a separate Bash restriction or OS-level sandboxing is needed to prevent shell-based reads.

Reference: docs/Claude Code/ Configuration/Settings and permissions/Permissions.md

---

## Question 6

TrialBridge uses a Claude Code hook to block writes from a sensitive MCP server. The team wants the hook to run before every tool call from the configured `clinical` MCP server.

Which matcher pattern is correct?

A) `clinical`, because MCP tool names are matched by server name only.

B) `mcp__clinical`, because MCP server prefixes match all tools automatically.

C) `mcp__clinical__.*`, because MCP tool names use the `mcp__<server>__<tool>` pattern and regex matching is needed for all server tools.

D) `MCP(clinical *)`, because hook matchers use the same syntax as `.mcp.json`.

Correct Answer: C

Claude Code exposes MCP tools in hook events as regular tool names like `mcp__server__tool`. To match every tool from a server, use a regex-style matcher such as `mcp__clinical__.*`; `mcp__clinical` is treated as an exact string and matches no concrete tool.

Reference: docs/Claude Code/Reference/Reference/Hooks reference.md

---

## Question 7

An MCP request `reconcile_site_documents(site_id)` may run for several minutes. A study manager cancels the operation from the UI. The request is a normal MCP request, not a task-augmented request.

What should happen?

A) The client should send `tasks/cancel`, and the server must return a final task state.

B) The server should continue processing and send a normal response because MCP cancellation is only a UI hint.

C) The client should send `notifications/cancelled` with the in-progress request ID; the server should stop if possible, free resources, and avoid sending a normal response.

D) The client should send `notifications/progress` with `progress: 0` to tell the server to rewind the request.

Correct Answer: C

For normal in-progress MCP requests, cancellation uses `notifications/cancelled` with the original request ID and optional reason. `tasks/cancel` is reserved for task-augmented requests.

Reference: docs/Model Context Protocol/Specification/Base Protocol/Utilities/Cancellation.md

---

## Question 8

The same document reconciliation operation streams progress to the TrialBridge UI. Multiple reconciliations may be active at once for different sites.

What must the request include so progress can be correlated correctly?

A) A unique `progressToken` in request `_meta`, unique across active requests.

B) A `progressId` field in the user prompt so Claude can repeat it in status text.

C) A `resources/subscribe` call before each `notifications/progress` event.

D) A new JSON-RPC request ID for every progress update.

Correct Answer: A

MCP progress updates are requested by including `_meta.progressToken` on the original request. Progress notifications reference that token, and the token must be unique across active requests.

Reference: docs/Model Context Protocol/Specification/Base Protocol/Utilities/Progress.md

---

## Question 9

Veridian wraps the Claude Agent SDK in a multi-tenant internal service. A developer sets `settingSources: []` and assumes no host-level Claude Code configuration or memory can affect tenant sessions.

What is the safest correction?

A) Run each tenant in isolated filesystem state and disable auto memory as well, because some managed/global config and auto memory can be read regardless of `settingSources`.

B) Keep `settingSources: []`; it disables all managed settings, global config, and auto memory.

C) Use `settingSources: ["local"]`; local settings are always more isolated than an empty setting source list.

D) Move all tenant prompts into `CLAUDE.md`; project memory overrides host state in the SDK.

Correct Answer: A

The Agent SDK documentation warns that `settingSources` controls user/project/local filesystem settings, but managed policy, global `~/.claude.json`, and auto memory may still be read. Multi-tenant isolation needs tenant-isolated filesystem state and explicit auto-memory disablement.

Reference: docs/Claude Code/Agent SDK/Core concepts/Use Claude Code features.md

---

## Question 10

TrialBridge enables extended thinking for complex adverse-event triage. The team also wants to force Claude to call exactly one of several extraction tools by setting `tool_choice: {"type": "any"}`.

What should they know?

A) Extended thinking requires `tool_choice: {"type": "tool"}` so the model has a fixed reasoning target.

B) Extended thinking with tool use supports only `auto` or `none`; forced tool choices like `any` or a named tool are incompatible.

C) Forced tool choice is required when using any tool with extended thinking.

D) `tool_choice` has no interaction with extended thinking and can be configured freely.

Correct Answer: B

Claude's extended-thinking tool-use limitation allows the default/auto behavior or no tools, but not forced tool use via `any` or a specific named tool. Forcing a tool conflicts with the thinking flow.

Reference: docs/Claude API/Build/Model capabilities/Building with extended thinking.md

---

## Question 11

The team wants a reusable "trial-readiness-review" Skill available to SDK agents. They plan to register it dynamically in code with a Python function and rely on the Skill to restrict its own tools through `allowed-tools` frontmatter.

What is the correct SDK pattern?

A) Register the Skill programmatically with the SDK and put `allowed-tools` in the function decorator.

B) Store the Skill in `.mcp.json` so MCP discovery loads it as a tool.

C) Create `.claude/skills/trial-readiness-review/SKILL.md`, load filesystem settings through `settingSources`, and include `Skill` in `allowedTools`; enforce tool access through SDK options.

D) Put the Skill text in `CLAUDE.md`; the SDK treats every heading as an invocable Skill.

Correct Answer: C

Agent Skills in the SDK are filesystem artifacts, discovered through setting sources. The SDK does not provide a programmatic registration API for Skills, and `allowed-tools` frontmatter is not the SDK tool-control mechanism.

Reference: docs/Claude Code/Agent SDK/Customize behavior/Agent Skills in the SDK.md; docs/Claude Code/Agent SDK/Core concepts/Use Claude Code features.md

---

## Question 12

TrialBridge repeatedly answers questions over the same long protocol PDF with citations enabled. The team wants to reduce input cost using prompt caching.

Which caching statement is correct?

A) Citation sub-blocks can be cached directly, so cache the generated citation list and omit the PDF next time.

B) Prompt caching cannot be used with citations under any circumstances.

C) Cache the top-level document content block that citations reference; citation sub-content itself is not cached directly.

D) Toggle citations on and off between calls to maximize cache reuse across cited and uncited answers.

Correct Answer: C

Prompt caching can work with citations by caching the top-level document content block. The citation sub-content is not cached directly, and toggling citations changes the system prompt, which can disrupt cache reuse.

Reference: docs/Claude API/Build/Context management/Prompt caching.md; docs/Claude API/Build/Model capabilities/Citations.md

---

## Question 13

The site-payments MCP server wants to collect non-sensitive invoice metadata through form mode elicitation. A developer proposes a deeply nested schema containing arrays of invoice line-item objects, each with nested tax fields.

What is the issue?

A) Form mode accepts only free-text strings, so JSON Schema is never valid.

B) Form mode can request secrets if the schema marks fields as encrypted.

C) URL mode is required for all invoice metadata, even when no credentials or payment authorization are requested.

D) Form mode schemas are restricted to flat objects with primitive properties and simple enum patterns; complex nested objects are not supported.

Correct Answer: D

MCP form mode elicitation intentionally supports a restricted subset of JSON Schema for client UX: flat objects with primitive fields and simple enum forms. Sensitive credentials or payment authorization still require URL mode, but ordinary non-sensitive metadata may use form mode within those schema limits.

Reference: docs/Model Context Protocol/Specification/Client Features/Elicitation.md

---

## Question 14

TrialBridge can draft site communications and can also submit an operational action to suspend enrollment at a site. The suspension action has compliance and business impact.

What is the best design?

A) Hide the suspension tool from the user and let Claude call it only when confidence is high.

B) Keep drafting automated, but require explicit human approval and deterministic prerequisite checks before executing enrollment-suspension actions.

C) Let the tool execute automatically, then notify compliance after the fact.

D) Replace the tool with a prompt instruction that says "never suspend enrollment unless necessary."

Correct Answer: B

High-impact real-world actions need enforced human approval and deterministic prerequisites. Prompt instructions and tool descriptions help guide the model, but they are not adequate enforcement for consequential side effects.

Reference: Official Exam Guide/Foundations Certification Exam Guide.md; docs/Model Context Protocol/Specification/Server Features/Tools.md

---

## Question 15

TrialBridge uses server tools and occasionally receives an API response with `stop_reason: "pause_turn"` while a server-side tool loop is not finished.

How should the application continue?

A) Treat `pause_turn` as equivalent to `end_turn` and show the partial response to the user.

B) Drop the assistant response and retry the original prompt from scratch to avoid polluted state.

C) Convert the next call into a Message Batch because `pause_turn` means the request timed out permanently.

D) Add the paused assistant response to the conversation and make another API request so Claude can continue from where it paused.

Correct Answer: D

`pause_turn` means the server-side loop hit its iteration limit before the work was complete. The application should continue the conversation by preserving the paused assistant response and sending another request.

Reference: docs/Claude API/Build/Tools/How tool use works.md; docs/Claude API/Build/Building with Claude/Handling stop reasons.md
