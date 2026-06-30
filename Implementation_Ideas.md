# Implementation Ideas

---

## 1) Architectural Flaws

### Coupling Issues
- Hard coupling via duplicated constants/regex logic across scripts (module taxonomy and intent logic).
- Multiple scripts assume precise artifact formats with no shared schema library.

### Separation of Concerns
- CLI, transport, parsing, transformation, and persistence are mixed in same files.
- Domain logic (classification/risk/coverage scoring) is not isolated from infrastructure.

### Code Organization
- Flat `scripts/` directory with very large scripts; lacks package boundaries like `core/`, `adapters/`, `domain/`.

### Scalability Limitations
- Recursive fetch + sequential processing + sleep-based throttling in `scripts/confluence-kb-builder.py`.
- In-memory processing of full story/report datasets in `scripts/generate-report.py`.

### Testability Problems
- No test suite discovered.
- Heavy use of globals/env/os paths and direct file/network calls makes unit testing difficult.

### Performance Bottlenecks
- Double file reads; regex-heavy repeated scans; no caching layer for repeated classifications.

### Security Concerns
- Hardcoded credential in `jira_fetcher.py`.
- Potential markdown/script injection paths from external content are not explicitly sanitized.

### Error Handling & Resilience
- Non-specific exception handling with weak propagation and partial failure semantics.

### Duplication
- Jira fetcher duplication and repeated frontmatter parsing are clear DRY violations.

---

## 2) Context Window Management

### What to Implement

**Context pack builder (don't feed whole files):**
- Rank sections by relevance (ticket ID, module, AC terms)
- Cap by token budget (e.g., 12k/24k depending on model)

**Three-layer context:**
- **Core:** story + AC + module KB overview
- **Relevant snippets:** top-N entity docs / historical test patterns
- **Optional tail:** low-priority context only if budget remains

**Map-reduce prompting for large domains:**
- Stage A: summarize each source doc
- Stage B: synthesize test strategy from summaries

**Rolling memory file per story:**
- `output/{story}/context-memory.json` with extracted facts and citations

### Automation Rule
- Always run context budget check before invoking test generation.
- If over budget: auto-drop least relevant chunks and log what was dropped.

---

## 3) Add Schema Validation Gates Between Pipeline Stages

### Plain English
Every stage should check: "Is this file structure valid?" before continuing. If invalid, stop early with clear errors. It's like airport security between flights: bad data cannot board the next stage.

### Example in Your Pipeline
- Stage A creates `strategy.json`
- Stage B reads it to create `test-cases.json`

Each stage validates the output of the previous stage before proceeding.

---

## 4) Add Baseline Test Suite for Core Logic

### Plain English
Before doing bigger refactoring, add tests for your most important logic so you can change code without fear. You don't need 100% coverage — start with the critical functions.

### What to Test First
- `parse_frontmatter()` behavior
- Module classification rules
- Intent derivation
- Validation scoring functions
- Duplicate detection/coverage checks
