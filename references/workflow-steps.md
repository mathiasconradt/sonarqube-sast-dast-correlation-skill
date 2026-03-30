# Workflow Steps

## 0. Check for Existing Report

**IMPORTANT:** Create `.sonar/` directory if it doesn't exist: `mkdir -p .sonar`

Before starting the analysis, check if `.sonar/sast-dast-correlation-report.md` already exists.

If it exists:
1. Use AskUserQuestion to ask the user:
   - **Option 1:** "Open existing report" - Just open the existing report in the browser and skip to step 7
   - **Option 2:** "Rerun analysis" - Delete the existing report and proceed with fresh analysis from step 1

If it doesn't exist, proceed to step 1.

## 1. Gather SonarQube Configuration

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

## 2. Retrieve SAST Issues

1. Check if `.sonar/sonar_issues.json` exists
2. If it exists:
   - Ask user if they want to use this file or fetch fresh data
3. If not exists or user wants fresh data:
   - Use the SonarQube API to fetch issues:
     ```bash
     curl -s -u "$SONAR_TOKEN:" \
       "{SONARQUBE_URL}/api/issues/search?componentKeys={projectKey}&statuses=OPEN,CONFIRMED,REOPENED&ps=500" \
       -o .sonar/sonar_issues.json
     ```
   - Handle pagination if more than 500 issues exist

**CRITICAL: Filter out imported DAST issues from SAST data**

4. After loading `.sonar/sonar_issues.json`, filter out any issues with rules starting with `external_StackHawk:` or other external tool prefixes
   - These are DAST findings that were imported into SonarQube
   - Only keep TRUE SAST issues (code analysis findings)
   - Track how many issues were filtered for reporting
   - Example: If there are 24 total issues and 12 are `external_StackHawk:*`, keep only the 12 true SAST issues
   - Save filtered results to `.sonar/sast_issues_filtered.json`

## 3. Retrieve DAST Issues

1. Search for all `*.sarif` files in the current directory using Glob
2. Present the list to the user with AskUserQuestion:
   - Show file names, sizes, and modification dates
   - Allow user to select one file
   - Provide option to enter a custom file path
3. Parse the selected SARIF file to extract:
   - Tool name (from `runs[0].tool.driver.name`)
   - Scan information (from `runs[0].automationDetails` if available)
   - All findings/results with their severity levels
