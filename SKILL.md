---
name: sonarqube-sast-dast-correlation
description: Correlate SAST findings from SonarQube with DAST findings from SARIF files (StackHawk, ZAP, etc.) and generate a comprehensive security report
trigger: |
  User wants to correlate SAST and DAST findings, generate a security correlation report,
  or compare static and dynamic analysis results
---

# SonarQube SAST-DAST Correlation

Generate a comprehensive security correlation report that maps Static Application Security Testing (SAST) findings from SonarQube with Dynamic Application Security Testing (DAST) findings from SARIF files.

## Workflow

### 0. Check for Existing Report

Before starting the analysis, check if `sast-dast-correlation-report.md` already exists in the current directory.

If it exists:
1. Use AskUserQuestion to ask the user:
   - **Option 1:** "Open existing report" - Just open the existing report in the browser and skip to step 7
   - **Option 2:** "Rerun analysis" - Delete the existing report and proceed with fresh analysis from step 1

If it doesn't exist, proceed to step 1.

### 1. Gather SonarQube Configuration

**IMPORTANT:** Automatically check environment variables FIRST using Bash before reading config files.

Check for SonarQube configuration in the following order:

1. **Environment variables** (check these FIRST with `env | grep -i sonar`):
   - `SONARQUBE_URL` or `SONAR_HOST_URL` (for SonarQube URL)
   - `SONARQUBE_TOKEN` or `SONAR_TOKEN` (for authentication token)
   - `SONAR_PROJECT_KEY` or `SONARQUBE_PROJECT_KEY` (for project key)

2. **`.sonarlint/connectedMode.json`** - Look for:
   - `sonarQubeUri` or `serverUrl` (SONARQUBE_URL)
   - `token` (SONARQUBE_TOKEN)
   - `projectKey`

3. **`sonar-project.properties`** - Look for:
   - `sonar.host.url` (SONARQUBE_URL)
   - `sonar.login` or `sonar.token` (SONARQUBE_TOKEN)
   - `sonar.projectKey`

**Configuration Priority:**
- Use environment variables if found (highest priority)
- Fall back to `.sonarlint/connectedMode.json` if env vars not available
- Fall back to `sonar-project.properties` if neither above are available

After gathering available values:
- If all required values (URL, token, projectKey) are found: use them automatically without asking
- If only some values are found: present them to the user with AskUserQuestion and ask to confirm or fill in missing values
- If no values are found: ask user to provide all values

### 2. Retrieve SAST Issues

1. Check if `sonar_issues.json` exists in the current directory
2. If it exists:
   - Ask user if they want to use this file or fetch fresh data
3. If not exists or user wants fresh data:
   - Use the SonarQube API to fetch issues:
     ```
     GET {SONARQUBE_URL}/api/issues/search?componentKeys={projectKey}&statuses=OPEN,CONFIRMED,REOPENED&ps=500
     ```
   - Save the response to `sonar_issues.json`
   - Handle pagination if more than 500 issues exist

**CRITICAL: Filter out imported DAST issues from SAST data**

4. After loading `sonar_issues.json`, filter out any issues with rules starting with `external_StackHawk:` or other external tool prefixes
   - These are DAST findings that were imported into SonarQube
   - Only keep TRUE SAST issues (code analysis findings)
   - Track how many issues were filtered for reporting
   - Example: If there are 24 total issues and 12 are `external_StackHawk:*`, keep only the 12 true SAST issues

### 3. Retrieve DAST Issues

1. Search for all `*.sarif` files in the current directory using Glob
2. Present the list to the user with AskUserQuestion:
   - Show file names, sizes, and modification dates
   - Allow user to select one file
   - Provide option to enter a custom file path
3. Parse the selected SARIF file to extract:
   - Tool name (from `runs[0].tool.driver.name`)
   - Scan information (from `runs[0].automationDetails` if available)
   - All findings/results with their severity levels

### 4. Deep Correlation Analysis

**IMPORTANT: Use the Agent tool for correlation, NOT simple Python scripts**

Use the `Agent` tool with `subagent_type: "general-purpose"` to perform deep, intelligent correlation analysis:

```
Agent with prompt:
"Perform in-depth correlation analysis between SAST findings from sonar_issues.json
and DAST findings from {selected_sarif_file}.

Read both files and identify correlations by:
1. Matching vulnerability types (SQL injection, XSS, etc.)
2. Matching code paths to endpoints (e.g., SearchRepository.java → /search endpoint)
3. Analyzing taint flows from SAST and matching to exploited endpoints in DAST

For each correlation found, provide:
- SAST issue details (rule, file, line, message, taint flow)
- DAST issue details (rule, URI, method, severity)
- Detailed correlation reasoning explaining WHY they match
- Confidence level (high/medium/low)

Output to correlations.json with structure:
{
  'correlations': [
    {
      'sast_issue_key': '...',
      'sast_rule': '...',
      'sast_component': '...',
      'sast_line': ...,
      'sast_message': '...',
      'sast_severity': '...',
      'sast_flow_summary': '...',
      'dast_rule_id': '...',
      'dast_uri': '...',
      'dast_method': '...',
      'dast_message': '...',
      'dast_level': '...',
      'correlation_reasoning': 'Detailed explanation...',
      'confidence': 'high|medium|low'
    }
  ],
  'summary': {
    'total_correlations': ...,
    'vulnerability_types_correlated': [...]
  }
}"
```

**Correlation Strategies the Agent should use:**

1. **Vulnerability Type Matching** (PRIMARY):
   - SQL Injection: `javasecurity:S3649`, `java:S2077` ↔ `sql-injection`
   - XSS: `javasecurity:S5131` ↔ `cross-site-scripting-reflected`
   - Path Traversal: `javasecurity:S2083` ↔ `path-traversal`
   - XXE: `java:S2755` ↔ `xxe`

2. **Endpoint/Component Mapping**:
   - Match controller class names to URI paths
   - Example: `SearchRepository.java` used by `HomeController POST "/"` ↔ DAST finding at `POST /`
   - Example: `ProductController.java GET "/products/direct"` ↔ DAST finding at `GET /products/direct`

3. **Taint Flow Analysis**:
   - Examine SAST taint flows (source → sink)
   - Match source (@RequestParam, @PathVariable) to DAST request parameters
   - Match sink (SQL query, HTML output) to DAST vulnerability type

4. **Confidence Scoring**:
   - HIGH: Same vulnerability type + matching endpoint/component + taint flow aligns
   - MEDIUM: Same vulnerability type + partial component match
   - LOW: Same vulnerability type only

### 5. Generate Markdown Report

Create a comprehensive report: `sast-dast-correlation-report.md`

**Icon Mapping (Severity-Based):**

Use icons based on SAST severity levels (NOT vulnerability type):
- 🔴 Red = BLOCKER or CRITICAL severity
- 🟠 Orange = MAJOR or HIGH severity
- 🟡 Yellow = MINOR or MEDIUM severity
- 🔵 Blue = INFO or LOW severity

**Report Structure:**

```markdown
# SonarQube SAST-DAST Correlation Report

**Generated:** {timestamp}
**Project:** {projectKey}

## Executive Summary

- **SAST Tool:** SonarQube ([{SONARQUBE_URL}]({SONARQUBE_URL}))
- **DAST Tool:** {tool name from SARIF} ([Scan Results]({scan_uri}))
- **Total SAST Issues:** {count} *(filtered out {n} imported DAST issues)*
- **Total DAST Issues:** {count}
- **✅ Correlated Issues:** {count} **HIGH CONFIDENCE**
- **SAST-Only Issues:** {count}
- **DAST-Only Issues:** {count}

## Severity Distribution

| Severity Level | SAST Count | DAST Count | Correlated |
|----------------|------------|------------|------------|
| Critical/Error | {n} | {n} | {n} ✅ |
| High/Warning | {n} | {n} | {n} |
| Medium/Note | {n} | {n} | {n} |
| Low/Info | {n} | {n} | {n} |

## 🔥 Correlated Findings - HIGH PRIORITY

**{count} critical vulnerabilities** confirmed by BOTH static and dynamic analysis.

⚠️ **These issues have the highest confidence and should be fixed immediately.** Both SAST and DAST independently discovered these vulnerabilities, confirming they are real, exploitable security flaws.

---

{For each correlated issue:}

### {icon} Correlation #{n}: {Vulnerability Type}

#### SAST Finding (Code Analysis)

- **Rule:** `{rule}`
- **Severity:** {severity} {icon}
- **File:** `{component}`
- **Line:** {line}
- **Issue:** {message}
- **Taint Flow:** {taint flow summary showing source → sink}
- 🔗 **[View in SonarQube]({SONARQUBE_URL}/project/issues?id={projectKey}&issues={issue_key}&open={issue_key})**

#### DAST Finding (Runtime Testing)

- **Rule:** `{ruleId}`
- **Severity:** {level}
- **Method:** {HTTP method}
- **Endpoint:** `{URI}`
- **Issue:** {message}
- 🔗 **[View in {tool name}]({helpUri})**

#### Why These Correlate

{Detailed correlation reasoning from Agent analysis}

**Confidence Level:** {confidence} ✅

---

## SAST-Only Findings

Issues found only by static analysis (may not be exploitable in runtime):

{For each SAST-only issue, grouped by severity:}

### {Severity Level}

| Rule | Component | Message | Link |
|------|-----------|---------|------|
| {rule} | {component} | {message} | [View]({link}) |

## DAST-Only Findings

Issues found only by dynamic analysis (may indicate runtime-specific vulnerabilities):

{For each DAST-only issue, grouped by severity:}

### {Severity Level}

| Rule | Location | Message | Link |
|------|----------|---------|------|
| {ruleId} | {URI} | {message} | [View]({link}) |

## Coverage Analysis

### Vulnerability Categories Covered

- **SQL Injection:** SAST ✓ | DAST ✓
- **XSS:** SAST ✓ | DAST ✓
- **CSRF:** SAST ✗ | DAST ✓
- {etc...}

## Recommendations

### 🔥 IMMEDIATE ACTION REQUIRED

1. **Fix the {n} correlated vulnerabilities FIRST** - These are confirmed exploitable:
   {List each correlation with file name and line number}

### Priority Actions

2. **🔴 Remaining Critical SAST Issues**: Address {n} code-level vulnerabilities
3. **⚠️ DAST Runtime Issues**: Fix {n} runtime configuration issues (headers, cookies, CSRF)

### Analysis Insights

- **Correlation Success**: {n} out of {total} SAST issues confirmed exploitable ({%} validation rate)
- **SAST Coverage**: Found code-level issues not detectable at runtime (path traversal, XXE, etc.)
- **DAST Coverage**: Found {n} runtime configuration issues invisible to static analysis
- **Complementary Testing**: Both tools provide essential, non-overlapping coverage

## Detailed Findings

### All SAST Issues

{Expandable/collapsible section with full SAST issue list}

### All DAST Issues

{Expandable/collapsible section with full DAST issue list}

---

*Report generated by Claude Code SonarQube SAST-DAST Correlation Skill*
```

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
- **IMPORTANT**: If correlation produces no matches, the Agent analysis may not have been thorough enough
  - Common correlations to look for: SQL injection, XSS, path traversal, XXE
  - Re-run the Agent with more specific instructions to find matching vulnerability types
  - Don't assume "no correlations" - usually there ARE correlations if both tools ran on the same app
- Validate SARIF file format before processing
- Remember to filter out `external_StackHawk:` or similar imported issues from SAST data

## Output Files

- `sonar_issues.json` - SAST issues from SonarQube (if created)
- `correlations.json` - Correlation analysis results from Agent (intermediate file)
- `sast-dast-correlation-report.md` - Final correlation report with detailed findings

## Implementation Workflow

When executing this skill:

1. **Check for Existing Report** - Look for `sast-dast-correlation-report.md` and ask if user wants to open it or rerun analysis
2. **Gather Configuration** - Check environment variables FIRST (`env | grep -i sonar`), then read `.sonarlint/connectedMode.json` if needed, use values automatically if all are found
3. **Find SARIF Files** - Use Glob to find `*.sarif` files and ask user which to use
4. **Check for Existing Data** - Look for `sonar_issues.json` and ask if user wants to use it or fetch fresh
5. **Filter SAST Issues** - Remove `external_StackHawk:*` rules from SAST data using Bash/Python
6. **Run Agent Analysis** - Use Agent tool (general-purpose) to create `correlations.json`
7. **Generate Report** - Use the correlations.json + filtered data to create the final markdown report
8. **Open Report in Browser** - Automatically open `sast-dast-correlation-report.md` in the default browser:
   - macOS: `open sast-dast-correlation-report.md`
   - Linux: `xdg-open sast-dast-correlation-report.md`
   - Windows: `start sast-dast-correlation-report.md`
9. **Inform User** - Tell user about correlations found and that the report has been opened in their browser
10. **Tag Correlated Issues (Optional)** - After opening the report, follow this multi-step process:

    **Step 10a: Ask User About Tagging**
    - Use AskUserQuestion to ask if user wants to tag the correlated issues on SonarQube
    - If user declines, skip the rest of step 10
    - If user agrees, proceed to Step 10b

    **Step 10b: Ask About Clearing Existing Tags**
    - Use AskUserQuestion to ask: "Should I clear existing 'dast-detected' tags and correlation comments before applying new ones?"
    - **If user agrees to clear:**
      1. Fetch all issues with 'dast-detected' tag using SonarQube API:
         ```bash
         curl -u "$SONAR_TOKEN:" "{SONARQUBE_URL}/api/issues/search?componentKeys={projectKey}&tags=dast-detected&ps=500"
         ```
      2. For each issue found:
         - Get the issue details including comments
         - Find and delete comments that contain "DAST Correlation" (any icon variant) using:
           ```bash
           curl -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/delete_comment" \
             -d "comment={comment_key}"
           ```
         - Remove the 'dast-detected' tag using:
           ```bash
           curl -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/set_tags" \
             -d "issue={issue_key}" \
             -d "tags="
           ```
      3. Report how many issues were cleared
    - **If user declines to clear:**
      - Skip clearing steps and proceed directly to Step 10c
      - Inform user that new tags will be added alongside any existing tags

    **Step 10c: Tag New Correlations**
    - **CRITICAL:** Use the SonarQube URL and token from step 1 (environment variables or config files)
    - Check environment variables again if needed: `SONAR_TOKEN` or `SONARQUBE_TOKEN`, `SONARQUBE_URL` or `SONAR_HOST_URL`
    - For each newly correlated issue from correlations.json:
      - Add tag "dast-detected" using SonarQube API with curl:
        ```bash
        curl -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/set_tags" \
          -d "issue={issue_key}" \
          -d "tags=dast-detected"
        ```
      - Add a comment with correlation details using SonarQube API:
        ```bash
        curl -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/add_comment" \
          -d "issue={issue_key}" \
          --data-urlencode "text={correlation_details}"
        ```
      - The comment should include:
        - {icon} DAST Correlation - HIGH CONFIDENCE header (icon based on SAST severity: 🔴=BLOCKER/CRITICAL, 🟠=MAJOR/HIGH, 🟡=MINOR/MEDIUM, 🔵=INFO/LOW)
        - Vulnerability type (SQL Injection, XSS, etc.)
        - SAST severity level
        - Correlation confidence level
        - DAST tool and finding details
        - Correlation reasoning (from the markdown report)
        - Link to the DAST finding
    - **Authentication:** Use `curl -u "$SONAR_TOKEN:"` (token as username, empty password) for API calls
    - Report success/failure for each tagging operation
    - Provide summary of:
      - How many existing tags were cleared (if applicable)
      - How many new issues were tagged successfully
      - Total issues now tagged with 'dast-detected'

## Example Usage

User: "Correlate my security findings"
User: "Generate a SAST-DAST correlation report"
User: "Compare SonarQube and StackHawk results"

## Key Success Factors

✅ **Check environment variables FIRST** - Use `env | grep -i sonar` to find `SONAR_TOKEN`, `SONARQUBE_TOKEN`, `SONARQUBE_URL`, etc. before reading config files
✅ **Filter out imported DAST issues** from SonarQube data before analysis
✅ **Use Agent tool** for correlation - it provides deeper reasoning than simple scripts
✅ **Match vulnerability types precisely** - SQL injection to sql-injection, XSS to cross-site-scripting
✅ **Trace code paths to endpoints** - SearchRepository.java → POST / endpoint
✅ **Use severity-based icons** - 🔴 BLOCKER/CRITICAL, 🟠 MAJOR/HIGH, 🟡 MINOR/MEDIUM, 🔵 INFO/LOW (NOT based on vulnerability type)
✅ **Emphasize correlated findings** - these have highest confidence and priority
✅ **Provide direct links** to both SonarQube and DAST tool UIs for each issue
✅ **Use detected credentials automatically** - When all required config values are found (URL, token, projectKey), use them without asking the user
