---
description: Run Checkstyle lint check on order-service, analyze findings, and suggest remediations
---

Execute the following steps to perform a Checkstyle lint check and provide a structured analysis:

1. **Run Checkstyle:**
   Execute `mvn -f order-service/pom.xml checkstyle:check`. Note that Checkstyle may fail the Maven build if violations exist, which is expected.

2. **Read Results:**
   Read and parse the Checkstyle XML report at `order-service/target/checkstyle-result.xml`.

3. **Analyze and Report Findings:**
   - **Summary by Severity:** Group all detected Checkstyle violations by severity level (`error`, `warning`, `info`), showing the count and affected files.
   - **Fix-First Items:** Highlight the highest-priority issues that should be addressed first (e.g., build-breaking errors, critical style/correctness violations, or high-frequency rules).
   - **Concrete Remediations:** Provide actionable, specific remediation steps with exact file paths, line numbers, rule names, and code snippets or formatting fixes needed to resolve the issues.
