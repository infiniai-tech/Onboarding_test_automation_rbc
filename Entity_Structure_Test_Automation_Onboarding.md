# Entity Structure Test Automation 


### The Simple Version

This is a **test automation project** that automatically checks whether
RBC's **Entity Structure** APIs behave correctly.

It:
1. Sends requests to the API.
2. Checks the responses.
3. Reports the results.

### What is "Entity Structure"?

The **Entity Structure** service manages relationships between financial
entities.

```mermaid
flowchart TD
    A["Company A (Parent Corp)"]
    B["Company B (Subsidiary)"]
    C["Company C (Partner)"]
    B1["Product: Commercial Loan"]
    C1["Product: Trade Finance"]
    ACC1["Account 1234"]
    ACC2["Account 5678"]

    A --> B
    A --> C
    B --> B1
    B --- ACC1
    C --> C1
    C --- ACC2
```

---

## The Big Picture -- Architecture Diagram

```mermaid
flowchart LR

    %% ==========================
    %% System Architecture
    %% ==========================

    subgraph REPO["THIS REPOSITORY<br/>(Test Automation Framework)"]
        TC["Test Class<br/>createRel..."]
        RA["REST Assured<br/>Client"]
        AS["Assertions<br/>Pass/Fail"]

        TC --> RA
        RA --> AS
    end

    subgraph OKTA["OKTA (OAuth)<br/>Identity Mgmt"]
        TOKEN["Token<br/>Service"]
    end

    subgraph API["FCRU API SERVICE<br/>(What we're testing)"]
        REL["/origin/relationship/<br/><br/>POST - Create<br/>PUT - Update<br/>GET - Read"]

        TAX["/origin/taxonomy<br/><br/>GET - Types"]

        INFO["Backend: Azure K8S<br/>Database: Cosmos DB"]
    end

    subgraph ALLURE["ALLURE REPORT<br/>(HTML Reports)"]
        AR["• Test Results<br/>• Screenshots<br/>• Request Logs<br/>• Trend Graphs"]
    end

    subgraph QTEST["qTest<br/>(Test Management)"]
        QT["• Stores test cases<br/>• Tracks executions<br/>• Links to requirements"]
    end

    subgraph GHA["GITHUB ACTIONS<br/>(CI/CD Pipeline)"]
        GH["• Nightly runs<br/>• Auto-reports<br/>• GitHub Pages"]
    end

    %% OAuth Flow
    TC -- "1. Get Token" --> TOKEN
    TOKEN -- "2. Return Bearer Token" --> TC

    %% API Flow
    RA -- "3. API Call with Token" --> REL
    REL -- "4. API Response (JSON)" --> RA

    %% Reporting
    AS --> AR
    AR --> QT

    %% CI/CD
    REPO -->|"Runs on"| GH
```

---

## Technology Stack Explained

### Core Languages & Build Tools

| Technology | Version | Purpose |
| --- | --- | --- |
| Java | 21 | Enterprise programming language |
| Maven | 3.x | Build & dependency management |

### Testing Framework

| Technology | Version | Purpose |
| --- | --- | --- |
| TestNG | 7.10.2 | Test runner |
| REST Assured | 5.4.0 | API testing |

### Reporting

| Technology | Version | Purpose |
| --- | --- | --- |
| Allure | 2.30.0 | HTML reports |

### Data Handling

| Technology | Version | Purpose |
| --- | --- | --- |
| Jackson | 2.18.6 | JSON processing |
| Lombok | 1.18.44 | Boilerplate reduction |
| JavaFaker | 1.0.2 | Fake data generation |
| Apache POI | 5.2.3 | Excel reader |

### Database

| Technology | Version | Purpose |
| --- | --- | --- |
| MongoDB Driver | 5.1.1 | Cosmos DB client |

### Infrastructure

- GitHub Actions
- GitHub Pages
- qTest

### Authentication

- Okta OAuth 2.0
- Corporate Proxy

---

## How Data Flows Through the System

### Test Execution Data Flow

```mermaid
flowchart TD

    %% ============================
    %% Phase 1
    %% ============================
    subgraph P1["Phase 1: Test Data Preparation"]

        EXCEL["Static Test Data<br/>(Excel / CSV / Config Files)"]

        GENERATED["Dynamic Test Data<br/>(Factories / Builders / Faker)"]

        MAP["Unified Test Data Store<br/><br/>Map&lt;TestCaseId, Field → Value&gt;<br/><br/>Example:<br/>• Relationship Name<br/>• Relationship Type<br/>• Expected Status"]

        EXCEL -->|"Read Test Data"| MAP
        GENERATED -->|"Generate Test Data"| MAP
    end

    %% ============================
    %% Phase 2
    %% ============================
    subgraph P2["Phase 2: Authentication"]

        TOKEN["Token Manager<br/><br/>• Checks cached token<br/>• Refreshes if expired"]

        AUTH["Identity Provider (OAuth)<br/><br/>• Validates credentials<br/>• Issues JWT Access Token"]

        TOKEN -->|"Authentication Request"| AUTH
        AUTH -->|"JWT Access Token"| TOKEN
    end

    %% ============================
    %% Phase 3
    %% ============================
    subgraph P3["Phase 3: API Execution"]

        CLIENT["REST Client<br/><br/>Request Specification<br/>• Base URL<br/>• Bearer Token<br/>• Headers<br/>• Request Body"]

        API["Target REST API<br/><br/>Processes request<br/>Returns JSON Response"]

        CLIENT -->|"HTTP Request"| API
        API -->|"JSON Response"| CLIENT
    end

    %% ============================
    %% Phase 4
    %% ============================
    subgraph P4["Phase 4: Validation"]

        ASSERT["Assertions<br/><br/>✓ Status Code<br/>✓ Response Fields<br/>✓ Business Rules<br/>✓ Required Values<br/><br/>PASS or FAIL"]

    end

    %% ============================
    %% Phase 5
    %% ============================
    subgraph P5["Phase 5: Reporting"]

        REPORT["Test Report<br/><br/>• HTML Reports<br/>• Execution History<br/>• Logs<br/>• Screenshots"]

        MGMT["Test Management Tool<br/><br/>• Test Results<br/>• Execution Status<br/>• Traceability"]

        CONSOLE["Console Output<br/><br/>• Logs<br/>• Errors<br/>• Summary"]

    end

    %% Flow
    MAP --> TOKEN
    TOKEN -->|"Bearer Token"| CLIENT
    CLIENT --> ASSERT
    ASSERT --> REPORT
    ASSERT --> MGMT
    ASSERT --> CONSOLE
```

---

## Summary -- The 60-Second Version

- Write Java automated tests.
- Authenticate with Okta.
- Call FCRU APIs.
- Validate responses.
- Publish Allure and qTest reports.
- Run nightly with GitHub Actions.

### Tech Stack

- Java 21
- Maven
- TestNG
- REST Assured
- Allure
- GitHub Actions

### API Under Test

- FCRU Entity Structure

### Database

- Cosmos DB (MongoDB compatible)

### Authentication

- Okta OAuth 2.0
