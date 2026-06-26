# AI- Test Case Generation System


## Architecture Overview

```mermaid
flowchart TD
    subgraph K["WHERE KNOWLEDGE COMES FROM"]
        C["Confluence
Business rules
Screen specs
API docs
Feature descriptions"]
        Q["qTest
Historical test cases"]
        C --> CB["scripts/confluence-kb-builder.py"]
        Q --> QB["scripts/qtest-kb-importer.py"]
        CB --> KB["kb/<APP_ID>/
Structured Markdown Knowledge Base"]
        QB --> KB
    end

    KB -->|"AI reads the KB as context"| J

    subgraph J["WHERE TEST CASES COME FROM"]
        Jira["Jira User Story"]
        Fetch["scripts/fetch-jira-story.py"]
        Story["user-stories/user_story_USCM-XXXXX.md"]
        Jira --> Fetch --> Story

        subgraph P["10-Agent AI Pipeline"]
            A1["1. @story-assessor"]
            A2["2. @context-auditor"]
            A3["3. @test-strategist"]
            A4["4. @test-writer"]
            A5["5. @test-reviewer"]
            A6["6. @test-refiner"]
            A1-->A2-->A3-->A4-->A5-->A6
        end

        Story --> P
        P --> Out["output/USCM-XXXXX/test-cases-refined.json"]
        Out --> Val["scripts/validate-test-cases.py"]
        Val --> QT["qTest Manager"]
    end
```

## 10-Agent AI Pipeline

1. **@story-assessor** -- "Is this story good enough to test?"
2. **@context-auditor** -- "Do we have enough KB docs?"
3. **@test-strategist** -- "What kinds of tests do we need?"
4. **@test-writer** -- Write the actual test cases.
5. **@test-reviewer** -- Punch holes; identify missing cases.
6. **@test-refiner** -- Apply reviewer fixes.
7. **Validate quality**
8. **Import to qTest**

## Data Flow

1. Fetch Jira story (`scripts/fetch-jira-story.py`)
    - Output: `user-stories/user_story_USCM-92511.md`
2. AI reads the Knowledge Base
    - Product module docs
    - Shared rules
    - Test knowledge
    - Business rules
    - API behavior
    - Edge cases
3. Story Quality Gate
    - Story scored out of 100
    - Score \<70 blocks processing
    - Output: `quality_assessment.md`
4. Knowledge Gap Check
    - Coverage report
    - Covered / Partial / Missing
    - Missing docs create Jira tasks
5. Build Test Strategy
    - Happy path
    - Error path
    - Edge cases
    - Boundary value
    - Equivalence
    - Risk areas
    - Output: `strategy.json`
6. Write Test Cases
    - Title
    - Preconditions
    - Steps
    - Expected results
    - Duplicate detection
    - Output: `test-cases.json`
7. Adversarial Review
    - Missing negative tests
    - Vague steps
    - Missing assertions
    - Output: `review-critique.md`
8. Refine & Fix
    - Apply reviewer feedback
    - Output:
        - `test-cases-refined.json`
        - `refinement-diff.md`
9. Validate Quality
    - Completeness
    - Assertiveness
    - Executability
    - Coverage
    - Target: **80/100**
10. Done
    - Import into qTest



## AI Agents

| Agent | Role |
| --- | --- |
| @story-assessor | Quality gate |
| @knowledge-curator | KB builder |
| @qtest-preprocessor | qTest importer |
| @context-auditor | Knowledge gap checker |
| @test-manager | Pipeline orchestration |
| @test-strategist | Test planning |
| @test-writer | Test generation |
| @test-reviewer | Critical review |
| @test-refiner | Apply fixes |
| @feedback-analyst | Pipeline analysis |

## Governance

- AI cannot modify its own governing instructions.
- Proposed changes require human approval.
- Governance prevents silent drift.

## Quality Metrics

Each test case is scored on:
- Completeness
- Assertiveness
- Executability
- Coverage
- Specificity
- Boundary Testing
- Negative Testing
- Non-Duplication

Targets:
- Human approval without edits: **95%**
- Per-test score: **80/100**
- Story quality gate: **70/100**
- Hallucinations: **0**


Governance - How the System Stays Safe
A key concern: *what happens if the AI tries to change its own instructions?*
This system uses a *"report-first" governance model*:

AI proposes a change to an agent instruction file, then governance-check-py runs, Generates a report: "Here is what changed and why" and Human reviewer reads it
- The AI can *suggest* - it cannot «*self-modify its own core rules.
- governance/protected-paths.json lists every file that requires human approval to change.
- This prevents "silent drift" - where the AI gradually rewrites its own behavior without anyone noticing.


|Metric | Target |
-------.-------

|Test cases passing human A review without changes | ≥ 95% |

|Per-TC validation score | ≥ 80 / 100 |

|Story quality gate (minimum to proceed) | ≥ 70 / 100 |

|Hallucinations (AI making de rules not in KB) | 0 |
## Quick Start

1.  prompt as ,Generate the TC for user stoty -xxxx


Scripts:

jira_fetcher. py
Fetches Jira user stories from the Jira REST API and saves them as structured markdown files with metadata tables. Supports pushing quality assessment reports back to Jira tickets as attachments with summary comments.
scripts/confluence-kb-builder-py
Fetches Confluence pages (with all sub-pages) for a configured application and writes structured raw-source staging files to kb-sources/{app_1d}/. Supports full fetch and incremental update modes with manifest tracking, attachment downloads, and consolidated context generation.

scripts/fetch-jira-story-py
Fetches Jira user story fields (description, acceptance criteria, linked issues) Via REST API and saves them as markdown. Supports pushing quality assessment reports and scores back to Jira tickets as attachments and comments.

scripts/generate-report. py
Generates interactive HTML reports from pipeline output artifacts (coverage, strategy, reviews, refinements) by normalizing JSON data. Supports standalone HTML (file://
compatible) with embedded data and HTTP server mode with live browser preview.

scripts/ governance-check.py
Enforces governance for protected Copilot customization files by detecting git changes and generating approval reports. Blocks commits to protected paths (agents, prompts, instructions) unless explicitly overridden with a manual review flag.

scripts/qtest-csv-to-kb-source.py
Converts Test XLSX/CSV exports into structured markdown staging files organized by KB module with automatic module detection. Supports incremental updates, duplicate detection, and auto-generates module mapping configuration for SME review.

scripts/qtest-kb-importer-py
Imports preprocessed qTest test cases into the KB structure by generating test-suite-index, enriching entity files, and updating knowledge-health. Executes Pass 4a-4e of the knowledge curation pipeline to integrate test coverall data into the knowledge base. scripts/

qtest-preprocessor.py
Enriches raw Test staging files with semantic metadata (intent, layer, risk level, behavior type) and AI-based module classification. Processes tc-* md files in-place, adding semantic context sections and updating frontmatter for KB import readiness.

scripts/validate-test-cases.py
Validates test case quality using an 8-dimension scoring rubric (assertion quality, density, clarity, traceability, etc.). Detects duplicates, identifies coverage gaps, and generates validation reports with pass/fail scores against an 80/100 threshold.