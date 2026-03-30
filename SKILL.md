---
name: sonarqube-sast-dast-correlation
description: Correlate SAST findings from SonarQube with DAST findings from SARIF files (StackHawk, ZAP, etc.) and generate a comprehensive security report
trigger: |
  User wants to correlate SAST and DAST findings, generate a security correlation report,
  or compare static and dynamic analysis results
---

# SonarQube SAST-DAST Correlation

Generate a comprehensive security correlation report that maps Static Application Security Testing (SAST) findings from SonarQube with Dynamic Application Security Testing (DAST) findings from SARIF files.

## Workflow Overview

This skill follows a structured workflow to correlate security findings:

1. **Check for Existing Report** - Look for existing `sast-dast-correlation-report.md` and offer to open it or rerun analysis
2. **Gather Configuration** - Auto-detect SonarQube credentials from environment variables or config files
3. **Retrieve SAST Issues** - Fetch or reuse SonarQube issues, filtering out imported DAST issues
4. **Retrieve DAST Issues** - Select and parse SARIF files from DAST tools (StackHawk, ZAP, etc.)
5. **Deep Correlation Analysis** - Use AI Agent to match SAST and DAST findings with source code verification
6. **Generate Report** - Create comprehensive markdown report with severity analysis and recommendations
7. **Open in Browser** - Automatically open the report for review
8. **Tag Issues (Optional)** - Tag correlated issues in SonarQube for team tracking

**📖 For detailed workflow steps, see [Workflow Steps](references/workflow-steps.md)**

## Correlation Analysis

The skill performs intelligent correlation using source code analysis to verify matches between SAST and DAST findings:

- **Reads actual source code** to extract endpoints, HTTP methods, and parameters
- **Validates vulnerability categories** match between SAST and DAST (SQL Injection ↔ SQL Injection)
- **Verifies endpoint mappings** from code annotations to DAST URLs
- **Traces data flow** from HTTP requests through controllers to vulnerable code
- **Assigns confidence levels** (HIGH/MEDIUM/LOW) based on correlation strength
- **Prevents false positives** by rejecting mismatched endpoints or vulnerability types

**📖 For detailed correlation rules and analysis process, see [Correlation Analysis](references/correlation-analysis.md)**

## Report Generation

Creates a comprehensive markdown report with:

- **Executive Summary** - Overview of SAST/DAST findings and correlations
- **Severity Distribution** - Breakdown by critical/high/medium/low severity
- **Correlated Findings** - HIGH PRIORITY vulnerabilities confirmed by both tools
- **SAST-Only Findings** - Code-level issues not detected at runtime
- **DAST-Only Findings** - Runtime issues invisible to static analysis
- **Coverage Analysis** - Complementary testing gaps identification
- **Actionable Recommendations** - Prioritized fix list with file/line references

**📖 For report template and structure, see [Report Template](references/report-template.md)**

## Tools Available

- **Read**: Read configuration files and issue data
- **Write**: Create the correlation report
- **Bash**: Execute SonarQube API calls with curl, filter SAST issues, and open the report in the default browser
- **Glob**: Find SARIF files
- **Grep**: Search for configuration values
- **AskUserQuestion**: Confirm configuration and file selections
- **Agent**: Perform deep correlation analysis (REQUIRED - use general-purpose agent)

## Error Handling

- If SonarQube API is unreachable, inform the user and offer to work with existing `sonar_issues.json`
- If no SARIF files are found, inform the user and ask for a file path
- Validate SARIF file format before processing
- Remember to filter out `external_StackHawk:` or similar imported issues from SAST data
- If correlation produces unexpectedly zero matches, verify that SAST and DAST scanned the same application (the most common reason is they scanned different apps or found completely different vulnerability categories)

## Output Files

- `sonar_issues.json` - SAST issues from SonarQube (if created)
- `correlations.json` - Correlation analysis results from Agent (intermediate file)
- `sast-dast-correlation-report.md` - Final correlation report with detailed findings

## Example Usage

User: "Correlate my security findings"
User: "Generate a SAST-DAST correlation report"
User: "Compare SonarQube and StackHawk results"

## Additional Resources

**Detailed Documentation:**
- [Workflow Steps](references/workflow-steps.md) - Detailed steps for gathering config, retrieving SAST/DAST issues
- [Correlation Analysis](references/correlation-analysis.md) - Complete correlation rules, validation logic, and source code analysis workflow
- [Report Template](references/report-template.md) - Full markdown report structure and formatting
- [Implementation Guide](references/implementation-guide.md) - Step-by-step implementation workflow and key success factors

**Examples and Reference:**
- [Contributing Guidelines](references/CONTRIBUTING.md) - How to contribute to this skill
- [License](references/LICENSE) - MIT License details
- [Examples Overview](references/examples/README.md) - Collection of example correlation reports
  - [SonarQube + ZAP Example](references/examples/sonarqube-zap-example/README.md) - Java Spring Boot application example
  - [SonarQube + StackHawk Example](references/examples/sonarqube-stackhawk-example/README.md) - Java Spring Boot application example
