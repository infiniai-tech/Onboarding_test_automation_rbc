# New Entity Automation — Analysis & Baseline

> **Purpose:** Before automating a new entity, we analysed the two existing implementations
> (FCRU/Orchestration and PRT0) to understand what each got right and wrong.
> This file records that analysis and defines the **target baseline** the new entity automation should follow.
>
> Reference for framework fundamentals: [Entity_Structure_Test_Automation_Onboarding.md](Entity_Structure_Test_Automation_Onboarding.md)

---

## The Simple Version

PRT0 was built **later and cleaner**. It learned from the mistakes of FCRU/Orchestration and fixed several of them — but it also introduced its own new problems.

**Neither is a clear winner.** They each solved different problems while introducing new ones. The new entity should cherry-pick the best decisions from both.

---

## Side-by-Side Comparison: FCRU/Orchestration vs PRT0

### 1. Authentication (Token Fetching)

|  | FCRU / Orchestration | PRT0 |
|---|---|---|
| **How** | Spawns a `curl` subprocess via `ProcessBuilder` | Uses REST Assured's own HTTP — stays in-process |
| **Secret safety** | ❌ Secret visible in OS process args (`ps aux`) | ✅ Secret stays inside JVM memory |
| **Expiry** | ❌ Hardcoded 55 mins — ignores actual `expires_in` | ✅ Reads actual `expires_in` from token response |
| **Lock type** | `synchronized(LOCK)` — basic | `ReentrantLock` — more flexible |
| **Auth flow** | Single step (Okta M2M) | Two-step for QAT (PingFed → Apigee → Okta); single step for CIT |

> **Winner: PRT0.** Token fetching is done the right way — in-process HTTP, actual expiry used, no OS process spawning.

---

### 2. Spec Class Design

|  | FCRU / Orchestration | PRT0 |
|---|---|---|
| **Spec class parent** | ❌ `FCRUSpecifications` extends `baseTestNg` (inherits test lifecycle) | ✅ `PRTOSpecifications` — no parent class at all |
| **Side effects** | Spec creation can trigger test lifecycle hooks | None — pure utility class |

> **Winner: PRT0.** The spec class is standalone, as it should be. A Specifications class is a utility — it should never know about or depend on the test runner lifecycle.

---

### 3. Inheritance Depth (Test Classes)

|  | FCRU / Orchestration | PRT0 |
|---|---|---|
| **Chain** | `baseTestNgGlobal` → `baseTestNg` → `TestBaseApi` → `BaseTest` → `TestClass` | `baseTestNgGlobal` → `baseTestNg` → `TestBaseApi` → `BaseTest` → `BasePRT0Test` → `TestClass` |
| **Depth** | 4 levels — already deep | 5 levels — even deeper |

> **Winner: Neither.** PRT0 made this **worse** by adding `BasePRT0Test` on top of the already-4-deep chain. The new entity should use 4 levels max and avoid adding another intermediate base class unless strictly necessary.

---

### 4. Test Data Source

|  | FCRU / Orchestration | PRT0 |
|---|---|---|
| **Source** | JavaFaker — generated at runtime dynamically | Excel file (`datafile_qa.xlsx`) — loaded from disk |
| **Flexibility** | ✅ No file dependency; works anywhere | ❌ Excel file must exist on the runner; environment-specific sheet names hardcoded (`DDA_Activation_qa`) |
| **Maintenance** | Update the factory class | Update spreadsheet + re-commit |

> **Winner: FCRU/Orchestration.** Code-based factories are more portable than Excel spreadsheets in CI. Excel files introduce a fragile file dependency and hardcoded environment names.

---

### 5. Payload Construction (in Tests)

|  | FCRU / Orchestration | PRT0 |
|---|---|---|
| **Style** | Typed model classes (`Relationship`, `OperatorInfo`) via Lombok builders | Raw `Map<String, Object>` — untyped `HashMap` |
| **Safety** | ✅ Compile-time field names | ❌ Typo in a map key = silent test failure |
| **Readability** | ✅ Clear what fields exist | ❌ Must grep to find what keys are valid |

**Code comparison:**
```java
// FCRU — typed, compiler-checked:
Relationship.builder()
    .relationshipName("RELAuto123")
    .relationshipType("Accessing Party")
    .build();

// PRT0 — untyped, error-prone:
payload.put("accountId", "TESTACCT-...");  // typo here = silent failure
payload.put("produtCode", "IB");           // compiler won't catch this
```

> **Winner: FCRU/Orchestration.** Typed Lombok models prevent silent typo failures that are extremely hard to debug at runtime.

---

### 6. Base URL Resolution

|  | FCRU / Orchestration | PRT0 |
|---|---|---|
| **How** | Owner library (`FCRUEndpointsConfig` via `ConfigFactory`) | Manual `Properties` file reading + `if/else` env mapping |
| **Env mapping** | Declarative config interface | `"qat".equalsIgnoreCase(env) ? "qa" : env` — manual string logic |
| **Duplication** | Resolved once in Endpoints class | `resolveEnv()` copied in 3 places: `BasePRT0Test`, `PRT0TokenManager`, `PRT0Specifications` |

> **Winner: FCRU/Orchestration.** PRT0 has internal duplication — the same `resolveEnv()` logic is copy-pasted across 3 classes. The FCRU declarative config approach resolves env once and shares it everywhere.

---

### 7. Secrets Management

|  | FCRU / Orchestration | PRT0 |
|---|---|---|
| **Secrets location** | ✅ Env vars only (`OKTA_CLIENT_SECRET`) — enforced by `resolveCredential()` | ❌ Credentials read directly from `prt0-config.properties` file (`client_secret_pingFed`, `cit.client.secret`) |
| **Risk** | Secret never in any config file | Secret can accidentally be committed to source control |

> **Winner: FCRU/Orchestration.** PRT0 reads credentials from a config file, which risks accidental secret commits. Secrets must live in environment variables only.

---

### 8. Retry Logic

|  | FCRU / Orchestration | PRT0 |
|---|---|---|
| **Retry utility** | ✅ `RetryUtils.execute()` — centralized, reusable | ❌ No retry mechanism |
| **Usage** | `OrchestrationClient` uses it for flaky network calls | All PRT0 calls are fire-and-forget |

> **Winner: FCRU/Orchestration.** Network calls to banking APIs can be flaky. PRT0 has zero tolerance for transient failures, making tests brittle in CI.

---

## Summary Scorecard

| Area | FCRU/Orch | PRT0 | Winner |
|---|---|---|---|
| Token fetching | ❌ `curl` subprocess | ✅ In-process HTTP | **PRT0** |
| Token expiry | ❌ Hardcoded | ✅ Uses `expires_in` | **PRT0** |
| Spec class design | ❌ Extends test class | ✅ Standalone class | **PRT0** |
| Typed payloads | ✅ Lombok models | ❌ Raw `HashMap` | **FCRU** |
| Test data | ✅ Code-based factory | ❌ Excel file | **FCRU** |
| Secrets handling | ✅ Env vars only | ❌ Config file | **FCRU** |
| Retry logic | ✅ `RetryUtils` | ❌ None | **FCRU** |
| Inheritance depth | ❌ 4 levels (deep) | ❌❌ 5 levels (deeper) | **Neither** |
| `resolveEnv()` duplication | ❌ Present | ❌❌ Worse (3 copies) | **Neither** |

**Bottom line:** PRT0 fixed the token architecture but regressed on payload safety, secrets management, and code duplication. Neither is a clear winner.

---

## The Ideal Baseline for the New Entity

Take the **best decision from each side** for every area:

| Area | Decision for New Entity | Rationale |
|---|---|---|
| **Token fetching** | Use in-process HTTP (PRT0 style) | No OS process spawning; secret stays in JVM |
| **Token expiry** | Read `expires_in` from token response (PRT0 style) | Dynamic — won't break when Okta changes TTL |
| **Spec class design** | Standalone class, no parent (PRT0 style) | No accidental lifecycle coupling |
| **Typed payloads** | Lombok `@Builder` models (FCRU style) | Compiler catches field typos |
| **Test data** | JavaFaker code-based factory (FCRU style) | No file dependency; CI-portable |
| **Secrets handling** | Env vars only via `resolveCredential()` (FCRU style) | Zero risk of secret commit |
| **Retry logic** | `RetryUtils.execute()` (FCRU style) | Resilient to transient network failures |
| **Inheritance depth** | Max 4 levels — do NOT add a new base class | Both existing approaches are already too deep |
| **URL resolution** | `ConfigFactory` + declarative interface (FCRU style) | Single source of truth; no copy-pasted `resolveEnv()` |

---

## What This Means When Building the New Entity

### DO — Copy from FCRU/Orchestration

- Typed model classes with Lombok `@Data @Builder @NoArgsConstructor @AllArgsConstructor`
- JavaFaker factory that generates alphanumeric-only data
- Secrets come from environment variables enforced by a `resolveCredential()` method
- `RetryUtils.execute()` wrapping client calls that touch external services
- `ConfigFactory`-based endpoints config interface for URL resolution
- URL path constants in a dedicated `XxxEndpoints.java` file

### DO — Copy from PRT0

- Token manager that uses in-process REST Assured HTTP (not `ProcessBuilder`/`curl`)
- Token manager that reads `expires_in` field from the token response for cache TTL
- Specifications class that has **no parent class** — pure static utility

### DO NOT — Repeat from either

- Do not spawn `curl` subprocesses for token fetching
- Do not hardcode token TTL (55 min or any fixed number)
- Do not extend `baseTestNg` in your Specifications class
- Do not use raw `HashMap` for request payloads
- Do not load test data from Excel files for new test coverage
- Do not put client secrets in any `.properties` file
- Do not copy-paste `resolveEnv()` across multiple classes
- Do not add a fifth inheritance level (no new `BaseXxxTest` class)

---

## New Entity — Recommended File Checklist

Following the 9-step process from the onboarding guide, the new entity needs:

```
src/main/java/api/<NewEntity>/
├── clients/
│   └── <NewEntity>Client.java          ← One method per API call; no HTTP details in tests
├── specs/
│   └── <NewEntity>Specifications.java  ← Standalone (no parent); auth + headers + response validators
├── data/
│   ├── endpoints/
│   │   ├── <NewEntity>EndpointsConfig.java  ← ConfigFactory interface; reads from .properties
│   │   └── <NewEntity>Endpoints.java        ← URL path constants only
│   └── factory/
│       └── <NewEntity>Factory.java          ← JavaFaker; alphanumeric-only generated values
├── enums/
│   └── <NewEntity>Values.java          ← Domain constants (status strings, type strings)
└── models/
    └── <NewEntity>Model.java           ← Lombok @Builder POJO; one per request/response shape

src/test/java/api/tests/<NewEntity>/
└── <NewEntity>Tests.java               ← Extends BaseTest; uses client directly

src/test/resources/
└── TestNG_EntityStructure.xml          ← Add <class name="api.tests.<NewEntity>.<NewEntity>Tests"/>
                                           ⚠️ Without this, the tests NEVER run in CI
```

---

## Token Manager Template (Best Practice)

The new entity's token manager should follow the PRT0 pattern (in-process HTTP, dynamic expiry), not the FCRU pattern (curl subprocess, hardcoded TTL):

```java
// GOOD — PRT0 style (what the new entity should use)
private static void fetchToken() {
    Response response = RestAssured.given()
        .contentType("application/x-www-form-urlencoded")
        .formParam("grant_type", "client_credentials")
        .formParam("client_id", clientId)
        .formParam("client_secret", resolveCredential("NEW_ENTITY_CLIENT_SECRET"))
        .post(tokenUrl);

    cachedToken = response.jsonPath().getString("access_token");
    int expiresIn = response.jsonPath().getInt("expires_in");  // read actual value
    expiryTime = System.currentTimeMillis() + (expiresIn - 300) * 1000L; // refresh 5 min early
}

// BAD — FCRU style (do not repeat)
ProcessBuilder pb = new ProcessBuilder("curl", "-d",
    "client_secret=" + secret,   // secret visible in ps aux!
    tokenUrl);
```

---

## Secrets Template (Best Practice)

```java
// GOOD — FCRU style enforced pattern
private static String resolveCredential(String envVarName) {
    String value = System.getenv(envVarName);
    if (value == null || value.isBlank()) {
        throw new IllegalStateException(
            "Required env var '" + envVarName + "' is not set. " +
            "Set it in your .env file locally or as a GitHub Secret in CI.");
    }
    return value;
}

// BAD — PRT0 style (never do this)
// Reading from prt0-config.properties:
// client_secret_pingFed=abc123    ← can be committed to git by accident
```
