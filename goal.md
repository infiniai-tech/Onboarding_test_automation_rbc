Run qa-story agent for USCM-XXXXX"
PHASE 1 - Story Analysis (I fetch from Jira)
* Pull story title, description, As, subtasks, comments
* Fetch live Swagger from Qad to confirm endpoints exist
* Identify gaps (missing ACs, unclear fields, unknown error codes)
* Save: context/qa/USCM-XXXXX/story-analysis. md
YOU REVIEW - confirm gaps are resolved, story is ready
PHASE 2 - Test Case Design
* Map each AC to 1+ test cases (happy path + negatives + edge cases)
* Write step-by-step TC table (Action → Expected Result)
* Save: context/qa/USCM-XXXXX/USCM-XXXOXX-e2e-test-cases .md
YOU REVIEW - approve or adjust test cases
PHASE 3 - Push TCs to qTest
* Creates Sprint folder if it doesn't exist
* Creates Story folder inside sprint
* Pushes each TC with steps
* Links Jira requirement to all TCS
* Gets back TC IDs (e.g. TC-18825)
PHASE 4 - Generate Java Automation
* Creates src/test/java/api/tests/FCRU/YourNewTest. java
* Adds @TestCase annotation with the TC IDs from Test
* Registers class in TestNG_EntityStructure.xml
* Adds any missing factory methods / endpoints if needed
YOU RUN:
./mvnw
test -Dtest-YourNewTest -Denv=qat -DproxyServer-saifg --settings settings.xm]
PHASE 5 - HTML Report
•/mvnw allure:serve
-settings settings.xml
→ Opens browser with full visual report
* Each test shows: pass/fail, HTTP request, HTTP response, step breakdown + Results also pushed to Test automatically