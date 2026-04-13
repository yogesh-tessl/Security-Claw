---
name: bugbounty-automation
description: "Fully automated bug bounty pipeline that logs into HackerOne, Bugcrowd, or OpenBugBounty via browser, selects a program, extracts scope, runs multi-phase recon and vulnerability scanning at 1 RPS, detects WAF and applies bypass strategies, captures screenshot evidence for confirmed findings, writes a structured report, and broadcasts alerts to all configured channels. Use when the operator wants to run an end-to-end bug bounty hunt with zero manual intervention after launch."
metadata:
  {
    "openclaw":
      {
        "emoji": "🐛",
        "requires":
          {
            "bins":
              [
                "python3",
                "nuclei",
                "subfinder",
                "httpx",
                "ffuf",
                "sqlmap",
                "wafw00f",
                "nmap",
                "curl",
              ],
          },
        "install":
          [
            {
              "id": "bb-macos",
              "kind": "shell",
              "cmd": "brew install nuclei subfinder httpx ffuf sqlmap wafw00f nmap && pip3 install requests httpx[cli] rich pyyaml playwright && playwright install chromium",
              "bins": ["nuclei", "subfinder", "httpx", "ffuf", "sqlmap", "wafw00f", "nmap"],
              "label": "Install bug bounty tools (macOS via brew)",
              "when": "platform == 'darwin'",
            },
            {
              "id": "bb-linux",
              "kind": "shell",
              "cmd": "sudo apt-get install -y nmap sqlmap wafw00f ffuf 2>/dev/null || true && go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest && go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest && go install github.com/projectdiscovery/httpx/cmd/httpx@latest && pip3 install requests rich pyyaml playwright && playwright install chromium",
              "bins": ["nuclei", "subfinder", "httpx", "ffuf", "sqlmap", "wafw00f", "nmap"],
              "label": "Install bug bounty tools (Linux via apt + Go + pip)",
              "when": "platform == 'linux'",
            },
            {
              "id": "bb-windows",
              "kind": "shell",
              "cmd": "choco install nmap sqlmap ffuf -y && go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest && go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest && go install github.com/projectdiscovery/httpx/cmd/httpx@latest && pip3 install requests rich pyyaml playwright && playwright install chromium",
              "bins": ["nuclei", "subfinder", "httpx", "ffuf", "sqlmap", "nmap"],
              "label": "Install bug bounty tools (Windows via choco + Go + pip)",
              "when": "platform == 'win32'",
            },
          ],
      },
  }
---

# Bug Bounty Automation — Zero-Touch Hunter

End-to-end automated bug bounty pipeline: program selection, recon, scanning, WAF bypass, evidence capture, report generation, and all-channel broadcast.

> [!IMPORTANT]
> Only test targets within the program's defined scope. Every test is rate-limited to **1 RPS** by default. Never exceed authorized scope.

## Pipeline Overview

| Phase | Description |
|-------|-------------|
| 0 | Browser Login (HackerOne / Bugcrowd / OpenBugBounty) |
| 1 | Program Selection and Scope Extraction |
| 2 | Recon (subdomains, live hosts, tech fingerprinting) |
| 3 | WAF Detection and Bypass Strategy Selection |
| 4 | Vulnerability Scanning at 1 RPS |
| 5 | Confirmation and Screenshot Evidence |
| 6 | Report Generation |
| 7 | Broadcast to All Channels |

## Usage

```
Run the bug bounty automation pipeline on HackerOne — pick the best program and find all bugs
Start a bug bounty hunt on Bugcrowd, rate-limit to 1 RPS, notify all channels on every finding
Hunt bugs on HackerOne program example.com with WAF bypass payloads enabled
```

## Phase 0 — Browser Login

The agent opens the platform in the browser, prompts sign-in, then reuses the session for all subsequent API and scope calls.

### Validation Checkpoint

Before proceeding, verify all required tools are installed:

```bash
for tool in nuclei subfinder httpx ffuf sqlmap wafw00f nmap curl python3; do
  command -v "$tool" >/dev/null || echo "MISSING: $tool — install before continuing"
done
```

### Login Steps

1. Navigate to the platform login page:
   - HackerOne: `https://hackerone.com/users/sign_in`
   - Bugcrowd: `https://bugcrowd.com/user/sign_in`
   - OpenBugBounty: `https://www.openbugbounty.org/login/`
2. Prompt user: "Please sign in to [platform] in the browser window."
3. Wait for redirect to dashboard or post-login page.
4. Save session cookies for API calls.

## Phase 1 — Program Selection and Scope Extraction

### Selection Criteria (priority order)

| Criterion | Preference |
|-----------|------------|
| Bounty availability | Programs with bounties (not VDP-only) |
| Scope breadth | Wildcard `*.domain.com` preferred over single domain |
| Recency | Recently updated programs (active) |
| Report velocity | Lower competition for unique bugs |
| Payout range | Highest max critical bounty |

### Program API Endpoints

```bash
# HackerOne — fetch open bounty programs sorted by payout
GET https://hackerone.com/programs.json?product_type=bug-bounty&ordering=Highest+bounty&open_to_public=true
# Extract scope: GET https://hackerone.com/{program-slug}/policy_scopes.json
# Only test scope_type=IN_SCOPE; skip OUT_OF_SCOPE entries

# Bugcrowd — fetch bounty programs
GET https://bugcrowd.com/programs.json?reward_type=bounty&sort=promoted
# Extract targets: GET https://bugcrowd.com/{program}/brief.json

# OpenBugBounty — scrape (no official API)
GET https://www.openbugbounty.org/bugbounty/
```

### Scope Validation Checkpoint

After extracting scope, confirm each target domain resolves and is explicitly listed as IN_SCOPE before any active testing. Log the validated scope list for audit trail.

## Phase 2 — Recon at 1 RPS

All active recon tools are rate-limited to 1 request per second per target host.

### Subdomain Enumeration (passive first)

```bash
# Passive — no direct contact with target
subfinder -d TARGET -o recon/subs.txt -silent -all

# Certificate transparency
curl -s "https://crt.sh/?q=%.TARGET&output=json" \
  | python3 -c "import json,sys;[print(r['name_value']) for r in json.load(sys.stdin)]" \
  | sort -u >> recon/subs.txt

sort -u recon/subs.txt -o recon/subs_all.txt
```

### Live Host Probing

```bash
httpx -l recon/subs_all.txt \
  -rate-limit 1 \
  -tech-detect -title -status-code \
  -o recon/live.txt --json -o recon/live.json
```

### Technology Fingerprinting

```bash
whatweb --log-json=recon/tech.json \
  --wait=1 --max-threads=1 \
  $(cat recon/live.txt | tr '\n' ' ')
```

## Phase 3 — WAF Detection and Bypass

### Detect WAF

```bash
wafw00f https://TARGET -o recon/waf.txt -f json
```

Detects Cloudflare, Akamai, AWS WAF, Imperva, F5, Sucuri, ModSecurity, Barracuda, and others.

### Bypass Strategies

If a WAF is detected, apply these bypass strategies to all payloads across vulnerability classes:

| Strategy | Technique | Example |
|----------|-----------|---------|
| Encoding | HTML entity, URL, double-URL, base64 encoding of payload keywords | `&#97;&#108;&#101;&#114;&#116;` for `alert` |
| Case mutation | Mixed-case keywords to evade case-sensitive rules | `SeLeCt`, `ScRiPt` |
| Comment obfuscation | Inline comments breaking keyword signatures | `SE/**/LECT`, `scr<!---->ipt` |
| Whitespace alternatives | Tabs, newlines, extra spaces instead of standard whitespace | `SELECT%09*%09FROM` |
| Protocol tricks | Alternative protocols, IP encoding for SSRF | Decimal IP `2130706433`, hex `0x7f000001`, IPv6 `[::1]` |
| Wildcard/expansion | Shell brace expansion and variable substitution | `{cat,/etc/passwd}`, `${IFS}` |

The agent generates context-specific bypass payloads at runtime based on the detected WAF product. Claude already knows the full payload sets — the strategy table above guides selection.

### sqlmap WAF Bypass Tampers

When WAF is detected, add tamper chain to sqlmap:

```
--tamper=space2comment,charencode,randomcase,between,equaltolike
```

## Phase 4 — Vulnerability Scanning at 1 RPS

All scans use `--rate-limit 1` or equivalent. Tools run sequentially per target with 1-second inter-request delay.

### Nuclei (CVE and Misconfiguration)

```bash
nuclei -l recon/live.txt \
  -severity critical,high,medium \
  -rate-limit 1 -bulk-size 1 -concurrency 1 \
  -stats -json -o findings/nuclei.json \
  -markdown-export findings/nuclei_report/
```

### SQL Injection (sqlmap)

```bash
sqlmap -l recon/requests.txt \
  --batch --level=3 --risk=2 \
  --delay=1 --timeout=30 \
  --technique=BEUST \
  --tamper=space2comment,charencode,randomcase \
  --dbs --json-output=findings/sqli.json
```

### XSS (ffuf + dalfox)

```bash
ffuf -u "https://TARGET/FUZZ" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -rate 1 -mc 200,301,302 -o findings/params.json

dalfox url "https://TARGET" \
  --delay 1000 --waf-bypass \
  --format json -o findings/xss.json
```

### SSRF Detection

Test common SSRF-prone parameters (`url`, `redirect`, `next`, `target`, `dest`, `uri`, `link`, `src`, `path`) against callback and internal targets. Use 1-second delay between requests. Check responses for indicators like `root:`, `ami-id`, or `instance-id`.

### Path Traversal

```bash
ffuf -u "https://TARGET/FUZZ" \
  -w findings/traversal_payloads.txt \
  -rate 1 -mr "root:x|\\[boot loader\\]" \
  -o findings/traversal.json
```

### Open Redirect

```bash
ffuf -u "https://TARGET/?redirect=FUZZ" \
  -w /usr/share/seclists/Fuzzing/redirect-urls.txt \
  -rate 1 -mr "Location: https://evil.com" \
  -o findings/redirects.json
```

### Subdomain Takeover

```bash
nuclei -l recon/subs_all.txt \
  -t takeovers/ -rate-limit 1 \
  -json -o findings/takeovers.json
```

## Phase 5 — Confirmation and Screenshot Evidence

Every finding is confirmed before reporting:

1. Replay the PoC payload via direct request to confirm deterministic response.
2. Open the endpoint in browser with the confirmed payload.
3. Capture screenshot: `evidence/CLAW-YYYY-NNN_[type]_[host]_screenshot.png`
4. Annotate the screenshot highlighting the finding.

### Evidence Package per Finding

```
evidence/CLAW-2026-001/
  request.txt          # raw HTTP request
  response.txt         # raw HTTP response
  screenshot.png       # browser screenshot
  poc_command.sh       # reproducible PoC
  nuclei_output.json   # raw tool output (if applicable)
```

## Phase 6 — Report Generation

### Finding Format

Each finding follows this structure:

```markdown
## CLAW-2026-001 — [Vulnerability Type in /endpoint]

| Field | Value |
|-------|-------|
| **Severity** | CRITICAL |
| **CVSS** | 9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H) |
| **CWE** | CWE-89 |
| **Host** | api.example.com |
| **Endpoint** | /api/search?q= |
| **WAF Bypassed** | Yes (Cloudflare — tamper: space2comment,charencode) |

### Description
[Concise description of the vulnerability and its impact]

### Reproduction (PoC)
[Exact command to reproduce]

### Evidence
[Screenshot and extracted data]

### Business Impact
[Impact statement]

### Remediation
[Fix recommendation]
```

### HackerOne API Submission

Submit confirmed findings via the HackerOne Reports API (`POST /v1/hackers/reports`) with `team_handle`, `title`, `vulnerability_information`, `severity_rating`, `impact`, and `weakness_id`. Attach screenshots as report assets.

## Phase 7 — All-Channel Broadcast

### Real-Time Finding Alert

For each confirmed finding, broadcast to all configured channels (Discord, Telegram, WhatsApp, iMessage, Signal, etc.):

```
BUG FOUND — CLAW-2026-001
Platform:   HackerOne / example.com
Type:       SQL Injection
Severity:   CRITICAL (CVSS 9.8)
Endpoint:   /api/search?q=
WAF:        Cloudflare (bypassed)
Evidence:   [screenshot attached]
Status:     Report submitted
```

### Final Run Summary

At the end of the run, broadcast a summary with total findings by severity, WAF status, number of reports filed, duration, and link to the full report file.

## Config

```json5
{
  env: {
    HACKERONE_USERNAME: "your_username",
    HACKERONE_API_TOKEN: "your_api_token",
    BUGCROWD_EMAIL: "your_email",
    BUGCROWD_PASSWORD: "stored_in_credentials",
    OBB_USERNAME: "your_username",
  },
}
```

## Rate Limiting

All tools default to **1 RPS**. Override per-run if the program explicitly permits higher rates. Never exceed program-specified rate limits.

> [!CAUTION]
> Getting banned means losing future earnings. Default 1 RPS is safe for all platforms. Never set above 5 RPS without explicit program permission.

## Safety Rules

1. **Scope check before every request** — verify the target is IN_SCOPE before testing
2. **No destructive tests** — no account deletion, no data destruction, no DoS payloads
3. **No PII access beyond proof** — stop at confirmation, do not bulk-exfiltrate
4. **Rate limit: 1 RPS default** — always enforced
5. **Report before disclosing** — submit to platform before sharing externally
6. **Validate tool availability** — confirm all required binaries are installed before starting
7. **Log all actions** — maintain audit trail of every request sent to target
