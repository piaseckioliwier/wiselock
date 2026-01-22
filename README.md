# WiseLock

A CLI for creating and refactoring **full, large-scale changes** in a repository — **including tests**.

`wiselock` takes your task (one line), gathers context from the repo, asks model(s) to generate a **diff/patch**, optionally performs a **diff review**, and then (if you want) **applies the changes** and **runs tests**.

```bash
wiselock "Add request validation to /users" \
  --model gpt-5.2 \
  --apply \
  --run-tests
```

---

## Why this makes sense

In real-world projects, the hardest part is not “writing a few lines”, but:

- gathering the right context (where to change things, what is related),
- maintaining consistency across multiple files,
- making changes without breaking contracts and edge cases,
- ensuring tests are handled properly.

`wiselock` structures this process into a repeatable pipeline.

---

## MVP (CLI)

The MVP is a single command that:

1. **collects repository context**
2. **builds a prompt**
3. **calls the API**
4. **writes a diff (patch)**
5. **git apply** (optional)
6. **runs tests** (optional)

Example:

```bash
wiselock "Add request validation to /users" \
  --model gpt-5.2 \
  --apply \
  --run-tests
```

### Interactive mode (next step, optional)

```bash
wiselock "Add validation to /users" --interactive
```

In this mode, the tool:

- generates a patch,
- shows the diff,
- asks:

```text
Apply changes? [y/N]
```

By default, in non-interactive mode, a “safe” behavior (dry-run) is recommended, requiring an explicit `--apply`.

---

## Key detail: separating “creation” from “evaluation”

This is one of the most important principles of this approach:

- **Prompt A (generator):** generates the diff/patch
- **Prompt B (reviewer):** critically evaluates the diff/patch

Benefits:

- the model does not “defend” its own decisions (less confirmation bias),
- you get a different thinking mode (creative vs. skeptical),
- it is easier to define automatic rules: what to accept, what to block.

### The review must receive: the diff + the original task

A review without the task is practically meaningless (“is the code pretty?”).
Only together with the task can the reviewer check:

- whether the change fulfills the requirements,
- whether it does too much,
- whether it changes contracts / public APIs,
- whether it breaks edge cases,
- whether tests cover critical paths.

---

## Review response format (3 parts)

The review prompt should return a stable, machine-parseable format.

### 1) Verdict (enum, not a scale)

```text
VERDICT:
- APPROVE
- APPROVE_WITH_WARNINGS
- REJECT
```

An enum is more stable than scores, and easier to use as a gate.

### 2) Risk assessment (concrete!)

Example:

```text
RISKS:
- [HIGH] Missing validation for optional fields
- [MEDIUM] No tests for error paths
- [LOW] Minor style inconsistency
```

### 3) Confidence score (optional, auxiliary)

```text
CONFIDENCE: 0.82
```

This does not have to be a “blocker”, more of a diagnostic signal.

### Recommended minimal payload (example JSON)

```json
{
  "verdict": "APPROVE_WITH_WARNINGS",
  "risks": [
    {"level": "HIGH", "note": "Missing validation for optional fields"},
    {"level": "MEDIUM", "note": "No tests for error paths"},
    {"level": "LOW", "note": "Minor style inconsistency"}
  ],
  "confidence": 0.82,
  "notes": [
    "Consider adding tests for 400 response on invalid payload.",
    "Keep validation errors consistent with existing endpoints."
  ]
}
```

---

## CLI flow (practically)

Default pipeline:

```text
wiselock "Add validation to /users"
  ↓
[generate diff]
  ↓
[review diff]
  ↓
[decision logic]
  ↓
[apply?] → [run tests?]
```

Rule: **use a different prompt / style for review than for generation**.

- Generator: creative, solution-focused
- Reviewer: skeptical, defensive, “paranoid”, like a security/staff engineer

---

## Model selection and “custom per task”

Assumption: different stages may use different models (and different providers).

Example flags (future):

```bash
wiselock "..." \
  --model gpt-5.2 \
  --review-model claude-opus \
  --apply
```

And profiles, e.g.:

- `--profile fast` – one model, minimal context, no review
- `--profile safe` – generate + review, no auto-apply
- `--profile paranoid` – generate + review + test plan + tests, strict gates

---

## Detailed plan (multi-model workflow)

Proposed “full” flow:

1. **GPT-5.2** creates a plan
2. **Claude** performs a *peer review* of the plan and improves it
3. **Claude** implements
4. **GPT-5.2** performs a final code review and looks for hidden bugs

Why this works:

- roles are separated (planning vs. implementation vs. audit),
- peer review of the plan catches misunderstandings before code is written,
- final review by a different model increases the chance of catching subtle regressions.

---

## Tests (future) – suggested workflow

Next stage: a separate pipeline for tests.

- **GPT-5.2 Pro** decides *what to test* (minimum sensible set), identifies risky areas.
- **Claude Opus 4.5** writes the actual tests (fast, no fluff).

This separation is often very effective, because “what to test” requires prioritization and risk-based thinking, while writing tests is often heavy, mechanical work.

### Test pyramid (approximate)

```text
                ┌──────────────┐   ← very few (1–8%)
                │   E2E / UI   │
           ┌────┴──────────────┴────┐
           │ Contract / Component   │   ← 10–20%
      ┌────┴────────────────────────────┴────┐
      │          Integration                 │
┌─────┴───────────────────────────────────────┴─────┐   ← most (60–85%)
│            Unit / component-level                  │
└────────────────────────────────────────────────────┘
```

In short:

- most value comes from fast unit/component tests,
- integration tests guard the glue and real dependencies,
- contract tests protect APIs and boundaries,
- E2E/UI only where regressions hurt the most.

---

## How `wiselock` works “under the hood”

### 1) Repository context collection

Goal: provide the model with **only what is needed**, within the token budget.

Typical context sources:

- repository structure (directory tree),
- metadata: language, framework, package manager, test runner,
- key files (e.g. routers, controllers, schemas, models),
- grep/ripgrep for endpoints/names,
- dependencies (imports, related modules),
- existing tests near the change,
- conventions (lint/format, styleguide, error handling).

Eventually: context modes, e.g. `smart` (only relevant), `full` (larger context), `custom` (include/exclude globs).

### 2) Prompt construction

In practice, a prompt is a “bundle” consisting of:

- task description (your command),
- repository context (selected fragments),
- output format instructions (unified diff patch),
- constraints (do not touch X, preserve contract Y, style Z),
- test requirements (if enabled).

### 3) API calls

An adapter layer per provider/model:

- OpenAI
- Anthropic
- (future) others

### 4) Patch persistence

- patch saved to a file (audit, reproducibility),
- metadata: model, context hash, time, parameters.

### 5) Applying changes

- preferred: `git apply` (clean, reversible)
- optionally: `git apply --check` before actual application

### 6) Running tests

- executed only on demand (`--run-tests`) or via profile,
- configurable via command (`--test-command "..."`).

---

## Decisions and gates

Example decision logic after review:

- `APPROVE` → auto-apply allowed (if `--apply`/profile permits)
- `APPROVE_WITH_WARNINGS` → show warnings, prefer interactive mode
- `REJECT` → do not apply; display reasons and suggested fixes

Additional gates (optional):

- block auto-apply on `[HIGH]` risk,
- require tests for changes in critical directories (e.g. auth, payments),
- require clean `git diff --check`.

---

## Configuration

Target configuration sources (highest priority first):

1. CLI flags
2. `.wiselock.yml` (per repo)
3. environment variables (API keys, default provider)

Example (conceptual):

```yaml
profile: safe
models:
  generate: gpt-5.2
  review: claude-opus
context:
  mode: smart
  include:
    - "src/**"
    - "tests/**"
  exclude:
    - "dist/**"
    - "**/*.min.js"
tests:
  command: "npm test"
  on_apply: true
```

---

## Security and hygiene

Recommended design guardrails:

- **do not apply changes by default** without `--apply` (or confirmation in `--interactive`),
- log everything clearly: patch, parameters, review verdict,
- limits on context size and number of files,
- clear separation: the “generator” never runs commands, it only produces a patch,
- test runner executes only commands from config/flags (no “arbitrary execution” from model output).

---

## Roadmap

- `--interactive` with a convenient diff viewer
- profiles and gate policies
- automatic repair loop:
  - tests fail → summarize errors → patch fix → re-review → retry
- better “smart context” (file relevance ranking)
- CI integration (PR comments / check runs)
- support for monorepos and multiple test runners
