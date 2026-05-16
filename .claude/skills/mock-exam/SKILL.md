---
name: mock-exam
description: Use when the user wants a generated/simulated CCAF mock exam (NOT the original Practice Exam questions). Triggers on phrases like "模試を作って/受けたい", "オリジナルの模擬試験", "新しい問題で模試", "mock exam", "新規に問題を作って". Synthesizes fresh multiple-choice questions grounded in `Official Exam Guide/Foundations Certification Exam Guide.md` and `docs/`, weighted by exam domain, presents them one at a time, and grades with explanations citing the source docs.
---

# Mock Exam Mode

Generate a fresh CCAF-style mock exam from the local documentation and run the user through it one question at a time. This mode does NOT use questions from `Practice Exam/` — those are reserved for the `practice-exam` skill. Use them only as a style reference.

## Sources

- **Exam structure & scope**: `Official Exam Guide/Foundations Certification Exam Guide.md` (domains, task statements, in-scope/out-of-scope appendix, sample question style)
- **Technical ground truth** for question authoring and explanations:
  - `docs/Claude API/` — Messages API, prompt caching, batch, tool use, structured output, files, evals, MCP from API side, etc.
  - `docs/Claude Code/` — Settings (`Settings.md`, `Permissions.md`, `Sandboxing.md`), CLI flags, hooks, slash commands, subagents, Agent SDK
  - `docs/Model Context Protocol/` — Spec, transports, capabilities, lifecycle, registry
- **Practice Exam files** (`Practice Exam/*.md`) — for tone and difficulty calibration ONLY. Never copy or paraphrase these questions in the mock exam.

## Domain Weights (from the exam guide)

| Domain | Weight |
|---|---|
| 1. Agentic Architecture & Orchestration | 27% |
| 2. Tool Design & MCP Integration | 18% |
| 3. Claude Code Configuration & Workflows | 20% |
| 4. Prompt Engineering & Structured Output | 20% |
| 5. Context Management & Reliability | 15% |

## Workflow

### 1. Configure the exam

When the skill is invoked, ask the user (briefly, in one message):

- **問題数**: default 25 (1問あたり ~1.5分 想定で full-length なら 50)
- **シナリオ縛り**: 任意 — e.g. "Customer Support Agent を題材にした問題だけ" / "MCP 寄りに" / おまかせ (default: 全ドメインを weight 通りに)
- **言語**: 英語(本番準拠) / 日本語 / 混在 (default: English, matches the real exam)
- **難易度**: standard / hard (default: standard, matching the practice exam tone)

If the user just says "start" or "おまかせ", use defaults: 25 questions, all domains, English, standard.

### 2. Plan the question distribution

Compute how many questions go to each domain by applying the weights to the chosen total. Round to integers and adjust the largest bucket if needed so the sum matches.

Within each domain, vary the **task statement** covered (the exam guide enumerates them per domain). Don't put 5 prompt-engineering questions all about the same sub-topic.

Decide on an over-arching **scenario** (or a couple of scenarios) so the questions feel like a coherent exam — e.g. a customer-support agent rollout, a CI integration, a research multi-agent system. Reuse scenario archetypes from the practice exam files, but invent fresh situations.

Output a brief plan to the user before starting (one short paragraph: scenario, domain breakdown, total). Don't ask for approval unless they want to tweak it.

### 3. Author each question

For each question, BEFORE presenting it:

1. Pick the domain + task statement.
2. **Read the relevant source** in `docs/` (or the exam guide) to ground the question in real, verifiable behavior. Do not invent flags, settings keys, API parameters, or MCP methods — every technical claim in the question stem, the choices, and the explanation must be checkable against the docs.
3. Draft 1 correct answer and 3 plausible distractors. Distractors should be:
   - Wrong but tempting (a common misconception, a deprecated approach, a near-miss flag name, or a correct statement that doesn't address *this* question's constraint)
   - Mutually exclusive with the correct answer
   - Roughly equal in length and specificity to the correct answer (length asymmetry is a tell)

**Beware "length asymmetry by accident."** A common authoring failure: drafting the correct answer with embedded reasoning ("X because Y, combined with Z for defense in depth"), while distractors are short and snappy. Test-takers can then guess by length alone — "the long one with multiple sentences is usually right." **Counter this actively:**
- Keep the correct answer's choice text *roughly the same word count* as the distractors. Save deep reasoning for the `explanation` block below the choices.
- If a distractor needs to be long (e.g., to spell out the wrong-but-tempting logic), pad the correct answer to match — or trim the correct answer to match the distractor brevity.
- Re-read your 4 choices before finalizing: if the longest is the correct one and the shortest is a flippant distractor, swap or rebalance.
- Quick sanity check: word-count each choice. If the correct answer is more than ~30% longer than the average distractor, rewrite.
4. Randomize choice order using true randomness (`shuf` via Bash, or `python -c "import random; ..."`).
5. Write a short explanation that names the doc / section the answer comes from.

**Balance the correct-answer distribution across A/B/C/D.** Per-question RNG alone tends to drift toward one or two letters across a 15-question scenario (LLM "randomness" instincts are biased — when you mentally place the correct answer, you tend to gravitate toward A or B). Counter this explicitly: before starting a scenario, generate a balanced shuffle of target letters (e.g., for 15 questions: `python3 -c "import random; pool = ['A']*4 + ['B']*4 + ['C']*4 + ['D']*3; random.shuffle(pool); print(pool)"`), and place each question's correct answer at the pre-assigned target. Some unevenness is fine (4/4/4/3 or 5/4/3/3 is normal); what to avoid is obvious skew like 11/3/1/0. Check the running tally as you go.

### 4. Present one question at a time

```
### Mock Exam — Question N of TOTAL  (Domain: <domain>)

<scenario context — only on the first question of a new scenario, or when context shifts>

<question stem>

A) ...
B) ...
C) ...
D) ...
```

Wait for the user's answer. Do not reveal the answer or hint at it.

### 5. Grade and explain

After the user answers:

- ✅ / ❌ + correct letter and choice text
- 1–3 sentence explanation grounded in the source
- A `Reference:` line pointing to the file that supports the answer (e.g. `Reference: docs/Claude Code/ Configuration/Settings.md` — use the actual path you used)
- Update running score and the per-domain tally

Then proceed to the next question without further prompting, unless the user wants to discuss.

### 6. End of exam

After the last question, show:

- **Total**: X / TOTAL (Y%)  with a pass/fail line at 720/1000 (72%) — the real exam threshold
- **Per-domain breakdown** as a small table: domain · attempted · correct · %
- **Weakest domains** (below 72%) with concrete `docs/` reading suggestions
- Offer to: (a) generate a follow-up mock focused on weak domains, (b) switch to `practice-exam` for the official practice questions

## Quality Rules

- **No fabricated technical details.** Every flag, setting, API parameter, MCP method, or tool name in a question or answer must exist in the local docs. If you can't verify it, don't write the question.
- **No copying from `Practice Exam/`.** These questions exist already in `practice-exam` skill. Mock questions must be original. Drawing on the same scenario *theme* (e.g. CI/CD review pipeline) is fine; reusing question stems or distractors is not.
- **One correct answer.** No "select all that apply", no "best of two correct" unless the exam guide samples explicitly use that style — the practice exam is single-best-answer, so match that.
- **Domain labeling must be honest.** If you can't cleanly map a question to one of the 5 domains, rewrite it.
- **Randomize choice order with a real RNG.**
- **Verbatim explanations are fine to draft from docs**, but rewrite long doc passages into 1–3 tight sentences.
- One question per message. Never batch.
- Scenario context paragraphs should be short — 2–4 sentences — and only repeated when the scenario shifts.

## Authoring checklist (run mentally before sending each question)

1. Does this question test a task statement listed in the exam guide for this domain?
2. Is the correct answer verifiable against a specific section of `docs/` or the exam guide?
3. Are all four choices about the same thing (so the question isn't trivially eliminated by topic)?
4. Are the distractors plausible to someone who *almost* knows the answer?
5. Did I randomize the choice order with `shuf` or equivalent?
6. Is the question stem self-contained (the user doesn't need to remember details from earlier questions)?

If any answer is "no", revise before sending.
