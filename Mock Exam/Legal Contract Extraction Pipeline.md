# Scenario 4: Structured Data Extraction

Scenario: **Praxis Legal**, a 90-engineer legal-tech company, is building an AI-powered contract analysis pipeline. The system ingests PDFs of procurement contracts, NDAs, and employment agreements (10–80 pages each, often messy OCR output) and extracts structured fields — parties, effective dates, payment terms, termination clauses, governing law, indemnity provisions — to populate the company's contract management system. The team builds the production pipeline against the Claude API directly; engineers also use Claude Code for developing and iterating on the extraction prompts and evaluating the system. Volumes are ~3,000 contracts/month with strict accuracy and auditability requirements.

Part of a full-length CCAF Foundations mock exam (4 scenarios × 15 questions = 60). Pass threshold: 43/60 (72%).

---

## Question 1

Praxis needs to extract structured contract data. For each contract, the output schema includes mandatory fields (`parties`, `effective_date`) and optional fields that aren't always present (`indemnity_cap_amount`, `termination_for_convenience_notice_days`). For optional fields, the team wants the extractor to clearly indicate "not present in document" rather than guessing.

Which approach gives the most reliable behavior on optional fields?

A) System prompt: "If a field is missing, write 'N/A' as a string. Always return valid JSON." Probabilistic but standard pattern.

B) Make every field required and instruct Claude to fill missing ones with an empty string `""`; downstream code interprets empty as missing.

C) Define a tool `record_contract` with `input_schema` that declares optional fields as `{"type": ["number", "null"]}` (or `["string", "null"]`), and instruct the description: "Set optional fields to `null` ONLY when the clause is genuinely absent from the document; do not infer." Forced tool use guarantees the schema shape; the description guides the null-vs-infer decision.

D) Split into two prompts: one extracts mandatory fields, a second specifically asks "does the contract contain an indemnity cap? a termination-for-convenience clause?" with yes/no answers. Two-pass is simpler than schema design.

Correct Answer: C
Two layers stacked: structural (schema) — `{"type": ["number", "null"]}` makes `null` an explicit, type-checked option; the model can't return "N/A", an empty string, or omit the key — only the documented values. Semantic (description) — telling the model when to use null ("clause is genuinely absent; do not infer") shapes the judgment call. A is wrong because free-text JSON via system prompt is probabilistic. B is wrong because empty string conflates "absent" with "present but empty" — semantically distinct cases collapsed. D is wrong because a two-pass approach doubles cost and the second pass still needs reliable structured output.

Reference: docs/Claude API/Build/Tools/, docs/Claude API/Build/Structured outputs.md

---

## Question 2

Praxis is processing contracts that range from 5 pages to **80 pages** (~120k tokens for the longest ones). A single `messages.create` call with the full document works for small contracts but for 80-page contracts the team observes:
- Findings from the middle of the document are inconsistent
- Total cost is high (every input token billed each time)

Which approach best balances quality and cost?

A) Always pass the entire document and trust Claude's long-context capability; modern models handle 200k tokens routinely.

B) Truncate each contract to its first 30 pages; legal documents typically front-load the important provisions.

C) Use Claude's vision-capable model only — vision models have better attention over long documents.

D) Chunk the document by section (e.g., split on headings like "Termination," "Indemnification," "Governing Law"), extract the relevant fields from each chunk in parallel calls, and combine results in a final synthesis step. Combine with **prompt caching** (`cache_control`) for the system prompt + few-shot examples so the cached portion is re-used across chunks at ~90% cost reduction.

Correct Answer: D
Three independent optimizations stacked: section chunking addresses position bias; parallel extraction reduces wall-clock latency; prompt caching reuses the cached prefix across chunks at ~90% input-token cost for cache hits. The synthesis step reconciles cross-section dependencies. A is wrong because even with 200k-token windows, attention degrades in the middle of long inputs. B is wrong because legal contracts often have load-bearing clauses in the middle and end. C is wrong because vision models help with image-based inputs but don't fix attention bias over long text.

Reference: docs/Claude API/Build/Building with Claude/Prompt caching.md, exam guide Domain 5

---

## Question 3

Praxis's prompt for the contract extractor has grown to ~3,500 tokens of system content: schema description, role guidance, 6 few-shot examples covering ambiguous clauses, and explicit rules. The lead engineer notices Claude occasionally confuses the few-shot examples (treats them as the *current* document to extract from) and produces wrong outputs.

What is the standard remedy?

A) Wrap each section of the system prompt in **XML tags** with clear semantic names: `<role>`, `<schema>`, `<rules>`, `<examples>` (with each example in its own `<example>` block), and `<current_document>` for the input. Anthropic's prompt-engineering guide recommends XML tags as a primary structural tool to help Claude distinguish sections.

B) Move the few-shot examples to a separate API call that runs first; Claude can't confuse examples with input if they aren't in the same call.

C) Shorten the system prompt by removing 4 of the 6 few-shot examples; fewer examples means less confusion.

D) Switch to a smaller model; smaller models are less prone to over-attention on examples.

Correct Answer: A
XML tags are one of Anthropic's most-recommended prompt-engineering primitives. Claude is trained to attend strongly to XML structure, and tagged sections give the model clear demarcations between "examples to learn from" and "current input to process." This eliminates the "Claude treated example #3 as the input" failure mode. B is wrong because separating into two calls disconnects examples from the inference step, defeating their purpose. C is wrong because the 6 examples cover ambiguous clauses — they're valuable. D is wrong because smaller models have worse instruction-following overall.

Reference: docs/Claude API/Build/Prompt engineering/ (XML tags pattern)

---

## Question 4

Praxis's contracts arrive as PDFs (mix of native PDF and scanned/OCR'd images). The current pipeline pre-processes PDFs into plain text using a third-party OCR library, then sends the text to Claude. The team is debating whether to switch to sending PDFs directly to Claude.

Which is the most accurate framing for this decision?

A) Always send pre-extracted text — Claude does not natively support PDF inputs and any "PDF support" is a third-party wrapper.

B) The Claude API supports **PDF documents as inputs** via the document content block (or the Files API for re-use across requests). For scanned PDFs, Claude's vision capability handles OCR natively, often more accurately than a generic OCR library because it preserves layout context. The tradeoff: PDF processing costs more input tokens than equivalent extracted text. For native (non-scanned) PDFs with clean text, pre-extraction may still be cheaper; for scanned/messy PDFs, sending the PDF directly often gives better quality.

C) Always send PDFs directly — Claude's vision is universally better than any OCR library, regardless of cost.

D) Convert PDFs to JPEG images and pass them as image inputs; this is the only way to use Claude's vision on PDF content.

Correct Answer: B
The Claude API accepts PDFs natively via document content blocks (or referenced via the Files API for repeat use). Vision-based OCR for scanned PDFs is often more accurate than third-party OCR because Claude preserves layout context (tables, multi-column, signatures). Cost tradeoff is real: PDF tokens > equivalent extracted text. Right answer is contextual — native clean PDFs may favor pre-extraction; scanned/messy favor direct PDF. A is wrong because PDF support is native. C is wrong because "universally better" is too absolute. D is wrong because PDF→JPEG conversion is unnecessary.

Reference: docs/Claude API/Build/Files API, docs/Claude API/Build/Building with Claude/ (PDF support, vision)

---

## Question 5

Praxis's extractor occasionally mislabels jurisdiction-of-governing-law. Specifically, when a contract has clauses like "Governed by the laws of the State of Delaware, USA, except that disputes arising in California will be heard in California courts," Claude sometimes returns `"jurisdiction": "California"` (focusing on the latter clause) when the team wants `"Delaware"` (the actual governing law).

Which intervention is most likely to fix this *specific* failure?

A) Switch to a larger model and re-run the same prompts; the failure is a model capacity issue.

B) Add 2–3 **targeted few-shot examples** showing contracts with both a primary governing-law clause AND a separate forum/venue clause, each labeled with the correct governing-law extraction and brief reasoning ("Delaware is the governing law; California is the dispute-forum venue, not the governing law"). The exam guide highlights this exact pattern.

C) Add a system-prompt rule "Extract the FIRST jurisdiction mentioned" and rely on order heuristic; this is robust because governing law typically appears first.

D) Output multiple labels and let downstream consumers decide; the model shouldn't have to disambiguate.

Correct Answer: B
The exam guide explicitly calls out targeted few-shot examples for ambiguous scenarios with reasoning showing why one choice was made over plausible alternatives. The failure is specifically about distinguishing governing law vs forum/venue — closely-related but legally distinct concepts. Showing worked examples with explicit reasoning teaches the actual distinction. A is wrong because model capacity isn't the issue; ambiguity is. C is wrong because "first mentioned" is a fragile heuristic. D is wrong because it passes the problem downstream.

Reference: Official Exam Guide Domain 4:420 (few-shot examples for ambiguous scenarios)

---

## Question 6

Praxis's extractor sends the same 5,000-token system prompt (role + schema + 6 few-shot examples + rules) on every contract — and processes 3,000 contracts per month. The lead engineer wants to reduce input-token costs.

Which Claude API mechanism is the direct fit?

A) Use the Message Batches API; batches are billed at 50% input rate, which covers the system prompt re-sending.

B) Switch to a smaller model variant; smaller models have proportionally cheaper input tokens.

C) Compress the system prompt to under 1,000 tokens; brevity is the only path to cost reduction.

D) Enable **prompt caching** by marking the system prompt with `cache_control: {type: "ephemeral"}`. Cached input tokens are billed at ~10% of the normal rate on cache hits (5-minute or optionally 1-hour TTL), giving substantial savings for repeated identical prefixes across the 3,000-contract pipeline.

Correct Answer: D
Prompt caching is the direct, documented mechanism for "I'm re-sending the same prefix many times." Marking the system prompt with `cache_control` makes those tokens billable at ~10% on cache hits. Default TTL is 5 minutes; 1-hour TTL is available via opt-in. A is wrong because Batches gives 50% off but is async (up to 24h latency) — different tradeoff. B is wrong because smaller model is a quality tradeoff, not a direct cost mechanism for repetition. C is wrong because compression loses signal; caching keeps the prompt at ~10% cost.

Reference: docs/Claude API/Build/Building with Claude/Prompt caching.md

---

## Question 7

Praxis's pipeline currently uses a single Claude API call per contract: pass the document + schema, get back the structured extraction. The product team is now adding *contract quality checks* (e.g., "does this contract have any missing standard clauses for our company's risk profile?") and *risk scoring* (e.g., "rate the indemnity exposure 1-5").

An engineer proposes building this as one giant prompt that does extraction + quality check + risk scoring in one call.

What is the strongest argument for *decomposing* this into separate calls or stages instead?

A) The Claude API caps each request at 4k input tokens; a giant prompt would exceed the limit.

B) Quality checks and risk scoring need access to *external data* (Praxis's risk policies, missing-clause checklists) that the extraction call doesn't need; bundling forces every call to load that extra data.

C) The Message Batches API can only run one task type per batch; mixing extraction + scoring would force separate batch infrastructure anyway.

D) Each task has distinct success criteria, distinct evaluation needs, and benefits from a focused prompt. Combining them dilutes attention, makes failures harder to attribute (which sub-task regressed?), and prevents independent prompt iteration. The team can still run them as sequential stages with prompt caching to keep the extraction output flowing into downstream stages cheaply.

Correct Answer: D
The architectural principle: focused prompts keep Claude's attention on one objective at a time; attribution becomes possible when each stage has its own pass/fail; independent iteration prevents cross-regressions; per-stage evals get their own golden sets. Prompt caching keeps the multi-stage approach cost-competitive. This is the complement to "don't over-decompose" — split unrelated objectives, don't split one coherent task. A is wrong; no 4k cap. B is a real concern but not the strongest argument. C is wrong; Batches has no "one task type per batch" restriction.

Reference: docs/Claude API/Build/Agents and tools/Building agents with the Claude API.md (decomposition principles)

---

## Question 8

Praxis needs every extracted field to be **auditable**: legal teams need to verify each extracted value against the source document, with page/line citations. The current pipeline returns just `{"governing_law": "Delaware"}` with no source attribution, making audits painful.

Which tool/schema design best fits?

A) Add a post-processing step: after extraction, run a second Claude API call asking "for each extracted field, find the exact page and line where you saw it" and merge results.

B) Add `provenance` as a free-text field to each extraction (e.g., `"governing_law_provenance": "page 12"`); free text gives Claude flexibility.

C) Change the tool's `input_schema` so each field is an object: `{"value": "Delaware", "evidence_quote": "...", "source_location": {"page": 12, "lines": "5-7"}}`. Forced tool use with this richer schema guarantees every extracted value comes with a verbatim quote and location. Instruct the model in the description: "Always copy `evidence_quote` verbatim from the document; do not paraphrase."

D) Return only the structured extraction and store the source PDF for human auditors to look up manually; citation generation is unreliable from LLMs.

Correct Answer: C
The canonical structured-citation pattern: schema-level commitment means every extracted field structurally requires its evidence (model can't return a value without a quote). Verbatim-quote requirement catches hallucinated paraphrases (you can grep the source for the quote). Source location lets reviewers jump to the spot in the PDF. Scalable to 3,000 contracts/month. A is wrong because two-pass citation generation can hallucinate citations to back up the first call's outputs. B is wrong because free-text provenance is unstructured. D is wrong because human-only audit doesn't scale.

Reference: Official Exam Guide Domain 5:590

---

## Question 9

Praxis engineers iterate on extraction prompts using Claude Code. A common workflow is: take a recent contract that the production system mis-extracted, paste it into Claude Code, run the current prompt against it, compare to expected output, and iterate. The team wants to package this workflow as a reusable invocation.

What's the most appropriate Claude Code feature?

A) Create a Claude Code **skill** at `.claude/skills/test-extraction-prompt/SKILL.md` checked into the repo, with frontmatter (`description`, `allowed-tools: Read, Bash(...)`) and body describing the workflow: "Load the current prompt from `prompts/extract.md`, run it against the contract specified in the argument, compare to the expected output in `evals/<id>.json`, report differences." Engineers invoke it as `/test-extraction-prompt <contract-id>`.

B) Hardcode the workflow as a section in the root `CLAUDE.md` titled "## Prompt iteration workflow" and rely on Claude to apply it whenever prompted.

C) Build a custom MCP server `praxis-prompt-tester` and expose tools to load/run/compare; MCP is the right mechanism for any multi-step workflow.

D) Write a Bash script in `scripts/test-prompt.sh` and tell engineers to invoke it manually; Claude Code workflows belong in shell scripts, not Claude Code features.

Correct Answer: A
Skills are purpose-built for reusable, invocable, multi-step workflows: clear name, `description` for auto-load, `allowed-tools` to restrict the tool surface, optional `argument-hint`, and a body describing the steps. Checked into `.claude/skills/`, every engineer gets the same workflow on clone. B is wrong because CLAUDE.md is for persistent context, not invocable workflows. C is wrong because MCP is for external service connections, not internal workflows. D is wrong because a bare Bash script disconnects from the Claude Code workflow entirely.

Reference: docs/Claude Code/Build with Claude Code/Tools and plugins/Extend Claude with skills.md

---

## Question 10

Praxis's extractor produces a structured output for each contract. The quality team wants to **detect** when an extraction is likely wrong before sending it downstream. They notice that for ambiguous or unusual contracts, the extractor sometimes confidently outputs values that are clearly mistaken on human review.

What is the most effective architectural addition?

A) Lower the extraction call's temperature to 0 to make outputs more deterministic; deterministic outputs are inherently more trustworthy.

B) Have the same extractor self-grade its own output by appending "Are you confident in this extraction? (yes/no)" to the prompt; self-grading is the standard reliability pattern.

C) Add an **independent verifier agent** as a second Claude API call (separate prompt, no shared context) that examines the extracted fields against the source document and flags low-confidence or contradicted extractions. This is the evaluator-optimizer pattern: the verifier has no anchoring on the first call's reasoning, so it catches errors a self-grader would defend.

D) Manually review every extraction; LLM verification is fundamentally unreliable for this use case.

Correct Answer: C
The canonical evaluator-optimizer pattern: the verifier starts fresh, examines the structured output against the source document, and casts an independent judgment. Disagreement surfaces cases needing human attention. A is wrong because `temperature=0` makes wrong answers consistent, not correct. B is wrong because same-context self-grading suffers from confirmation bias — the model defends its own output. D is wrong because 3,000 contracts/month makes full manual review impractical.

Reference: Official Exam Guide Domain 1:480

---

## Question 11

Praxis's CI runs nightly evals: 100 hand-curated contracts with expected extractions. The eval job invokes Claude Code in headless mode, runs the current production prompt against each contract, compares output to expected, and reports pass-rate. The job is one of several CI workflows.

Which Claude Code invocation pattern is right for this CI job?

A) `claude --interactive "run extractions and report results"`; interactive mode is needed to handle the per-contract processing.

B) `claude --eval mode`; Claude Code has a built-in eval flag for this exact use case.

C) `claude -p "<prompt>" --output-format json --settings .claude/ci-settings.json` per contract, with permissions pre-configured in `ci-settings.json` to allow only the tools the eval needs (Read, specific Bash commands). The `-p` mode runs headless, `--output-format json` produces machine-parseable results, and the CI-specific settings file isolates eval permissions from local-dev settings.

D) Use the Message Batches API directly; Claude Code is not designed for eval workloads.

Correct Answer: C
The right pieces stacked: `-p` (headless mode, no TTY), `--output-format json` (machine-parseable response envelope), `--settings <file>` (CI-specific permissions config isolating eval permissions from local dev). The CI runs each contract, collects JSON outputs, diffs against expected, and emits pass/fail. A is wrong because `--interactive` keeps TTY active. B is wrong because there's no `--eval` flag. D is wrong because Claude Code IS designed for headless CI; Batches has up-to-24h latency, wrong tool for nightly evals.

Reference: docs/Claude Code/Reference/Reference/CLI reference.md, docs/Claude Code/Getting started/Use Claude Code/Headless mode.md

---

## Question 12

Praxis wants the extractor to express **uncertainty** when a clause is ambiguous (e.g., a poorly-OCR'd date, a clause that could be interpreted two ways). The team doesn't want Claude to silently pick one interpretation; they want a structured signal that a human should review.

How should this be wired into the extraction?

A) Add a `confidence` field to the schema (enum: `"high" | "medium" | "low"`) and a `notes` field for free-text explanation when confidence isn't high. Instruct in the description: "Set confidence to 'low' when the clause is OCR-degraded, ambiguous, or interpretable in multiple ways. Always populate `notes` with the specific reason for any non-`high` confidence." Downstream code routes `low` extractions to human review.

B) Have Claude reply with prose for ambiguous clauses and JSON for confident ones; downstream code branches on whether the response parses as JSON.

C) Drop ambiguous clauses silently from the extraction; downstream code can fall back to "field absent" handling.

D) Run the extraction twice and flag mismatches as ambiguous; agreement = confident, disagreement = uncertain.

Correct Answer: A
Schema-level commitment (typed enum, Claude can't omit or invent values), actionable signal (`low` is routable to human review queues), and auditable (`notes` captures why). The pattern works because Claude can self-assess "I'm not sure" reasonably well when given structured outlets for the assessment. B is wrong because mixed prose+JSON breaks parsing. C is wrong because silent dropping is exactly the failure mode the team wants to avoid. D is wrong because two-run mismatch costs 2× and doesn't capture intrinsic uncertainty.

Reference: docs/Claude API/Build/Tools/, exam guide Domain 4 + Domain 5

---

## Question 13

Praxis's product team wants to measure extraction quality so they can decide when to ship prompt changes. The engineering manager proposes three eval strategies:

- **Strategy 1**: Use Claude itself to grade Claude's extractions (LLM-as-judge), aggregating pass-rate
- **Strategy 2**: Hand-curate a golden set of 100 contracts with expected extractions; compare outputs field-by-field; produce a pass-rate per field
- **Strategy 3**: Track customer-reported errors from the contract management system; treat the rate as the quality signal

Which strategy is most appropriate as the **primary** gate for shipping prompt changes?

A) Strategy 2 — hand-curated golden set with field-level comparison. It's reproducible, runnable in CI, debuggable (you can open the specific failing contract), and decoupled from customer behavior. LLM-as-judge can be a useful *supplementary* signal alongside, and customer-reported errors are a valuable *trailing* outcome metric, but neither is the shipping gate.

B) Strategy 1 — LLM-as-judge. It scales arbitrarily and requires no human curation.

C) Strategy 3 — customer-reported errors. End-user outcomes are what ultimately matter; intermediate proxies mislead.

D) Strategy 1 + 3 combined — automated breadth plus real-user signal; Strategy 2 is too expensive to curate in a fast-moving environment.

Correct Answer: A
For shipping decisions the eval must be reproducible (same input → same score → CI-runnable), stable (no drift), debuggable (open the failing contract), and owned by the team. Field-level comparison shows which fields regress. LLM-as-judge and customer errors are useful supplementary/trailing signals, not gates. B is wrong as primary because LLM-as-judge is biased and not reproducible enough for shipping. C is wrong as primary because customer reports arrive post-ship and have selection bias. D is wrong because dismissing Strategy 2 dismisses the only deterministic pre-ship signal.

Reference: Official Exam Guide Domain 1 (eval design for shipping)

---

## Question 14

Praxis's pipeline currently makes 3,000 individual `messages.create` calls per month (one per contract). The engineering manager wants to reduce cost further, given that:
- Each contract's extraction is **independent** (no cross-contract dependencies)
- Results are needed within 12-24 hours, not real-time
- The team already uses prompt caching for the system prompt

Which mechanism is the right additional cost optimization?

A) Switch to a smaller model variant for all 3,000 calls; smaller models are uniformly cheaper.

B) Move the pipeline to the **Message Batches API**. Batches process asynchronously (up to 24h) and are billed at 50% of standard input/output rates. Praxis's "12-24h SLA" + "independent contracts" perfectly fits the Batches use case. Prompt caching still works inside batched requests, so the cache discount stacks with the batch discount.

C) Call the API in parallel from 100 worker threads to negotiate volume discounts with Anthropic; concurrency unlocks lower rates.

D) Pre-process contracts client-side with a local LLM and only call Claude for ambiguous cases; cost reduction comes from reducing call volume.

Correct Answer: B
The Batches API is purpose-built for Praxis's profile: independent requests, latency-tolerant (12-24h SLA matches the up-to-24h batch window), high volume. 50% off both input and output tokens. Crucially, prompt caching still works within batched requests, so discounts stack. A is wrong because model swap changes a different axis (accuracy). C is wrong because there's no concurrency-based volume discount. D is wrong because introducing a local LLM is a major architectural change with likely accuracy degradation on legal documents.

Reference: docs/Claude API/Build/ (Message Batches API)

---

## Question 15

Praxis is processing some unusually long contracts — a 200-page master services agreement with all its exhibits exceeds 250k tokens. The team's standard chunking approach (split by section, extract per chunk, synthesize) is failing on this contract because there are cross-references between sections (e.g., Section 14 references "the Indemnification Cap defined in Section 3, Exhibit B").

What's the right strategy?

A) Reject documents over 100 pages from the pipeline; the system is not designed for documents this long.

B) Adapt the chunking strategy: (i) do a **first pass** that extracts a **glossary** of cross-referenced definitions (capitalized defined terms with their definitions and source locations) from the whole document; (ii) include this glossary in the system prompt for each section-extraction call so cross-references resolve correctly; (iii) the final synthesis uses both the glossary and the per-section findings. This is the "scratchpad / shared findings" pattern adapted for long-document extraction.

C) Disable chunking and pass the full 250k-token document to Claude in a single call; long-context models handle this routinely.

D) Have engineers manually annotate each long contract before extraction; cross-reference resolution is fundamentally a human task.

Correct Answer: B
Two-pass with glossary extraction adapts the canonical "shared findings / scratchpad" pattern for long, cross-referenced documents. First pass produces a small glossary of defined terms + locations. Section passes get the glossary in context — cross-references like "Indemnification Cap defined in Section 3" resolve without re-loading the whole 250k-token document. Synthesis combines per-section findings using the glossary. Combine with prompt caching (glossary becomes part of the cached prefix) for cost-efficiency. A is wrong because rejecting valid documents is overcorrection. C is wrong because 250k tokens approaches/exceeds context limits and suffers from "lost in the middle." D is wrong because manual annotation doesn't scale.

Reference: Official Exam Guide Domain 5:556-557 (scratchpad findings pattern), Prompt caching docs
