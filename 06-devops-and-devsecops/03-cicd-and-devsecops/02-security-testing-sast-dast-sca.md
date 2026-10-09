# Application Security Testing Spectrum: SAST, DAST, IAST, SCA, and Container Scans

## 1. Topic & Definition
Securing modern software requires continuous vulnerability testing across the entire development lifecycle. The **Application Security Testing (AST)** spectrum comprises specialized methodologies that analyze code at different states of compilation, execution, and composition.

Deploying AST controls effectively requires understanding the distinct vantage points:
- **SAST (Static Analysis):** Analyzes uncompiled source code or bytecode from the inside out (White-Box).
- **DAST (Dynamic Analysis):** Tests a running application over HTTP from the outside in (Black-Box).
- **IAST (Interactive Analysis):** Instruments the application runtime with internal sensors (Gray-Box).
- **SCA (Software Composition Analysis):** Analyzes third-party open-source libraries and transitive dependencies.
- **Container Scanning:** Inspects base Linux OS packages and filesystem layers inside container images.

---

## 2. How Each Testing Paradigm Operates Internally

```
+---------------------------------------------------------------------------------------------------+
|                            The Application Security Testing Spectrum                              |
+---------------------------------------------------------------------------------------------------+
  METHODOLOGY    TESTING TARGET                INTERNAL MECHANISM                  PRIMARY TOOLS
  SAST           Raw Source Code / Bytecode    AST Traversal, Taint Analysis       Semgrep, SonarQube
  DAST           Running Deployed Web App      HTTP Fuzzing, Payload Injection     OWASP ZAP, Burp Suite
  IAST           Runtime Application Server    Bytecode Instrumentation Sensors    Contrast Security
  SCA            Lockfiles (package.json)      Dependency Graph CVE Matching       Snyk, Dependabot
  Container      Docker Image Layers (tar)     OS Package Database Inspection      Trivy, Grype
```

### A. SAST: Abstract Syntax Trees (AST) & Taint Analysis
- **Abstract Syntax Tree (AST):** SAST engines parse raw source code into hierarchical tree representations of programming language syntax.
- **Taint Analysis (Data Flow Tracking):** Traces untrusted user input from an untrusted **Source** (e.g., `request.getParameter("id")`) to a sensitive execution **Sink** (e.g., `db.execute()`).
  - If tainted data reaches a sink without passing through an approved **Sanitizer** (e.g., parameterized query or escaping function), SAST flags a vulnerability.

### B. DAST: Dynamic Black-Box Fuzzing
- **Operation:** Crawls a deployed web application, mapping forms, URL parameters, and API endpoints.
- **Execution:** Sends crafted exploit payloads (SQL injection strings `' OR 1=1--`, XSS `<script>alert(1)</script>`) and analyzes HTTP responses (status codes, response bodies, time delays) to identify vulnerabilities.
- **Advantage:** Validates actual exploitability under real server, WAF, and network configurations with near-zero false positives.

### C. SCA: Dependency Graph Resolution
- Modern applications consist of 80–90% open-source third-party code.
- **Operation:** Scans lockfiles (`package-lock.json`, `poetry.lock`, `go.sum`, `pom.xml`), constructs the complete direct and transitive dependency graph, and queries vulnerability databases (National Vulnerability Database / NVD, GitHub Advisory Database) to identify known CVEs and license compliance risks (e.g., AGPL-3.0).

---

## 3. Practical Pipeline Integration & Unified Reporting (SARIF)

### The Static Analysis Results Interchange Format (SARIF)
Modern CI/CD pipelines require disparate security scanners (Semgrep, Trivy, CodeQL) to output findings in **SARIF (JSON-based OASIS standard)**. This allows code-hosting platforms (GitHub Advanced Security, GitLab) to render annotations directly inline on developer pull requests:

```json
{
  "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/master/Schemata/sarif-schema-2.1.0.json",
  "version": "2.1.0",
  "runs": [
    {
      "tool": {
        "driver": { "name": "Semgrep", "version": "1.60.0" }
      },
      "results": [
        {
          "ruleId": "python.lang.security.deserialization.avoid-pickle",
          "level": "error",
          "message": { "text": "Avoid using unsafe pickle.loads() on untrusted data." },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": { "uri": "src/api/auth.py" },
                "region": { "startLine": 42 }
              }
            }
          ]
        }
      ]
    }
  ]
}
```

---

## 4. Key Differences Matrix: SAST vs DAST vs IAST vs SCA

| Dimension | SAST | DAST | IAST | SCA | Container Scanning |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Testing State** | Uncompiled / Static | Running / Dynamic | Running with Agent | Static Lockfiles | Static Container Image |
| **Perspective** | White-Box | Black-Box | Gray-Box | Compositional | OS / Infrastructure |
| **False Positive Rate**| **High** (Lacks runtime context) | **Very Low** (Confirmed via response)| Low | Low | Low |
| **Execution Speed** | Fast to Moderate (Minutes) | Slow (Hours for full crawl) | Real-time during QA tests | Extremely Fast (Seconds)| Fast (Seconds) |
| **Code Location** | Pinpoints exact file & line number | Reports URL / parameter only | Pinpoints exact line number | Reports library version | Reports OS package |
| **Pipeline Stage** | Pre-commit / Early CI | Staging / Pre-prod QA | QA / Automated functional tests| Early CI (Commit) | Artifact packaging (Post-build) |

---

## 5. Cybersecurity Relevance, Threats & False Positive Fatigue

### 1. The SAST False Positive Crisis & Developer Friction
- **The Challenge:** Traditional SAST tools (e.g., legacy Fortify or Checkmarx) generate massive false positive volumes (up to 70%), flagging dead code paths or sanitized inputs where custom sanitizers were not recognized.
- **Consequence:** Developers treat security reports as noise and ignore real vulnerabilities.
- **Remediation:** 
  1. Transition to modern syntactic AST tools (**Semgrep**) where security teams can customize lightweight YAML rules matching company coding standards.
  2. Implement **Baseline Scanning**: only fail builds on *newly introduced* vulnerabilities on changed lines (diff-aware scanning), while logging legacy tech debt for sprint remediation.

### 2. DAST Inability to Reach Authenticated State
- **Challenge:** DAST crawlers often fail to navigate modern Single Page Applications (SPAs) or get stuck behind MFA login screens.
- **Remediation:** Provide DAST crawlers with pre-authenticated OpenAPI/Swagger specs, recorded Selenium/Playwright login scripts, or authorized API tokens to test deep backend routes.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Securing applications requires a layered testing approach across the development lifecycle. SAST performs white-box analysis on source code using AST traversal and taint analysis to identify flaws like SQL injection early in development, but suffers from higher false positive rates. DAST tests running applications externally via HTTP fuzzing, confirming true exploitability with minimal false alarms, but operates later in staging. IAST combines both by instrumenting application runtimes with agent sensors during functional testing. SCA analyzes open-source dependency trees to catch known CVEs and license risks, while Container Scanners inspect OS filesystem layers. To prevent developer burnout, teams must use SARIF for unified PR annotations and enforce diff-aware scanning to block only newly introduced vulnerabilities."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Assuming SAST catches all vulnerabilities. *Correction:* SAST cannot evaluate runtime misconfigurations, authentication session cookie flags, or environmental infrastructure flaws that only exist when the app is running (DAST's domain).
- **Trap 2:** Confusing SCA with SAST. *Correction:* SAST inspects the custom code written by your developers; SCA inspects the open-source libraries imported via package managers.
- **Trap 3:** Blocking builds on low-severity SCA CVEs that are not loaded in memory. *Correction:* Many dependencies contain CVEs in auxiliary functions never called by the application; use reachability analysis (e.g., Snyk DeepCheck) to prioritize exploitable libraries.

### Expected Follow-Up Questions
1. *What is Taint Analysis in SAST?*
   - A data flow tracking technique that traces untrusted user input from an entry point (Source) to a sensitive execution function (Sink), raising an alert if the data reaches the sink without being validated or sanitized.
2. *Why is SARIF format significant for enterprise DevSecOps?*
   - Because it provides a single, vendor-neutral JSON schema for static analysis results, allowing organizations to aggregate findings from Semgrep, Trivy, CodeQL, and ESLint into a single unified dashboard without custom converters.
