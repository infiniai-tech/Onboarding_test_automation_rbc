# Enterprise Architecture Recommendations

## AI Automation Framework — chu0-EntityStructure / Engineering Forge

**Author:** Principal Architect Review
**Date:** June 30, 2026
**Scope:** Scalability · Reusability · Token Optimization
**Classification:** RBC Internal — Clear Technology Program

---

## Executive Summary

The current system is two tightly coupled layers:

| Layer | What it is | Current Maturity |
|---|---|---|
| **Engineering Forge Toolkit** | AI skills/agents/workflows in `.github/` | Proof-of-Concept |
| **Java QA Automation Framework** | REST Assured + TestNG test harness | Production-grade single-service |

Both layers have significant architectural debt that will limit enterprise adoption. The recommendations below are prioritized by impact and implementation cost.

---

## 1. Critical Issues (Must Fix Before Enterprise Rollout)

### 1.1 God Class Anti-Pattern — `CommonFunctions`

**Current state:** A single 500+ line utility class mixes: proxy config, email, JSON mutation, Selenium screenshots, date utilities, and HTTP sending.

**Risk:** Any change breaks unrelated tests. Teams can't reuse individual capabilities without the full class.

**Recommendation:**

```
common/libraries/
├── ProxyUtils.java          # Proxy config only
├── JsonPathUtils.java       # JSONPath CRUD operations
├── DateTimeUtils.java       # Date/time helpers (timezone-aware)
├── EmailNotifier.java       # Email via Jenkins — async
├── ScreenCaptureUtils.java  # UI screenshots (deprecated marker)
└── CommonFunctions.java     # @Deprecated — delegates to above
```

**Migration:** Add `@Deprecated` to `CommonFunctions` immediately. Create targeted classes. Keep delegate methods for 2-sprint backward compatibility window.

---

### 1.2 Deep Inheritance Chain — Violates Composition-over-Inheritance

**Current state:**

```
ITestDataHandler → baseTestNgGlobal → baseTestNg → TestBaseApi → BaseTest → [...]
```

6 levels deep. Any change to `baseTestNgGlobal` risks all 3 test classes simultaneously.

**Risk:** Adding a new service (e.g., Document Management) requires pulling the entire FCRU inheritance chain.

**Recommendation — Composition via Injectable Capabilities:**

```java
// Instead of inheritance, inject capabilities
public class CreateRelationshipTests {

    @Inject private OAuthTokenManager tokenManager;
    @Inject private AllureReporter reporter;
    @Inject private QTestPublisher qTestPublisher;
    @Inject private TestDataLoader testDataLoader;
    @Inject private RequestResponseLogger logger;

    // Pure test logic — no framework concerns
}
```

Use TestNG's `@Factory` + Guice dependency injection (already a common TestNG pattern). Each capability is independently testable and swappable.

---

### 1.3 `OAuthTokenManager` Uses `curl` Subprocess

**Current state:** Shells out to `curl` for every Okta token request. Thread-safe via `synchronized` block.

**Risks:**
- `curl` path differences between CI (Linux) and local (Windows) cause flaky failures
- `synchronized` creates a bottleneck under parallel test execution
- Base64 credentials stored in `.properties` files — violates RBC secrets management policy

**Recommendation:**

```java
@Component
public class OktaTokenProvider implements TokenProvider {

    private final OkHttpClient httpClient;           // Already a project dependency
    private final Cache<String, String> tokenCache;  // Caffeine or Guava cache
    private final SecretResolver secretResolver;     // Azure Key Vault / SAIF

    // Replace curl subprocess with OkHttpClient POST
    // Replace synchronized with Caffeine's async refresh-after-write
    // Replace .properties credentials with vault-resolved secrets at startup
}
```

**Benefit:** Eliminates OS-specific subprocess. Proper async token refresh. Aligns with RBC Key Vault secret governance.

---

## 2. Scalability Recommendations

### 2.1 Replace Excel Test Data with Structured YAML/JSON

**Current state:** `EntitlementDataOAT.xlsx` loaded into `HashMap<String, HashMap<String, String>>` — no type safety, merge conflicts in Git, breaks on column rename.

**Scale problem:** When the framework covers 20 services, you'll have 20 Excel files, each unversionable.

**Recommendation:**

```yaml
# src/test/resources/testdata/USCM-95867/create-relationship.yaml
testCases:
  - id: TC-001
    description: "Create with mandatory fields only"
    input:
      relationshipName: "{{faker.company}}"
      relationshipType: "Relationship type"
      relationshipStatus: "status"
      createdBy: "{{faker.email}}"
      createdDateTimeStamp: "{{now.iso}}"
    expected:
      statusCode: 200
      relationshipId:
        pattern: "^REL-"
    qtest:
      path: "USCM S26.2.5 (1Apr-14Apr)/USCM-95867/Execution"
      tcName: "Create Relationship - Mandatory Fields"
```

**Benefits:**
- Git-diffable, reviewable, merge-safe
- Type-safe deserialization via Jackson
- `{{faker.x}}` expressions resolved at runtime by `TestDataResolver`
- qTest metadata co-located with test data (eliminates Excel-qTest coupling)

---

### 2.2 Parallel Test Execution Architecture

**Current state:** TestNG suite is single-threaded. `synchronized` in `OAuthTokenManager` would serialize all threads even if parallelism were enabled.

**Recommendation — Parallel-Safe Design:**

```xml
<!-- TestNG_EntityStructure.xml -->
<suite name="Entity Structure Test Suite" parallel="methods" thread-count="4">
  <test name="Create Relationship Tests">
    <classes>
      <class name="api.tests.FCRUServiceTest.CreateRelationshipTests"/>
    </classes>
  </test>
</suite>
```

**Dependencies for parallelism:**
1. `OAuthTokenManager` → replace `synchronized` with Caffeine async cache (non-blocking)
2. `RequestResponseLogger` → already uses `ThreadLocal` ✅
3. `ITestDataHandler` step/skip tracking → already uses `ThreadLocal` ✅
4. `qTestUtil` → make result publishing async (queue + batch flush in `@AfterSuite`)

**Expected gain:** 4x execution speed for 50+ test suite. Critical for CI feedback loops.

---

### 2.3 Multi-Service Test Framework Architecture

**Current state:** Framework is hardwired to FCRU. Package `api.clients.FCRU`, `api.data.FCRU`, `api.tests.FCRUServiceTest`.

**Scale problem:** Adding Document Management or Party services requires copying the entire structure.

**Recommendation — Service-Agnostic Core + Service Modules:**

```
src/
├── main/java/
│   ├── core/                           # Service-agnostic framework
│   │   ├── client/BaseApiClient.java   # Generic REST Assured wrapper
│   │   ├── auth/TokenProvider.java     # Auth interface
│   │   ├── data/TestDataLoader.java    # YAML/JSON data loader
│   │   ├── reporting/AllureReporter    # Allure helpers
│   │   └── qtest/QTestPublisher.java   # qTest integration
│   └── services/
│       ├── fcru/                       # FCRU-specific
│       │   ├── FcruClient.java
│       │   ├── FcruFactory.java
│       │   └── model/
│       └── document/                   # Future service
│           ├── DocumentClient.java
│           └── model/
└── test/java/
    └── services/
        ├── fcru/CreateRelationshipTests.java
        └── document/                    # Future tests
```

`BaseApiClient` absorbs the 5 HTTP verbs, Allure steps, and logging.
`FcruClient` only knows FCRU endpoints and payloads.

---

### 2.4 Engineering Forge — Skill Registry

**Current state:** Skills are discovered by reading `SKILLS_MAP.md`. No versioning, no dependency declaration between skills, no capability namespacing.

**Scale problem:** When 10 teams contribute skills, naming collisions and stale skill references break agent workflows silently.

**Recommendation — Skill Manifest Standard:**

```yaml
# .github/skills/qa/automate/skill.manifest.yaml
id: qa-automate
version: 3.2.1
category: qa
description: "Generate Java REST Assured + Playwright automation"
requires:
  - skill: qa-e2e-test-design@>=2.0.0
  - skill: spring-patterns@>=1.0.0
context_tokens: 4200          # Estimated tokens when loaded
lazy_sections:
  - id: playwright-ui
    trigger_keywords: ["UI test", "Playwright", "frontend"]
    tokens: 1800
  - id: rest-assured-api
    trigger_keywords: ["API test", "REST Assured", "backend"]
    tokens: 1200
inputs:
  required: [story_id, test_cases_path]
  optional: [base_package, auth_type]
outputs: [automation_files, tc_updates]
```

This manifest enables:
- **Semantic versioning** → agents pin skill versions
- **Lazy section loading** → only load Playwright section when needed (see §3)
- **Token budget declaration** → orchestrator can plan context allocation
- **Dependency graph** → detect circular or missing skill dependencies

---

## 3. Token Optimization Recommendations

### 3.1 Tiered Context Loading (Highest ROI)

**Current state:** Every AI interaction loads:
- Full `copilot-instructions.md`
- Full `SKILLS_MAP.md` (42 lines, all categories)
- Full skill SKILL.md file (avg ~200–500 lines)
- Full `PROJECT_STRUCTURE.md` (877 lines)

**Token cost per average interaction: ~8,000–12,000 tokens just for context setup.**

**Recommendation — Three-Tier Loading Strategy:**

```
Tier 0 — Always Loaded (< 500 tokens)
  copilot-instructions.md    ← Keep minimal: project ID, stack, contact
  SKILLS_MAP.md              ← Table only, no descriptions

Tier 1 — Role-Triggered (~1,500 tokens)
  Loaded when: user role matches (QA / Dev / PMO)
  qa-agent-context.md        ← QA-specific patterns, qTest IDs, team config
  dev-agent-context.md       ← Spring patterns, DB schema, API contracts

Tier 2 — Task-Triggered (~2,000–4,000 tokens)
  Loaded when: specific skill invoked
  skills/qa/automate/SKILL.md    ← Only when generating automation
  skills/spring/patterns/        ← Only when implementing backend
```

**Implementation:**

```
<!-- .github/copilot-instructions.md — REVISED -->
# RBC Engineering Forge — chu0-EntityStructure

Stack: Java 21 · Spring Boot · TestNG · REST Assured · React · TypeScript
Team: Clear Technology / USCM Entity Structure
qTest Project: 718 | Jira Board: 9039

## Context Loading Rules
Load tier-1 context based on task type:
- QA task  → read `.github/context/qa-context.md`
- Dev task → read `.github/context/dev-context.md`
- PMO task → read `.github/context/pmo-context.md`

## Skills
See `.github/SKILLS_MAP.md` for available skills.
```

**Estimated token savings:** 40–60% reduction in context tokens per session.

---

### 3.2 Skill Sectioning with Lazy Loading

**Current state:** Skills like `qa-automate/SKILL.md` load everything: REST Assured patterns, Playwright patterns, TC eligibility rules, file naming conventions — regardless of whether the task is API or UI.

**Recommendation — Section Markers:**

```
<!-- .github/skills/qa/automate/SKILL.md -->

## [ALWAYS] Eligibility Assessment
<!-- Always loaded — 300 tokens -->
Rules for determining if a TC is automatable...

## [LAZY:api] REST Assured Generation
<!-- Load when: task mentions API, REST, backend — 800 tokens -->
### File Naming Convention
### Class Structure
### Authentication Pattern
...

## [LAZY:ui] Playwright Generation
<!-- Load when: task mentions UI, browser, Playwright — 900 tokens -->
### Page Object Pattern
### Selector Strategy
...
```

The AI agent reads the `[ALWAYS]` sections first, then selectively fetches `[LAZY:*]` sections based on the task's keywords. This requires a lightweight context-fetch protocol in the agent loop.

---

### 3.3 Compressed Knowledge Artifacts

**Current state:** `PROJECT_STRUCTURE.md` is 877 lines. It's loaded as context for almost every task. Most interactions need only 10% of it.

**Recommendation — Indexed Summary + Detail Split:**

```
context/
├── PROJECT_STRUCTURE.md        # Full detail — load only when researching
├── PROJECT_SUMMARY.md          # 50-line compressed summary — always loadable
└── api-contracts/
    ├── fcru-endpoints.yaml     # OpenAPI snippet — load when writing API tests
    └── fcru-error-codes.yaml   # Error codes only — load when writing assertions
```

`PROJECT_SUMMARY.md` template (target: < 600 tokens):

```
# Project: chu0-EntityStructure
Service: FCRU (Flex Client Relationship Utility) | App: CHU0 | Env: QAT
Stack: Java 21, TestNG 7.10, REST Assured 5.4, Allure 2.30
Auth: Okta M2M via OAuthTokenManager (cached, 55min TTL)
Base URL: https://origin-svc-chu0-private-qat.westus2.aks.nonp.c1.rbc.com
Key Endpoints: POST/PUT/GET /origin/relationship, GET /origin/taxonomy
Test Classes: CreateRelationshipTests, UpdateRelationshipTest, TaxonomyTests
CI: Nightly GH Actions → Allure GitHub Pages + qTest Project 718
Known Gaps: GET /{id} rejects 22-char IDs (500), x-request-id not enforced
Full detail: context/PROJECT_STRUCTURE.md
```

---

### 3.4 Agent State Persistence (Cross-Session Token Savings)

**Current state:** Every new agent session rebuilds all context from scratch. The `qa-story` agent re-reads Jira, re-reads qTest config, re-reads team config on every invocation.

**Recommendation — Session State Files:**

```
context/qa/USCM-99314/
├── story-analysis.json       # Jira story data — fetched once, reused
├── test-cases.md             # Approved TCs — locked after review
├── automation-status.json    # Which TCs have been automated
└── session.state.json        # Agent phase tracker
```

```json
// session.state.json
{
  "story": "USCM-99314",
  "phase": "automation",
  "completed_phases": ["analysis", "test-design", "qtest-push"],
  "automation": {
    "eligible_tcs": ["TC-001", "TC-002", "TC-003"],
    "generated": ["TC-001"],
    "pending": ["TC-002", "TC-003"]
  },
  "last_updated": "2026-06-30T14:23:00Z"
}
```

**Token savings:** Resuming a mid-story session costs ~500 tokens vs ~4,000 tokens to reconstruct from scratch. 8x reduction for multi-session stories.

---

### 3.5 Prompt Template Deduplication

**Current state:** Skills like `qa-automate`, `qa-e2e-test-design`, and `qa-story-analysis` each embed near-identical boilerplate: project context, auth patterns, qTest API endpoints, file naming rules.

**Recommendation — Shared Prompt Fragments:**

```
.github/prompts/
├── fragments/
│   ├── rbc-project-context.md    # 80 tokens — project identity
│   ├── qtest-api-reference.md    # 120 tokens — qTest REST API calls
│   ├── java-test-conventions.md  # 100 tokens — naming, annotations
│   └── okta-auth-pattern.md      # 60 tokens — how to get token
└── skills/
    └── qa/automate/SKILL.md
        # Instead of embedding auth pattern:
        # {{include: fragments/okta-auth-pattern.md}}
```

---

## 4. Compliance & Security Architecture

### 4.1 Secret Governance — Critical Gap

**Current state:**
- `okta.client.id.encoded` and `okta.client.secret.encoded` stored in `qat.api.properties` (base64, not encrypted)
- `BearerToken` for qTest stored in `qtest.properties`
- `saifgServiceIdSecret` in `config.properties`

**RBC Policy Violation:** Credentials — even encoded — must not reside in Git repositories.

**Recommendation:**

```yaml
# GitHub Actions workflow — secret injection
- name: Run Tests
  env:
    OKTA_CLIENT_ID: ${{ secrets.CHU0_OKTA_CLIENT_ID }}
    OKTA_CLIENT_SECRET: ${{ secrets.CHU0_OKTA_CLIENT_SECRET }}
    QTEST_BEARER_TOKEN: ${{ secrets.CHU0_QTEST_TOKEN }}
    COSMOS_DB_URI: ${{ secrets.CHU0_COSMOS_URI }}
```

```java
// OAuthTokenManager — read from environment
String clientId = System.getenv("OKTA_CLIENT_ID");
String clientSecret = System.getenv("OKTA_CLIENT_SECRET");
```

For local dev: Use Azure Key Vault CLI (`az keyvault secret show`) or a `.env` file excluded by `.gitignore`.

---

### 4.2 AI Skill Security Boundaries

**Current state:** Skills have access to execute arbitrary terminal commands (via `run_in_terminal` tool). No skill declares what tools it's allowed to use.

**Recommendation — Skill Permission Manifest:**

```yaml
# skill.manifest.yaml — permissions section
permissions:
  allowed_tools:
    - read_file
    - grep_search
    - run_in_terminal
  restricted_tools:
    - create_file       # Require human approval before writing
  requires_approval:
    - patterns: ["git push", "mvn deploy", "curl.*POST"]  # Regex on commands
```

---

## 5. Reusability Recommendations

### 5.1 Skill Promotion Pipeline

**Current state:** Skills are written and consumed within a single repo. No mechanism to promote a proven skill to the enterprise forge.

**Recommendation — Promotion Workflow:**

```
Local Repo Skill → Team Review → Forge PR     → Enterprise Registry
   (private)        (2 approvers)  (lint + test)   (semver tagged)
```

**Criteria for enterprise promotion:**
- No service-specific hardcoding (no `FCRU`, `uscm`, `CHU0` references)
- Works with `{{config.serviceUrl}}` variables
- Has an `examples/` directory with 2+ real usage examples
- Token budget declared in manifest

---

### 5.2 Test Framework — Adapter Pattern for Service Onboarding

**Current state:** Adding a new service requires understanding 6 inheritance levels and copying `FCRU`-specific patterns.

**Recommendation — Service Adapter Interface:**

```java
public interface ServiceAdapter {
    String getBaseUri();
    RequestSpecification buildRequestSpec(String token);
    Class<?> getModelClass();
    String getQTestPath();
}

public class FcruAdapter implements ServiceAdapter {
    @Override
    public String getBaseUri() {
        return FCRUEndpoints.FCRU_SERVICE_URL;
    }
    // ...
}

// New service in 30 minutes:
public class DocumentAdapter implements ServiceAdapter {
    @Override
    public String getBaseUri() {
        return System.getProperty("document.service.url");
    }
    // ...
}
```

---

### 5.3 Factory Pattern Standardization

**Current state:** `RelationshipFactory` uses static methods and hardcoded string constants (`"Relationship type"`, `"status"`, `"uscm"`). These values are business domain data that should live in config.

**Recommendation:**

```java
@Component
public class RelationshipFactory extends AbstractPayloadFactory<Relationship> {

    private final TaxonomyConfig taxonomy;  // Loaded from taxonomy API or config

    @Override
    public Relationship validMinimal() {
        return Relationship.builder()
            .tenant(taxonomy.getDefaultTenant())
            .relationshipName(faker.company().name())
            .relationshipType(taxonomy.getDefaultType())
            .relationshipStatus(taxonomy.getDefaultStatus())
            .createdBy(faker.internet().emailAddress())
            .createdDateTimeStamp(DateTimeUtils.nowIso())
            .build();
    }

    @Override
    public Relationship withOverrides(Map<String, Object> overrides) {
        // Generic override application via reflection or builder
    }
}
```

---

## 6. Implementation Roadmap

| Priority | Change | Effort | Risk |
|---|---|---|---|
| 🔴 P0 | Move secrets to GH Actions secrets / env vars | 0.5 sprint | Low |
| 🔴 P0 | Replace Excel with YAML test data | 1 sprint | Medium |
| 🟠 P1 | Replace `curl` subprocess with OkHttp token fetch | 0.5 sprint | Low |
| 🟠 P1 | Split `CommonFunctions` into targeted utilities | 1 sprint | Low |
| 🟠 P1 | Implement tiered context loading (Tier 0/1/2) | 0.5 sprint | Low |
| 🟡 P2 | Add session state persistence for agents | 1 sprint | Low |
| 🟡 P2 | Add `skill.manifest.yaml` with token budgets | 1 sprint | Low |
| 🟡 P2 | Restructure to `core/` + `services/` module layout | 2 sprints | Medium |
| 🟢 P3 | Skill promotion pipeline + enterprise registry | 2 sprints | High |
| 🟢 P3 | Parallel test execution (4 threads) | 1 sprint | Medium |

---

## 7. Metrics to Track Post-Implementation

| Metric | Current Baseline | Target |
|---|---|---|
| Avg tokens per AI session | ~10,000 | < 5,000 (50% ↓) |
| Time to add new service | 2 days | 2 hours |
| CI test execution time | ~8 min (single thread) | ~3 min (parallel) |
| Secrets in source code | 6 credential keys | 0 |
| Skill reuse across teams | 0 (single repo) | 3+ teams |
| Test suite flakiness rate | Unknown | < 2% |

---

*Review this document with Tech Lead and PO before sprint planning. P0 items should block enterprise onboarding.*
