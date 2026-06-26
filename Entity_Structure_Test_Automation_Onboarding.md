# Entity Structure Test Automation 


## What Does This Repository Do?

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
subgraph Repo["This Repository (Test Automation Framework)"]
TC["Test Class"]
REST["REST Assured Client"]
ASSERT["Assertions"]
TC-->REST-->ASSERT
end

subgraph OKTA["OKTA OAuth"]
TOKEN["Token Service"]
end

subgraph API["FCRU API Service"]
REL["/origin/relationship\nPOST\nPUT\nGET"]
TAX["/origin/taxonomy\nGET"]
DB["Azure K8S\nCosmos DB"]
end

ALLURE["Allure Report"]
QTEST["qTest"]
GH["GitHub Actions"]

TC--Get Token-->TOKEN
TOKEN--Bearer Token-->REST
REST--API Call-->REL
REL--JSON-->REST
ASSERT-->ALLURE-->QTEST
ASSERT-->GH
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
subgraph Phase1["Phase 1: Test Data Preparation"]
EXCEL["EntitlementDataQAT.xlsx"]
FACTORY["RelationshipFactory.java"]
MAP["Test Data Map"]
EXCEL-->MAP
FACTORY-->MAP
end

subgraph Phase2["Phase 2: Authentication"]
AUTH["OAuthTokenManager"]
OKTA["OKTA Server"]
AUTH-->OKTA
OKTA-->AUTH
end

subgraph Phase3["Phase 3: API Call"]
REST["REST Assured"]
API["FCRU API"]
REST-->API
API-->REST
end

subgraph Phase4["Phase 4: Validation"]
ASSERT["Assertions"]
end

subgraph Phase5["Phase 5: Reporting"]
ALLURE["Allure"]
QTEST["qTest"]
CONSOLE["Console"]
end

MAP-->AUTH-->REST-->ASSERT
ASSERT-->ALLURE
ASSERT-->QTEST
ASSERT-->CONSOLE
```

---

## How Tests Run Automatically

```mermaid
flowchart TD
TRIGGER["Nightly Schedule or Manual Trigger"]
JOB1["Run API Tests
- Checkout
- Java 21
- Inject Secrets
- mvn test
- Upload Results"]
JOB2["Deploy Allure Report
- Download Results
- Generate Report
- Deploy GitHub Pages"]
RESULT["Published Allure Report"]

TRIGGER-->JOB1-->JOB2-->RESULT
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
