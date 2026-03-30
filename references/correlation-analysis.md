# Deep Correlation Analysis with Source Code Analysis

**CRITICAL: The skill runs inside the project folder with full access to source code. USE THIS!**

**IMPORTANT: Use the Agent tool for correlation, NOT simple Python scripts**

Use the `Agent` tool with `subagent_type: "general-purpose"` to perform deep, intelligent correlation analysis:

```
Agent with prompt:
"Perform in-depth correlation analysis between SAST findings from sonar_issues.json
and DAST findings from {selected_sarif_file}.

**MANDATORY RULES:**
1. ONLY create correlations between SAST and DAST findings that are the SAME vulnerability category
2. ONLY correlate if the DAST URL actually maps to the code location of the SAST issue
3. You have full access to the project source code - READ THE ACTUAL FILES to verify correlations

**CORRELATION PROCESS - FOLLOW STRICTLY:**

For each SAST issue:

STEP 1: **Read the Source File**
   - Use Read tool to read the file mentioned in the SAST issue (e.g., ProductController.java)
   - Find the vulnerable line number from the SAST issue
   - Identify the controller method containing this line
   - Extract the endpoint mapping from annotations:
     * @RequestMapping("/path")
     * @GetMapping("/path")
     * @PostMapping("/path")
     * @PutMapping, @DeleteMapping, @PatchMapping
     * Look for both class-level and method-level mappings (they combine!)
   - Identify parameter names from:
     * @RequestParam("paramName")
     * @PathVariable("varName")
     * @RequestBody fields
     * Method parameter names

STEP 2: **Map Endpoint to URL**
   - Combine class-level and method-level mappings
   - Example: Class has @RequestMapping("/products"), method has @GetMapping("/direct")
   - Full endpoint: /products/direct
   - Note the HTTP method (GET, POST, etc.)

STEP 3: **Find Matching DAST Findings**
   - Look through DAST findings for URLs that match this endpoint
   - URL matching rules:
     * Exact match: DAST URL "/products/direct" matches endpoint "/products/direct"
     * Parameter match: DAST URL "/products/direct?param=value" matches endpoint "/products/direct"
     * Path variable match: DAST URL "/users/123" matches endpoint "/users/{id}"
     * Base URL variations: Strip protocol and domain, match path only
   - HTTP method MUST match (GET to GET, POST to POST)

STEP 4: **Verify Vulnerability Category Match**
   - SAST and DAST findings MUST be the same category (SQL Injection, XSS, Path Traversal, etc.)
   - Use CWE numbers if available in both sources (exact or related CWE match)
   - If CWE not available, extract category from: rule IDs, rule names, issue titles, descriptions, tags
   - Examples of VALID correlations: SQL Injection ↔ SQL Injection, XSS ↔ XSS, Path Traversal ↔ Path Traversal
   - Examples of INVALID correlations: SQL Injection ↔ XSS, Path Traversal ↔ CSRF, XSS ↔ Security Headers
   - **REJECT if categories don't match** - DO NOT create a correlation

STEP 5: **Verify Data Flow**
   - Read the SAST taint flow details
   - Check if the vulnerable parameter in the code matches the parameter DAST tested
   - Example: If SAST shows @RequestParam("param") is vulnerable, verify DAST tested "?param=..."
   - For POST requests, check if the vulnerable field matches the POST body parameter
   - Trace the data flow: source (request param) → sink (SQL query, HTML output, file operation)

STEP 6: **Assign Confidence Level**
   - **HIGH**: Same category + exact URL match + same HTTP method + parameter match + data flow verified
   - **MEDIUM**: Same category + URL match + same HTTP method (but parameters unclear)
   - **LOW**: Same category only, URL might be related but not exact match
   - **NO CORRELATION**: Different categories OR URL doesn't match OR wrong HTTP method

STEP 7: **Document the Correlation**
   - Explain exactly which endpoint in the code maps to which DAST URL
   - Show the controller annotation → endpoint mapping
   - Explain the parameter flow
   - Provide file:line references

**EXAMPLES OF GOOD CORRELATIONS:**

✅ CORRECT:
- SAST: ProductController.java:101, @GetMapping("/products/direct"), @RequestParam("param"), XSS vulnerability
- DAST: GET /products/direct?param=<script>alert(1)</script>, XSS detected
- Reasoning: "Exact endpoint match (/products/direct), same HTTP method (GET), vulnerable parameter 'param' was tested with XSS payload, same vulnerability category (XSS)"

✅ CORRECT:
- SAST: SearchRepository.java:23, used by HomeController @PostMapping("/"), SQL injection in search parameter
- DAST: POST / with body param "search='; DROP TABLE--", SQL injection detected
- Reasoning: "Endpoint match (POST /), SearchRepository.findBySearch() is called from HomeController.search(), parameter matches, same vulnerability (SQL injection)"

❌ INCORRECT:
- SAST: ProductController.java:101, @GetMapping("/products/direct"), XSS vulnerability
- DAST: POST /login, Cookie security issue
- Why wrong: Different endpoints, different HTTP methods, different vulnerability categories

❌ INCORRECT:
- SAST: UserController.java:50, @PostMapping("/users"), SQL injection
- DAST: GET /products/direct?param=<script>, XSS
- Why wrong: Completely different endpoints AND different vulnerability categories

**Remember: Better to return ZERO correlations than to create even ONE false correlation!**

For each correlation found, provide:
- SAST issue details (rule, file, line, message, taint flow, CWE if available)
- DAST issue details (rule, URI, method, severity, CWE if available)
- Detailed correlation reasoning explaining WHY they match (including category/CWE match)
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
      'sast_cwe': '...' (if available, e.g., 'CWE-89'),
      'sast_category': '...' (vulnerability type: 'SQL Injection', 'XSS', etc.),
      'sast_endpoint': '...' (extracted from source code, e.g., 'GET /products/{id}'),
      'sast_controller_method': '...' (method name from source, e.g., 'ProductController.getProduct()'),
      'sast_parameter': '...' (vulnerable parameter from source, e.g., '@RequestParam("query")'),
      'dast_rule_id': '...',
      'dast_uri': '...',
      'dast_method': '...',
      'dast_message': '...',
      'dast_level': '...',
      'dast_cwe': '...' (if available in SARIF, e.g., 'CWE-89'),
      'dast_category': '...' (vulnerability type extracted from rule/title),
      'dast_tested_parameter': '...' (parameter tested by DAST, extracted from URI or body),
      'endpoint_match': 'exact|partial|none' (how well the endpoints match),
      'parameter_match': true|false (whether parameters match),
      'http_method_match': true|false (whether HTTP methods match),
      'correlation_reasoning': 'Detailed explanation including:
        - Why vulnerability categories match (with CWE)
        - How endpoints map from source code to DAST URL
        - Why parameters match or don't match
        - Source code evidence (file:line, annotations)
        - Data flow verification',
      'confidence': 'high|medium|low',
      'source_code_verified': true|false (whether source code was actually read)
    }
  ],
  'summary': {
    'total_correlations': ...,
    'high_confidence_correlations': ...,
    'medium_confidence_correlations': ...,
    'low_confidence_correlations': ...,
    'vulnerability_types_correlated': [...],
    'source_files_analyzed': [...],
    'rejected_due_to_endpoint_mismatch': ...,
    'rejected_due_to_category_mismatch': ...,
    'rejected_due_to_http_method_mismatch': ...
  }
}"
```

## Source Code Analysis Workflow

**⚠️ CRITICAL: You MUST read the actual source files to verify endpoint mappings!**

For each SAST issue, follow this workflow:

1. **Read the Controller/Service File**
   ```
   Use Read tool on the file from SAST issue's 'component' field
   Example: if component is "...controller/ProductController.java", read that file
   ```

2. **Find the Vulnerable Code**
   - Locate the line number from the SAST issue
   - Identify the containing method
   - Extract controller annotations (@GetMapping, @PostMapping, etc.)
   - Identify parameter annotations (@RequestParam, @PathVariable, @RequestBody)

3. **Map the Endpoint**
   - Combine class-level @RequestMapping with method-level mapping
   - Example:
     ```java
     @RestController
     @RequestMapping("/api/products")  // Class level
     public class ProductController {
         @GetMapping("/{id}")  // Method level
         public Product getProduct(@PathVariable String id) {
             // Full endpoint: GET /api/products/{id}
         }
     }
     ```

4. **Find Files That Call This Code** (for Repository/Service classes)
   - If SAST issue is in a Repository or Service class (not a Controller):
     * Use Grep to search for method calls to this class
     * Find which Controller calls this Repository/Service
     * Read that Controller to get the endpoint
   - Example: SearchRepository.findByName() → Used by HomeController.search() → POST /search

5. **Match Against DAST URLs**
   - Extract path from DAST URI (strip protocol, domain, query params)
   - Compare with endpoint from source code
   - Verify HTTP method matches
   - Check if parameters in DAST request match code parameters

## Correlation Validation Rules

**⚠️ MANDATORY: All conditions must be true for a valid correlation:**

✅ **Category Match** (MANDATORY - ALWAYS CHECK FIRST):
   - SAST and DAST MUST be the same vulnerability type
   - Check CWE numbers first (e.g., CWE-89 for SQL Injection)
   - If no CWE, extract from rule names/descriptions
   - SQL Injection ↔ SQL Injection ✅
   - XSS ↔ XSS ✅
   - SQL Injection ↔ XSS ❌ REJECT
   - Path Traversal ↔ CSRF ❌ REJECT

✅ **Endpoint Match** (MANDATORY - READ SOURCE CODE):
   - DAST URL path must match the endpoint from source code annotations
   - HTTP method must match (GET, POST, PUT, DELETE, etc.)
   - Examples:
     * Code: @GetMapping("/products/{id}") → DAST: GET /products/123 ✅
     * Code: @PostMapping("/search") → DAST: POST /search ✅
     * Code: @GetMapping("/users") → DAST: POST /login ❌ REJECT

✅ **Parameter Match** (HIGHLY RECOMMENDED):
   - Vulnerable parameter in code should match tested parameter in DAST
   - Code: @RequestParam("query") → DAST: ?query=<malicious> ✅
   - Code: @PathVariable("id") → DAST: /users/123 ✅
   - If parameters don't match, lower confidence or reject

✅ **Data Flow Verification** (RECOMMENDED):
   - Verify the code path from SAST actually handles the request
   - For Repository/Service classes: verify a Controller calls this code
   - Trace: HTTP Request → Controller Method → Service/Repository Method → Vulnerable Line

## Confidence Levels

Status progresses after all validations pass:

- **HIGH**: Category match + exact endpoint match + HTTP method match + parameter match + data flow verified
- **MEDIUM**: Category match + endpoint match + HTTP method match (parameters unclear or don't match exactly)
- **LOW**: Category match + similar endpoint (might be related) + HTTP method match
- **REJECT**: Any of the following:
  * Different vulnerability categories
  * Endpoints don't match at all
  * HTTP methods don't match
  * Clear evidence the code path isn't related to the DAST finding

## Vulnerability Type Reference

- SQL Injection: `javasecurity:S3649`, `java:S2077` ↔ `40018`, `sql-injection`, CWE-89
- XSS: `javasecurity:S5131` ↔ `40012`, `40014`, `cross-site-scripting`, CWE-79
- Path Traversal: `javasecurity:S2083` ↔ `path-traversal`, `directory-traversal`, CWE-22
- XXE: `java:S2755` ↔ `xxe`, `xml-external-entity`, CWE-611
- Command Injection: ↔ `command-injection`, `90020`, CWE-78
- CSRF: ↔ `csrf`, `40016`, CWE-352
