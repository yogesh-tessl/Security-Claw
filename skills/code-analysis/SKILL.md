---
name: code-analysis
description: "Static and dynamic code analysis (SAST/DAST/SCA) for identifying security vulnerabilities in application source code and running systems. Supports Python, JavaScript, Java, Go, PHP, Ruby, and TypeScript. Performs taint analysis, secret detection, dependency CVE scanning, insecure function detection, hardcoded credential identification, and unsafe deserialization pattern detection. Use when auditing a codebase for vulnerabilities, scanning dependencies for known CVEs, detecting leaked secrets in git history, or running DAST against a live application."
metadata:
  {
    "openclaw":
      {
        "emoji": "🧬",
        "requires": { "bins": ["python3", "semgrep"] },
        "install":
          [
            {
              "id": "pip-sast",
              "kind": "shell",
              "cmd": "pip3 install semgrep bandit safety",
              "bins": ["semgrep", "bandit"],
              "label": "Install SAST tools (semgrep, bandit, safety)",
            },
            {
              "id": "brew-secrets",
              "kind": "shell",
              "cmd": "brew install trufflehog gitleaks",
              "bins": ["trufflehog", "gitleaks"],
              "label": "Install secret scanners (brew)",
            },
          ],
      },
  }
---

# Code Analysis — Static & Dynamic Security Analysis

## Recommended Workflow

Follow this sequence for comprehensive security analysis. Each phase feeds the next.

### Phase 1: SAST — Identify Code-Level Vulnerabilities

Run static analysis first to catch injection flaws, insecure patterns, and logic bugs.

```bash
semgrep --config=p/owasp-top-ten ./src --severity=ERROR --sarif > sast_results.sarif
semgrep --config=p/python-security ./src
semgrep --config=p/javascript ./src
```

For Python codebases, add Bandit for deeper coverage:

```bash
bandit -r ./app --severity-level high -f json -o bandit.json
```

**Checkpoint:** Review SAST findings before proceeding. Confirm true positives by inspecting the flagged code path. Dismiss false positives by checking whether the input is already sanitized or validated upstream.

### Phase 2: Triage Findings

Not all findings are exploitable. For each SAST result:

1. Check the data flow — does untrusted input actually reach the sink?
2. Check for existing sanitization or validation in the call chain.
3. Classify as true positive, false positive, or needs-investigation.
4. Prioritize by severity: ERROR > WARNING > INFO.

```bash
# Extract high-severity findings with CWE mapping
bandit -r ./app --format json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for r in data['results']:
    print(f\"[{r['issue_severity']}] {r['issue_text']}\")
    print(f\"  File: {r['filename']}:{r['line_number']}\")
    print(f\"  CWE:  {r.get('issue_cwe',{}).get('id','?')}\")
"
```

### Phase 3: SCA — Scan Dependencies for Known CVEs

```bash
# Python
safety check -r requirements.txt --json
pip-audit -r requirements.txt -f json -o pip_audit.json

# Node.js
npm audit --json > npm_audit.json

# Multi-language / containers
dependency-check --project "Target App" --scan ./lib --format JSON --out dep_check_report
trivy fs ./
trivy image target:latest
```

### Phase 4: Secret Detection

Scan git history and current files for leaked credentials.

```bash
trufflehog git file://./repo --only-verified
trufflehog github --org=target-org --only-verified --json > secrets.json
gitleaks detect --source ./repo --report-format json --report-path leaks.json
```

### Phase 5: IaC & DAST (If Applicable)

Run infrastructure and dynamic scans when Terraform/K8s configs or a live application are in scope.

```bash
# IaC scanning
checkov -d ./terraform --framework terraform --compact
semgrep --config=p/terraform-security ./terraform
semgrep --config=p/dockerfile-security .

# DAST against a running target
nuclei -u https://target.com -severity critical,high -t cves/ -t exposures/ -t misconfiguration/ -o dast_findings.txt
docker run -t owasp/zap2docker-stable zap-baseline.py -t https://target.com -r zap_report.html
docker run -t owasp/zap2docker-stable zap-api-scan.py -t https://target.com/swagger.json -f openapi
```

### Phase 6: Validate Fixes

After remediating findings, re-run the relevant scanner to confirm the fix and check for regressions.

```bash
# Re-scan after SAST fix
semgrep --config=p/owasp-top-ten ./src --severity=ERROR

# Re-scan after dependency update
safety check -r requirements.txt --json

# Re-scan after secret rotation
trufflehog git file://./repo --only-verified
```

---

## Tool Reference

| Type              | Tool                    | Coverage                             |
| ----------------- | ----------------------- | ------------------------------------ |
| **SAST**          | Semgrep                 | Multi-language taint analysis        |
| **SAST (Python)** | Bandit                  | Python-specific security issues      |
| **Secrets**       | TruffleHog, Gitleaks    | Keys, tokens, passwords in code      |
| **SCA**           | Safety, OWASP Dep-Check | Vulnerable dependencies              |
| **DAST**          | nuclei, ZAP             | Running application vulnerabilities  |
| **IaC Scanning**  | Semgrep, checkov        | Terraform, CloudFormation misconfigs |

---

## Additional Commands

### Language-Specific SAST Packs

```bash
semgrep --config=p/java ./src
semgrep --config=p/golang-security ./src
semgrep --config=p/php ./src
semgrep --config=p/jwt ./src
semgrep --config=p/flask-secure-defaults ./src
semgrep --config=p/django-security ./src
```

### Custom Semgrep Rules

```yaml
# custom_rules/unsafe_db.yaml
rules:
  - id: unsafe-string-format-sql
    patterns:
      - pattern: |
          db.execute("..." % ...)
      - pattern: |
          db.execute(f"...{...}...")
    message: "Potential SQL injection via string formatting. Use parameterized queries."
    languages: [python]
    severity: ERROR
    metadata:
      owasp: "A03:2021 Injection"
      cwe: "CWE-89"
```

```bash
semgrep --config=custom_rules/unsafe_db.yaml ./src
```

For further reference, see the [Semgrep rule registry](https://semgrep.dev/r), [Bandit test plugins](https://bandit.readthedocs.io/en/latest/plugins/index.html), and [OWASP Dependency-Check documentation](https://owasp.org/www-project-dependency-check/).
