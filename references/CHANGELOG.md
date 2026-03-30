# Changelog

## 2026-03-30 - Tagging Behavior Fix

### Issues Fixed

1. **Incorrect Tag Name** - Was adding multiple tags (`dast-confirmed`, `sast-dast-correlation`, `sql-injection`), now correctly uses ONLY `dast-detected`
2. **Missing Comments** - Comments with correlation details are now properly added to each tagged issue
3. **Missing Cleanup** - Old `dast-detected` tags and correlation comments are now automatically removed before applying new tags

### Changes Made

#### SKILL.md
- Added prominent "⚠️ CRITICAL: Issue Tagging Rules" section emphasizing:
  - Use ONLY the tag `dast-detected` (no other tags)
  - Automatic cleanup of old tags and comments before tagging new issues
  - Comment format with severity icons and correlation details
- Removed redundant API command examples (kept in Implementation Guide only)
- Reduced line count from 215 to 156 lines for better efficiency

#### references/implementation-guide.md
- **Step 10b**: Changed from "Ask About Clearing" to "Clear Existing Tags (AUTOMATIC)"
  - No longer asks user - cleanup is automatic and required
  - Emphasizes deleting ALL correlation comments (not just the first one)
  - Clear workflow: fetch issues with `dast-detected` tag → delete all correlation comments → remove tags
- **Step 10c**: Added explicit warning:
  - ⚠️ Use ONLY tag `dast-detected` - NO OTHER TAGS
  - Lists examples of what NOT to add: `sast-dast-correlation`, `sql-injection`, `xss`, `dast-confirmed`
  - Added complete comment format example with all required fields

### Tagging Workflow (Correct Behavior)

When user requests tagging (Step 9):

1. **Automatic Cleanup (no user prompt)**
   ```bash
   # Fetch all issues with existing 'dast-detected' tag
   curl -s -u "$SONAR_TOKEN:" "{SONARQUBE_URL}/api/issues/search?componentKeys={projectKey}&tags=dast-detected&ps=500"

   # For each issue: delete ALL correlation comments (search for 🔴 🟠 🟡 🔵 or "DAST Correlation")
   curl -s -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/delete_comment" -d "comment={comment_key}"

   # Remove 'dast-detected' tag from all issues
   curl -s -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/set_tags" -d "issue={issue_key}" -d "tags="
   ```

2. **Tag New Correlations**
   ```bash
   # Add ONLY 'dast-detected' tag (no other tags!)
   curl -s -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/set_tags" \
     -d "issue={issue_key}" \
     -d "tags=dast-detected"

   # Add detailed correlation comment
   curl -s -u "$SONAR_TOKEN:" -X POST "{SONARQUBE_URL}/api/issues/add_comment" \
     -d "issue={issue_key}" \
     --data-urlencode "text=🔴 **DAST Correlation - HIGH CONFIDENCE**

   **Vulnerability:** SQL Injection
   **SAST Severity:** BLOCKER
   **Confidence Level:** HIGH

   **DAST Tool:** ZAP
   **DAST Rule:** 40018

   **Correlation Details:**
   - ✅ Exact endpoint match
   - ✅ Parameter match verified
   - ✅ Source code verified

   **Remediation:** Use parameterized queries

   🔗 [View DAST Documentation](link)"
   ```

### Quality Metrics

- **Tessl Score**: 93% (maintained after optimization)
- **Line Count**: 156 lines (down from 215, -27% reduction)
- **Description**: 100% (specificity, trigger quality, completeness, distinctiveness)
- **Content**: 85% (conciseness, actionability, workflow clarity, progressive disclosure)

### Validation

✅ Tag name corrected: `dast-detected` only
✅ Automatic cleanup implemented
✅ Comment format documented with examples
✅ Implementation guide updated with explicit warnings
✅ Tessl score maintained at 93%
✅ File size optimized (-59 lines)

### Documentation Structure

```
sonarqube-sast-dast-correlation/
├── SKILL.md (main entry point, 156 lines)
│   └── Links to detailed documentation
├── references/
│   ├── implementation-guide.md (Step 10 has complete tagging workflow)
│   ├── workflow-steps.md
│   ├── correlation-analysis.md
│   └── report-template.md
└── CHANGELOG.md (this file)
```

### Breaking Changes

None - this is a bug fix that restores the original intended behavior.

### Migration Notes

If you have existing issues tagged with incorrect tags (`dast-confirmed`, `sast-dast-correlation`, etc.), the automatic cleanup will remove ALL `dast-detected` tags before reapplying them correctly. The incorrect tags will remain but won't interfere with the new tagging system.

To manually clean up incorrect tags:
```bash
# Remove incorrect tags from all issues (run once)
curl -s -u "$SONAR_TOKEN:" "https://sonarqube.example.com/api/issues/search?componentKeys={projectKey}&tags=dast-confirmed&ps=500" | \
  jq -r '.issues[].key' | \
  xargs -I {} curl -s -u "$SONAR_TOKEN:" -X POST "https://sonarqube.example.com/api/issues/set_tags" -d "issue={}" -d "tags="
```
