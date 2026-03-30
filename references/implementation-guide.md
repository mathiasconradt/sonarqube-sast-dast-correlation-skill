# Implementation Guide

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
         - **IMPORTANT:** Find and delete **ALL** comments that contain "DAST Correlation" (search for 🔴, 🟠, 🟡, 🔵 icons or "DAST Correlation" text)
         - **CRITICAL:** Collect ALL matching comment keys FIRST, then delete them ALL in sequence
         - Do NOT stop after deleting the first matching comment - an issue may have multiple old correlation comments that all need to be removed
         - For each matching comment, delete using:
           ```bash
           curl -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/delete_comment" \
             -d "comment={comment_key}"
           ```
         - After deleting all correlation comments, remove the 'dast-detected' tag using:
           ```bash
           curl -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/set_tags" \
             -d "issue={issue_key}" \
             -d "tags="
           ```
      3. Report how many issues were cleared and how many total comments were deleted
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

## Key Success Factors

✅ **CRITICAL: READ THE SOURCE CODE** - The skill runs inside the project folder with full access to source code. ALWAYS read controller/service files to verify endpoint mappings before creating correlations. Use the Read tool to examine actual code.

✅ **CRITICAL: Validate endpoint mapping** - DAST URL must map to the actual endpoint defined in the source code:
   - Read the controller file from the SAST issue
   - Extract @GetMapping, @PostMapping, @RequestMapping annotations
   - Combine class-level and method-level mappings
   - Verify DAST URL matches this endpoint
   - Verify HTTP method matches (GET, POST, etc.)
   - **REJECT if endpoints don't match** - even if vulnerability categories match!

✅ **CRITICAL: Validate category/CWE matching** - Only correlate SAST and DAST findings if they are the SAME vulnerability type:
   - SQL Injection ↔ SQL Injection ✅
   - XSS ↔ XSS ✅
   - SQL Injection ↔ XSS ❌ REJECT
   - Use CWE numbers or extract category from rule names/titles/descriptions
   - **REJECT correlations with different categories** - this is non-negotiable

✅ **CRITICAL: Verify parameter flow** - Match the vulnerable parameter in code to the parameter DAST tested:
   - If SAST shows @RequestParam("query") is vulnerable, DAST should test "?query=..."
   - If SAST shows @PathVariable("id"), DAST should test the path variable
   - Read the actual code to see parameter annotations

✅ **CRITICAL: Trace data flow for Repository/Service classes** - If SAST issue is not in a Controller:
   - Use Grep to find which Controller calls this Repository/Service method
   - Read that Controller to get the endpoint
   - Verify the data flows from HTTP request → Controller → Repository/Service → Vulnerable line

✅ **Check environment variables FIRST** - Use `env | grep -i sonar` to find `SONAR_TOKEN`, `SONARQUBE_TOKEN`, `SONARQUBE_URL`, etc. before reading config files

✅ **Filter out imported DAST issues** from SonarQube data before analysis (rules starting with `external_`)

✅ **Use Agent tool** for correlation - it provides deeper reasoning and can read source files during analysis

✅ **Use severity-based icons** - 🔴 BLOCKER/CRITICAL, 🟠 MAJOR/HIGH, 🟡 MINOR/MEDIUM, 🔵 INFO/LOW (NOT based on vulnerability type)

✅ **Emphasize correlated findings** - these have highest confidence and priority (both SAST and DAST confirmed)

✅ **Provide direct links** to both SonarQube and DAST tool UIs for each issue

✅ **Use detected credentials automatically** - When all required config values are found (URL, token, projectKey), use them without asking the user

✅ **Prevent false correlations** - Better to return ZERO correlations than to create even ONE false correlation:
   - Don't correlate different vulnerability types
   - Don't correlate different endpoints
   - Don't correlate different HTTP methods
   - When in doubt, read the source code to verify
