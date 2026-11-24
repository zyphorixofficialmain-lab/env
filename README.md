# env
# mrhiddenbbhh

A production-grade command line tool for auditing your own web servers for exposed `.env` files. Built for authorized security reviews, deployment verification, and continuous infrastructure checks.

mrhiddenbbhh scans one or more domains you own, probes a large library of common `.env` paths, analyzes any file it finds, scores the risk, and produces reports in multiple formats.

Version: 3.0.0

---

## Table of contents

1. What it does
2. Legal and ethical use
3. Features
4. Requirements
5. Installation
6. Quick start
7. Command line reference
8. File formats
9. Reports
10. State and diff mode
11. Webhook integration
12. Exit codes
13. How the risk score works
14. Recommended deployment hardening
15. FAQ
16. License

---

## 1. What it does

mrhiddenbbhh performs the following steps for every target domain:

1. Expands wildcards such as `*.example.com` into concrete hosts.
2. Probes a configurable list of candidate paths (default list ships with 60+ common `.env` locations).
3. Fetches each path with a bounded concurrency pool and per-host rate limit.
4. Validates that the response body actually looks like a dotenv file (rejects HTML, SPA fallbacks, JSON errors).
5. Parses every `KEY=VALUE` line and computes:
   - total key count
   - secret-looking keys (PASS, SECRET, KEY, TOKEN, ...)
   - critical keys (DB_PASS, JWT_SECRET, AWS_SECRET, ...)
6. Assigns a severity level (low, medium, high, critical) from the extracted keys.
7. Persists findings to an optional SQLite database to enable diffing between scans.
8. Emits a summary and optional reports in JSON, CSV, ENV, or HTML.

mrhiddenbbhh never writes to the target server. It only issues GET requests.

---

## 2. Legal and ethical use

mrhiddenbbhh is strictly for systems you own or have written authorization to test.

The `--i-own-this` flag is mandatory. Without it the tool refuses to run.

Do not use mrhiddenbbhh against third-party infrastructure. Unauthorized access to computer systems is a criminal offense in most jurisdictions, including under the Computer Fraud and Abuse Act (US), the Computer Misuse Act (UK), and equivalent statutes elsewhere.

If you are performing a client engagement, keep a signed scope document and a copy of mrhiddenbbhh scan logs as evidence of authorization.

---

## 3. Features

- Async engine with `aiohttp`, threaded fallback via standard library
- Bounded concurrency with per-host token bucket rate limiter
- Retry with exponential backoff
- Wildcard target expansion (`*.domain.tld`)
- Custom wordlist support for extra paths
- Custom HTTP header injection
- Content sniffing that rejects HTML error pages and SPA fallbacks
- SHA256 fingerprinting to deduplicate identical files across hosts
- Severity scoring based on parsed keys
- Safe-key whitelist so public keys are not flagged as secrets
- Secret masking by default with opt-in reveal
- Multiple report formats (JSON, CSV, ENV, HTML)
- Structured JSON logging to stderr
- SQLite state database for scan history and diff between runs
- Auto backup of existing report files before overwrite
- Graceful shutdown on SIGINT and SIGTERM
- Dry-run mode with no network requests
- Webhook notification for Slack and Discord
- SSL verification skip option for self-signed staging servers
- Timeout and byte cap per response

---

## 4. Requirements

- Python 3.8 or newer
- `aiohttp` (optional but strongly recommended for concurrency)
- Internet access to the targets you own

The tool uses only the Python standard library when `aiohttp` is not available, in which case it runs in sequential sync mode.

---

## 5. Installation

Clone or copy `mrhiddenbbhh.py` into a working directory.

Install the optional async dependency:

```bash
python -m pip install aiohttp
```

Make the script executable:

```bash
chmod +x mrhiddenbbhh.py
```

Verify:

```bash
python mrhiddenbbhh.py --help
```

---

## 6. Quick start

Scan a single domain:

```bash
python mrhiddenbbhh.py --url https://my-site.example --i-own-this
```

Scan multiple domains from a file:

```bash
python mrhiddenbbhh.py --urls-file domains.txt --i-own-this
```

Full report suite:

```bash
python mrhiddenbbhh.py \
    --urls-file domains.txt \
    --i-own-this \
    --output-json report.json \
    --output-csv report.csv \
    --output-env report.env \
    --output-html report.html
```

Preview without making any requests:

```bash
python mrhiddenbbhh.py --urls-file domains.txt --i-own-this --dry-run
```

---

## 7. Command line reference

```
--url TEXT                 Single target URL. Scheme optional (https assumed).
--urls-file PATH           Text file with one URL per line. Lines starting with # are ignored.
--i-own-this               Required. Confirms you own or are authorized to test the target.
--paths TEXT               Comma separated custom path list. Replaces the default list.
--wordlist PATH            Extra path list loaded from a file, appended to defaults.
--timeout INT              Per request timeout in seconds. Default: 10.
--concurrency INT          Maximum in-flight requests per host. Default: 20.
--sync                     Force sequential sync mode even if aiohttp is installed.
--show-secrets             Print unmasked values to the terminal and reports.
--ignore-ssl               Skip TLS certificate verification. Use for staging only.
--headers TEXT             Custom request headers. Format: "Key:Value,Key:Value".
--output-json PATH         Write a JSON report to this path.
--output-csv PATH          Write a CSV report to this path.
--output-env PATH          Append an ENV-format report to this path.
--output-html PATH         Write an HTML report to this path.
--state-db PATH            SQLite database for scan history and diff mode.
--webhook URL              POST a JSON summary to this URL after the scan.
--dry-run                  Show what would be scanned without making requests.
```

Flags can be combined. For example:

```bash
python mrhiddenbbhh.py \
    --url https://a.example \
    --urls-file domains.txt \
    --i-own-this \
    --wordlist extra.txt \
    --headers "X-Audit:mrhiddenbbhh" \
    --output-html report.html \
    --state-db state.sqlite \
    --webhook https://hooks.slack.com/services/XXX
```

---

## 8. File formats

### Target file

A plain text file, one URL per line, UTF-8:

```
# mrhiddenbbhh targets
https://example.com
https://staging.example.com
*.wildcard.example.org
example4.com
```

Lines beginning with `#` and blank lines are ignored. Wildcards are expanded automatically.

### Wordlist file

A plain text file, one path per line, UTF-8:

```
.env.ci
.env.preview
credentials.env
secrets/.env
deploy/.env
```

The wordlist is appended to the default candidate list, then deduplicated.

---

## 9. Reports

### JSON

Contains a summary object and one entry per target. Designed for programmatic consumption.

```json
{
  "tool": "mrhiddenbbhh",
  "version": "3.0.0",
  "summary": {
    "targets": 3,
    "requests": 180,
    "findings": 2,
    "by_severity": { "critical": 1, "high": 1, "medium": 0, "low": 0 }
  },
  "results": [
    {
      "base": "https://example.com",
      "findings": [
        {
          "url": "https://example.com/.env",
          "path": ".env",
          "status": 200,
          "size": 412,
          "sha256": "abc123...",
          "severity": "critical",
          "severity_score": 74,
          "key_count": 14,
          "secret_keys": ["DB_PASS", "MAIL_PASSWORD"],
          "critical_keys": ["DB_PASS", "JWT_SECRET"],
          "content_preview": "APP_NAME=..."
        }
      ]
    }
  ]
}
```

### CSV

One row per finding. Suitable for spreadsheet import and ticket tracking.

Columns: `base, url, path, status, size, sha256, severity, severity_score, key_count, secret_keys, critical_keys, timestamp`

### ENV

Appendable. One section per target, then one section per finding with full content. Useful for offline audit trails.

### HTML

Self-contained dark theme report with sortable severity colouring. No external assets. Safe to email or archive.

---

## 10. State and diff mode

Pass `--state-db` to record every finding and scan in a local SQLite file:

```bash
python mrhiddenbbhh.py --urls-file domains.txt --i-own-this \
    --state-db mrhiddenbbhh_state.sqlite
```

On subsequent runs, mrhiddenbbhh computes:

- **new** findings not seen in previous scans
- **resolved** findings that are no longer exposed

This is useful for tracking remediation progress over time. The database never leaves your machine.

---

## 11. Webhook integration

mrhiddenbbhh can POST a JSON summary to a webhook URL after every scan.

Example payload:

```json
{
  "tool": "mrhiddenbbhh",
  "findings_total": 3,
  "by_severity": { "critical": 2, "high": 1, "medium": 0, "low": 0 },
  "targets": 5
}
```

Slack example:

```bash
python mrhiddenbbhh.py --urls-file domains.txt --i-own-this \
    --webhook https://hooks.slack.com/services/T000/B000/XXXX
```

Discord example:

```bash
python mrhiddenbbhh.py --urls-file domains.txt --i-own-this \
    --webhook https://discord.com/api/webhooks/XXXX/YYYY
```

Webhook failures are logged to stderr and do not abort the scan.

---

## 12. Exit codes

| Code | Meaning |
|------|---------|
| 0    | Scan completed, no exposed files found |
| 1    | Scan completed, at least one exposed file found |
| 2    | Invalid invocation (missing `--i-own-this`, no targets, missing file) |
| 130  | Interrupted by SIGINT |

Exit codes make mrhiddenbbhh easy to wire into CI:

```bash
python mrhiddenbbhh.py --urls-file domains.txt --i-own-this \
    --output-json ci-report.json
if [ $? -ne 0 ]; then
    echo "env exposure detected, failing build"
    exit 1
fi
```

---

## 13. How the risk score works

For every detected `.env` file, mrhiddenbbhh calculates a numeric score:

```
score = min(key_count, 30)
      + (number of secret_looking_keys * 5)
      + (number of critical_keys * 15)
```

Severity thresholds:

| Score | Severity |
|-------|----------|
| >= 60 | critical |
| >= 35 | high   |
| >= 15 | medium |
| >= 0  | low    |

Keys in the safe whitelist (`APP_NAME`, `APP_ENV`, `LOG_LEVEL`, and similar) are never counted as secrets. This prevents a harmless `.env.example` from being rated critical.

Critical key patterns include `DB_PASS`, `DATABASE_URL`, `JWT_SECRET`, `AWS_SECRET`, `SMTP_PASS`, `STRIPE_SECRET`, `TWILIO_AUTH`, and similar.

---

## 14. Recommended deployment hardening

An exposed `.env` is almost always a deployment misconfiguration, not an application bug. Use the following checklist to eliminate the risk at the source.

### 14.1 Keep `.env` outside the web root

```
project/
  public/          <- only this directory is served
  app/
  config/
  database/
  storage/
  .env             <- never under public/
```

### 14.2 Block dotfiles at the web server level

Nginx:

```nginx
location ~ /\.(?!well-known) {
    deny all;
    return 404;
}
```

Apache `.htaccess` at the document root:

```apache
<FilesMatch "^\.">
    Require all denied
</FilesMatch>
```

### 14.3 Add a CI gate

Run mrhiddenbbhh after every deploy:

```yaml
- name: mrhiddenbbhh scan
  run: |
    python mrhiddenbbhh.py --urls-file domains.txt --i-own-this \
        --output-json deploy-report.json
```

Fail the pipeline on exit code 1.

### 14.4 Rotate credentials immediately if anything is found

If mrhiddenbbhh reports an exposed `.env`, treat it as a live incident:

1. Rotate every credential listed in the file (database, mail, third-party APIs)
2. Review access logs for the exposed path over the last 90 days
3. Notify the affected services
4. Fix the deployment so the file is no longer served

---

## 15. FAQ

**Does mrhiddenbbhh upload anything to a remote server?**

No. All requests go from your machine to the target. No data is sent to any mrhiddenbbhh server because no such server exists.

**Why does the tool refuse to run without `--i-own-this`?**

To prevent accidental misuse against infrastructure you do not control.

**Why did mrhiddenbbhh return 0 when the site clearly has a `.env` file?**

The file may be properly protected by the web server (403 or 404), or it may be an `.env.example` with no secrets, which the tool correctly classifies as safe.

**Can it detect `.env` files behind basic authentication?**

No. mrhiddenbbhh performs unauthenticated probing only, which is what an opportunistic attacker would do. If the file requires auth, it is not publicly exposed.

**Does it support IPv6 targets?**

Yes. Pass the literal IPv6 address in brackets:

```bash
python mrhiddenbbhh.py --url "http://[2001:db8::1]" --i-own-this
```

**What happens if a target is down?**

The tool logs the network error to stderr, increments the failure counter, and continues with the next target. It does not abort the scan.

**Are reports written incrementally?**

The ENV report appends. JSON, CSV, and HTML reports are written at the end of the scan. Existing files are backed up with a timestamp suffix before overwrite.

**Can I run this from a Docker container?**

Yes. Base it on `python:3.12-slim`, install `aiohttp`, copy `mrhiddenbbhh.py`, and run with the same CLI.

---

## 16. License

mrhiddenbbhh is released under the MIT License.

You may use, modify, and redistribute it freely. No warranty is provided. The author is not responsible for any misuse or for any damages resulting from scanning systems you do not own.

By using mrhiddenbbhh you agree that:

- You will only run it against systems you own or have written authorization to test
- You will not use it to obtain, store, or disclose credentials belonging to third parties
- You will rotate any credential discovered during an authorized scan

---

mrhiddenbbhh
