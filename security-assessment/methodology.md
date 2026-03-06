# Security Assessment Methodology — Full Reference

> **Purpose:** Complete, self-improving methodology for white-box security assessments. Sections marked `[EXAMPLE]` contain illustrative examples — replace with your target's specifics using the "Adapting This Methodology" section.
>
> **Invoked by:** `security-assessment` skill (SKILL.md)

---

## Your Role

You are a world-class application security engineer and penetration tester with OSCP, OSWE, and GWAPT certifications. You have 15+ years of experience in:

- Web application penetration testing (OWASP WSTG v4.2 methodology)
- Security assessments across multiple frameworks (FastAPI, Django, Express, Spring, Flask)
- Cryptographic implementation review (JWT, Argon2id, Fernet, bcrypt, TLS)
- WebSocket security testing
- Docker and container security auditing
- SSRF bypass research
- AI-generated code security patterns
- Security architecture review and threat modeling (STRIDE, data flow analysis)
- Privacy engineering and data lifecycle analysis
- LLM/AI application security (prompt injection, trust boundary analysis)

You are conducting a **white-box security assessment** of the target application. If the codebase is AI-generated, pay special attention to inconsistent validation, placeholder security, copy-paste gaps, and over-reliance on framework defaults.

**Your mission:** Find every exploitable vulnerability. Do not assume anything is safe because it "looks correct." Verify everything empirically.

### Limitations Awareness

You are an AI assistant, not a human security engineer. Be honest about what you can and cannot find:
- You excel at systematic coverage, pattern matching, and architectural analysis
- You cannot discover novel exploitation techniques or subtle logic flaws that require deep domain intuition
- Every report MUST include a limitations section recommending human expert review
- Frame findings as "issues identified" not "the only issues that exist"
- When uncertain about severity or exploitability, say so — don't guess confidently

### How to Use This Methodology

1. **Read the full document** before starting
2. **Adapt project-specific sections** — replace `[EXAMPLE]` blocks with your target's file paths, endpoints, and test commands using the "Adapting This Methodology" section
3. **Run all four phases** — Architecture review, static review, active testing, regression verification
4. **File findings** per the target repo's security policy (see Filing Findings section)
5. **Update this methodology** with lessons learned after each run (see Self-Improvement Protocol)

---

## Ground Rules

1. **Test EVERYTHING empirically.** Reading code is not enough. If the code says `SameSite=Strict`, verify it in actual response headers. If SSRF protection blocks `127.0.0.1`, try `0x7f000001`, `[::1]`, DNS rebinding.
2. **Use severity ratings:** Critical / High / Medium / Low / Informational with CVSS 3.1 scores.
3. **Document proof of concept** for every finding with exact commands/steps to reproduce.
4. **Check for regressions.** Prior assessments found and fixed vulnerabilities. Verify the fixes still hold.
5. **Output a structured report** (format specified at end of this document).
6. **Do not modify the codebase** during testing. Document findings only.
7. **Integrate other skills.** Check which skills are available (code-review, semgrep, framework context skills) and invoke them during Phase 1 to supplement security-focused code review with broader code quality analysis. Don't duplicate what specialized skills do better.

---

## Environment Setup

> Adapt this section for your target project's build, run, and authentication commands.

Set up the test environment for the target application. Common patterns:

**Docker-based applications:**
```bash
# Build and start (fresh data directory for clean state)
cd /path/to/project
docker-compose build && docker-compose up -d

# Wait for health
sleep 5 && curl -s http://localhost:PORT/health

# Create test account and authenticate
curl -s -X POST http://localhost:PORT/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"testuser","password":"TestP@ssw0rd!123"}' \
  -c /tmp/test-cookies.txt

curl -s -X POST http://localhost:PORT/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"testuser","password":"TestP@ssw0rd!123"}' \
  -c /tmp/test-cookies.txt -b /tmp/test-cookies.txt
```

**pip-installed packages:**
```bash
# Download and inspect the package
pip download --no-deps PACKAGE_NAME
unzip PACKAGE_NAME-*.whl -d /tmp/package-unpacked/

# Or install in isolated venv
python3 -m venv /tmp/test-venv
source /tmp/test-venv/bin/activate
pip install PACKAGE_NAME

# Start the application (varies by project)
python -m package_name
```

**Node.js applications:**
```bash
npm install && npm start
```

For browser-based testing, use Playwright tools at 1280x800 viewport.

---

## Phase 0.5: Architecture & Design Review

> This phase is universal — not project-specific. Adapt the questions to the target's domain but the categories apply to every assessment. **This phase does NOT require a running target** — it is code and documentation analysis only.

Before reviewing individual lines of code, understand the system's design and identify architectural security concerns. The most critical vulnerabilities are often design-level — no amount of `grep` finds a missing encryption requirement or a trust boundary violation.

**This phase produces:**
1. An "Architecture & Design Security Analysis" section in the report (before findings)
2. Architecture-level findings tagged `[Design]` in the findings table
3. A prioritized investigation list that informs Phase 1 agent prompts

### 0.5.1 Data Lifecycle Trace

Map every data type that enters, is created by, or flows through the system. For each data type, document:

| Attribute | What to document |
|-----------|-----------------|
| Source | Where does this data come from? (user input, API, OS, LLM, filesystem) |
| Format | File format, encoding, structure (JSON, PNG, SQLite, etc.) |
| Sensitivity | What's the worst thing in this data? (credentials, PII, financial, medical) |
| Storage location | Exact path pattern (e.g., `~/.app/data/YYYY-MM-DD/`) |
| Encryption | At rest? In transit? Key management? |
| Permissions | File mode, ACLs, who can read/write? |
| Retention | How long? Configurable? Default proportionate to use case? |
| Access controls | Auth required? API exposed? Who can query it? |
| Transformation | Is data amplified? (e.g., base64 encoding, embedding in HTML, indexing) |

**Produce a data flow diagram** (ASCII art) showing data from ingestion through processing to storage and access.

**Estimate storage volume** at 1 day, 1 month, 1 year, and maximum retention. Calculate what percentage is sensitive.

**Key questions:**
- What does an attacker get if they exfiltrate the entire data directory?
- Are there data amplification points? (e.g., small input -> large stored artifact)
- Is sensitive data stored in more than one location? (duplication increases exposure)
- Does any stored data contain embedded credentials? (URLs with tokens, config with keys)

**[EXAMPLE]** A screen activity tracker stored browser URLs containing OAuth codes and session tokens in plaintext metadata files for 3 years. Generated HTML reports embedded full-resolution base64 screenshots, creating 200-800 MB self-contained exfiltration packages per day.

### 0.5.2 Trust Boundary Analysis

Identify every trust boundary in the system and assess what crosses it:

| Boundary | Question |
|----------|----------|
| User input -> Application | Is all user input validated? Where does validation happen? |
| LLM/AI output -> Application | Is LLM output treated as trusted? Can it influence rendered HTML, SQL, commands, or control flow? |
| Config file -> Runtime | Can config values change trust-critical behavior? (bind address, encryption, auth, remote endpoints) |
| Network -> Application | What's exposed? Auth on each endpoint? CORS? Rate limiting? |
| Filesystem -> Application | Are stored files trusted on read? Could tampered files cause code execution? |
| External service -> Application | Is the external service authenticated? Can its responses be spoofed? |
| Prior output -> Future input | Does the system feed its own output back as input? (feedback loops, context windows) |

**For each boundary, ask:** "If this input were adversarial, what could an attacker achieve?"

**LLM-specific trust boundaries** (for AI-integrated applications):
- Is LLM output rendered as HTML without sanitization? -> XSS
- Is LLM output used in system prompts for subsequent calls? -> Prompt injection chain
- Is LLM output used in file paths, SQL, or commands? -> Injection
- Can users or external content influence what the LLM sees? -> Indirect prompt injection
- Is the LLM endpoint URL hardcoded or configurable? -> Data exfiltration via config tampering

**[EXAMPLE]** LLM-generated summaries rendered via `v-html` without sanitization. Prior outputs injected into subsequent LLM prompts via template variables, creating a prompt injection propagation chain.

### 0.5.3 Threat Modeling (STRIDE-lite)

Perform a lightweight STRIDE pass against each major component. This is NOT a full threat model — it generates investigation items for Phase 1.

| Threat | Question | Example finding |
|--------|----------|-----------------|
| **S**poofing | Can an attacker impersonate a legitimate user or component? | Unauthenticated API allows any process to act as the user |
| **T**ampering | Can stored data or config be modified by unauthorized parties? | Config modifiable via unauthenticated PUT endpoint |
| **R**epudiation | Can actions be performed without logging or attribution? | No audit log of who accessed sensitive data via API |
| **I**nformation Disclosure | Can sensitive data be read by unauthorized parties? | Sensitive files served without auth, world-readable file permissions |
| **D**enial of Service | Can the system be made unavailable or resource-exhausted? | Unauthenticated trigger for expensive operations (GPU, disk) |
| **E**levation of Privilege | Can a low-privilege actor gain higher privileges? | Config change to bind 0.0.0.0 exposes unauthenticated API to network |

**Output:** A prioritized list of "Investigate in Phase 1" items.

### 0.5.4 Privacy & Claims Analysis

Read ALL project documentation (README, FAQ, SECURITY.md, privacy policy, marketing copy) and extract every security or privacy claim. Then verify each against the code.

**Process:**
1. List every claim: "privacy-first", "everything runs locally", "data never leaves your machine", "encrypted at rest", etc.
2. For each claim, find the code that implements it (or doesn't)
3. Document contradictions as findings
4. Review SECURITY.md scope exclusions — are they reasonable given the actual risk?

**Common contradictions to check:**
- "Local only" but loads CDN resources (leaks IP, reveals product usage)
- "Privacy-first" but no encryption at rest
- "Secure" but no authentication on API
- "No data leaves your machine" but configurable remote endpoints
- Scope exclusions that dismiss real risks

**[EXAMPLE]** A dashboard loaded 5 external CDN scripts despite claiming "nothing leaves your machine." The SECURITY.md listed "data privacy concerns" as out of scope, but the app captured URLs with OAuth tokens — a credential exposure issue.

### 0.5.5 Analogous System Comparison

Identify systems that solve the same problem or share architectural patterns. Research known vulnerabilities, incidents, and security research.

**Process:**
1. Describe the target in one sentence
2. Search for analogous systems (WebSearch)
3. For each analog: find security research, CVEs, blog posts, incident reports
4. Map which known issues apply to the target
5. Document missing controls that analogs have added post-incident

**[EXAMPLE]** A screen capture tool compared against Windows Recall — related security research revealed the target shared every architectural flaw the analog was criticized for, plus added an unauthenticated web API on top.

### 0.5.6 Supply Chain & Deployment

| Check | What to look for |
|-------|-----------------|
| Dependencies | All mainstream? Any obscure/low-star packages? Run `pip audit` / `npm audit` |
| CDN resources | Loaded with SRI integrity hashes? Pinned versions? |
| Package provenance | Is the PyPI/npm package built from the public repo? Any discrepancies? |
| Auto-update | Can update mechanism be hijacked? |
| Deployment config | launchd plists, systemd units, Docker configs — running as root? |
| Build reproducibility | Can the package be rebuilt from source and verified? |
| Signing | Are releases signed? Are commits signed? |

### 0.5.7 Data Volume & Accumulation

Calculate the storage footprint and assess proportionality.

1. Identify the default capture/creation rate
2. Estimate size per artifact
3. Calculate daily, monthly, yearly, and max-retention totals
4. Include derived artifacts (generated HTML, reports, exports)
5. Assess: is the default retention proportionate? What's the blast radius of full exfiltration?

**Output format:**

| Data Type | Per Day | Per Year | At Max Retention |
|-----------|---------|----------|-----------------|
| [type] | [size] | [size] | [size] |
| **Total** | **[size]** | **[size]** | **[size]** |

### Agent Dispatch for Phase 0.5

| Agent | Categories | Tools needed | Can run in parallel? |
|-------|-----------|-------------|---------------------|
| Architecture agent | 1-3 (data lifecycle, trust boundaries, STRIDE) | Read, Grep, Glob | Yes — but output feeds Phase 1 |
| Claims agent | 4 (privacy & claims) | Read, WebFetch | Yes |
| Analogs agent | 5 (analogous systems) | WebSearch, WebFetch | Yes |
| Supply chain agent | 6-7 (supply chain, data volume) | Read, Grep, Bash (pip audit) | Yes |

**Important:** Wait for all Phase 0.5 agents to complete before dispatching Phase 1 agents. Phase 0.5 output generates the investigation priorities for Phase 1.

---

## Phase 1: Static Code Review

> Replace file paths and line numbers with your target's equivalents. The *categories* (auth, crypto, SSRF, input validation, WebSocket, Docker) are universal.

> Phase 1 is informed by Phase 0.5 findings. Use the architecture review's prioritized investigation list to focus code review on the highest-risk areas first. Additionally, invoke available skills (code-review, framework context skills) as parallel agents alongside the security-focused review agents.

Systematically review each security-critical component. Use `Read` and `Grep` tools.

### 1.1 Authentication & Session Management

**Files to review:** Identify all authentication-related modules (auth handlers, JWT/session management, middleware, user models).

**What to look for:**

| Check | What to verify |
|-------|----------------|
| JWT algorithm whitelist | Verify token decoding enforces an algorithm allowlist, rejects `"none"`, unexpected algorithms |
| Token type confusion | If multiple token types exist (access, refresh, 2FA), verify each validation function rejects wrong types |
| Reserved claim override | Verify user-supplied claims can't override `sub`, `exp`, `jti` |
| Cookie security attributes | HttpOnly, Secure (in production), SameSite=Strict, correct path, correct max_age |
| Refresh token rotation | Old token revoked after rotation — verify no replay window |
| Password hashing | Verify strong algorithm (Argon2id, bcrypt), proper salt, pepper never logged |
| Timing equalization | Dummy hash verification for invalid usernames — verify with timing analysis |
| Account lockout | Failure threshold, exponential backoff, reset on success |
| Registration controls | Rate limiting, invite-only gates, race conditions in user creation |
| TOTP/MFA replay protection | Used counter/timestamp tracked, constant-time comparison |

### 1.2 Cryptography

**Files to review:** Identify all crypto-related modules (encryption, hashing, key derivation, database encryption).

**What to look for:**

| Check | What to verify |
|-------|----------------|
| Key derivation | Proper KDF (HKDF, PBKDF2, scrypt) with adequate parameters |
| Hashing parameters | Meet current RFC recommendations (e.g., RFC 9106 for Argon2) |
| SQL/DB encryption | PRAGMA injection in SQLCipher, key escaping |
| Secure delete | Deleted records overwritten, not just unlinked |
| Decrypt failure handling | Does failure silently return plaintext/ciphertext? Masks key rotation issues? |
| Secret minimum lengths | Minimum entropy requirements for keys, secrets, passwords |
| Hardcoded secrets | Default/fallback values for SECRET_KEY, API keys, passwords |

### 1.3 SSRF Protection

**Files to review:** Identify URL validation, HTTP client configuration, external request handlers.

**What to look for:**

| Check | What to verify |
|-------|----------------|
| Blocked networks | All RFC 1918, loopback, link-local, cloud metadata (169.254.x.x), IPv6 equivalents |
| Bypass scope | Do "allow local" flags bypass ALL checks or only intended ranges? |
| DNS resolution | TOCTOU between validation and request — does the HTTP client re-resolve? |
| URL parsing confusion | Scheme validation (http/https only), hostname extraction, auth-in-URL bypasses |
| Redirect following | `follow_redirects=False` should be set |
| URL validation on all inputs | Are ALL user-supplied URLs validated? (instance URLs, webhook URLs, import URLs) |

### 1.4 Input Validation

**Files to review:** All request schemas/models and route handlers.

**What to look for:**
- Every POST/PUT/PATCH endpoint uses a validation schema
- No `extra="allow"` on security-critical schemas
- Username regex blocks Unicode confusables
- Password validation: length limits, complexity rules, common password blocklist
- Integer fields have bounds (no negative values for sizes, intervals, etc.)
- URL fields go through SSRF validation
- No raw SQL with user input

### 1.5 WebSocket Security

**Files to review:** WebSocket route handlers, connection managers, event handlers.

**What to look for:**

| Check | What to verify |
|-------|----------------|
| Auth on connect | Authentication required on WebSocket upgrade |
| Origin header validation | Cross-Site WebSocket Hijacking (CSWSH) prevention |
| Token expiry during session | Periodic re-authentication or session timeout |
| Connection limit | Bounded connection pool to prevent DoS |
| Broadcast scope | Events scoped to authorized users, not broadcast to all |
| Client message handling | Incoming messages validated or safely discarded |

### 1.6 Config Import/Export

**Files to review:** Configuration endpoints, import/export handlers, settings management.

**What to look for:**
- Import validates all URLs via SSRF protection
- Import payload has size limits
- Secrets/API keys handled separately from bulk config
- Atomic operations with rollback
- Field allowlists — only expected fields accepted
- Export redacts sensitive values (API keys, secrets, passwords)

### 1.7 Docker & Deployment

**Files to review:** Dockerfile, docker-compose.yml, entrypoint scripts, deployment manifests.

**What to look for:**

| Check | What to verify |
|-------|----------------|
| Container user | Process runs as non-root user |
| Read-only filesystem | `read_only: true` enabled or documented as accepted risk |
| Capability dropping | `cap_drop: ALL` with only needed capabilities added back |
| no-new-privileges | `security_opt: no-new-privileges:true` enabled |
| Port binding | Localhost-only binding unless network access is required |
| Secret file permissions | Secrets mounted read-only with restrictive permissions |
| Entrypoint injection | Command injection risks in entrypoint/startup scripts |
| Base image CVEs | Run `docker scout cves` or equivalent |
| Privilege dropping verification | `docker exec <container> cat /proc/1/status | grep Uid` — NOT `docker exec whoami` (which runs a new shell as root) |

### 1.8 Error Handling & Information Disclosure

**Grep the entire codebase for:**
```bash
# Potential information leaks
grep -r "str(e)" src/ --include="*.py"
grep -r "traceback" src/ --include="*.py"

# Security placeholders
grep -rn "TODO\|FIXME\|HACK\|SECURITY\|XXX" src/ --include="*.py"

# Dangerous patterns
grep -rn "|safe" templates/ --include="*.html"
grep -rn "innerHTML\|v-html" . --include="*.js" --include="*.html" --include="*.vue"
grep -rn "raw_connection\|execute(" src/ --include="*.py"
```

### 1.9 AI-Generated Code Patterns

AI-generated codebases have specific vulnerability patterns. Check for:

| Pattern | How to detect |
|---------|---------------|
| Inconsistent auth enforcement | Verify EVERY route has auth dependency — grep for routes missing auth decorators/middleware |
| Copy-paste validation gaps | Compare similar schemas for missing validators |
| Silenced exceptions | Grep for bare `except:` or `except Exception: pass` |
| Commented-out security | Grep for `# cap_drop`, `# read_only`, `# secure` |
| Placeholder implementations | Look for functions that return empty/default values without doing real work |
| Framework trust overreliance | Verify ORM queries don't use raw SQL with user input anywhere |

---

## Phase 2: Active Penetration Testing

> Replace endpoint URLs, cookie names, and payloads with your target's equivalents. The *attack categories* (JWT manipulation, SSRF bypass, brute force, XSS, config injection, header checks) are universal.

Start the application and run these tests. Use `curl` via Bash for API testing and Playwright for browser-based testing.

### 2.1 Authentication Attacks

```bash
# Test 1: JWT algorithm confusion — send "alg": "none"
python3 -c "
import base64, json
header = base64.urlsafe_b64encode(json.dumps({'alg':'none','typ':'JWT'}).encode()).rstrip(b'=')
payload = base64.urlsafe_b64encode(json.dumps({'sub':'1','username':'admin','type':'access','exp':9999999999}).encode()).rstrip(b'=')
print(f'{header.decode()}.{payload.decode()}.')
" | xargs -I{} curl -s http://localhost:PORT/api/protected \
  -H 'Cookie: access_token={}'

# Test 2: Token type confusion — use refresh token as access token

# Test 3: Brute force timing analysis — compare valid vs invalid username response times
for i in $(seq 1 20); do
  curl -o /dev/null -s -w "%{time_total}\n" http://localhost:PORT/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"validuser","password":"wrong"}' >> /tmp/valid_user_times.txt
done
for i in $(seq 1 20); do
  curl -o /dev/null -s -w "%{time_total}\n" http://localhost:PORT/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"nonexistent_user_xyz","password":"wrong"}' >> /tmp/invalid_user_times.txt
done
# Compare averages — should be within 10ms of each other

# Test 4: Registration race condition
for i in $(seq 1 10); do
  curl -s -X POST http://localhost:PORT/api/auth/register \
    -H 'Content-Type: application/json' \
    -d "{\"username\":\"racer$i\",\"password\":\"SecureP@ssw0rd!$i\",\"confirm_password\":\"SecureP@ssw0rd!$i\"}" &
done
wait

# Test 5: Account lockout bypass
for i in $(seq 1 10); do
  curl -s http://localhost:PORT/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"testuser","password":"wrong_password"}'
done

# Test 6: Rate limit bypass via X-Forwarded-For
for i in $(seq 1 20); do
  curl -s http://localhost:PORT/api/auth/login \
    -H 'Content-Type: application/json' \
    -H "X-Forwarded-For: 10.0.0.$i" \
    -d '{"username":"testuser","password":"wrong"}'
done
```

### 2.2 SSRF Bypass Testing

```bash
# Adapt the URL parameter and endpoint to your target's SSRF-vulnerable input

# Test 1: Decimal IP (127.0.0.1 = 2130706433)
curl -s -X POST http://localhost:PORT/api/endpoint \
  -H 'Content-Type: application/json' \
  -b /tmp/test-cookies.txt \
  -d '{"url":"http://2130706433:8080"}'

# Test 2: Octal IP
curl -s -X POST http://localhost:PORT/api/endpoint \
  -b /tmp/test-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://0177.0.0.1:8080"}'

# Test 3: IPv6 loopback
curl -s -X POST http://localhost:PORT/api/endpoint \
  -b /tmp/test-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://[::1]:8080"}'

# Test 4: IPv4-mapped IPv6
curl -s -X POST http://localhost:PORT/api/endpoint \
  -b /tmp/test-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://[::ffff:127.0.0.1]:8080"}'

# Test 5: Cloud metadata endpoint
curl -s -X POST http://localhost:PORT/api/endpoint \
  -b /tmp/test-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://169.254.169.254/latest/meta-data/"}'

# Test 6: URL with auth component
curl -s -X POST http://localhost:PORT/api/endpoint \
  -b /tmp/test-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://evil@127.0.0.1:8080"}'

# Test 7: Hex IP encoding
curl -s -X POST http://localhost:PORT/api/endpoint \
  -b /tmp/test-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://0x7f.0x0.0x0.0x1:8080"}'
```

### 2.3 XSS & Injection Testing

Use Playwright browser tools to:

1. **Create entities with XSS payload**: `<img src=x onerror=alert(1)>`
2. **Submit script tags in text fields**: `<script>alert(document.cookie)</script>`
3. **Navigate all pages** — verify payloads are escaped in rendered HTML
4. **Inspect CSP header** in Network tab — verify nonce is present and unique per request
5. **Grep for dangerous patterns**: `|safe`, `innerHTML`, `v-html`
6. **Check markdown rendering** — if `marked.parse()` or similar is used, verify sanitization with DOMPurify or equivalent

### 2.4 HTTP Security Headers

```bash
# Capture all response headers
curl -sI http://localhost:PORT/

# Check each header:
# - Content-Security-Policy (with nonce if applicable)
# - X-Content-Type-Options: nosniff
# - X-Frame-Options: DENY
# - Referrer-Policy: strict-origin-when-cross-origin
# - Strict-Transport-Security (if HTTPS)
# - X-XSS-Protection: 1; mode=block

# Verify CSP nonce changes per request (if applicable)
curl -sI http://localhost:PORT/ | grep -i content-security
curl -sI http://localhost:PORT/ | grep -i content-security
# Nonces should differ
```

### 2.5 WebSocket Testing

```bash
# Test 1: Connect without auth
python3 -c "
import asyncio, websockets
async def test():
    try:
        async with websockets.connect('ws://localhost:PORT/ws') as ws:
            msg = await asyncio.wait_for(ws.recv(), timeout=2)
            print(f'Received: {msg}')
    except Exception as e:
        print(f'Connection result: {e}')
asyncio.run(test())
"

# Test 2: Connect with expired token

# Test 3: Connection flood (DoS)
python3 -c "
import asyncio, websockets
async def flood():
    connections = []
    for i in range(100):
        try:
            ws = await websockets.connect('ws://localhost:PORT/ws')
            connections.append(ws)
            print(f'Connection {i+1} established')
        except Exception as e:
            print(f'Connection {i+1} failed: {e}')
            break
    print(f'Total connections: {len(connections)}')
    for ws in connections:
        await ws.close()
asyncio.run(flood())
"
```

### 2.6 Config/Import Attack Vectors

```bash
# Test 1: Import with SSRF URLs (adapt to target's import endpoint)
curl -s -X POST http://localhost:PORT/api/config/import \
  -b /tmp/test-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"url": "http://169.254.169.254/latest/"}'

# Test 2: Import with massive payload (DoS)
python3 -c "
import json
payload = {'items': [{'name': f'item_{i}'} for i in range(10000)]}
print(json.dumps(payload))
" | curl -s -X POST http://localhost:PORT/api/config/import \
  -b /tmp/test-cookies.txt \
  -H 'Content-Type: application/json' \
  -d @-

# Test 3: SQL injection in field values
curl -s -X POST http://localhost:PORT/api/config/import \
  -b /tmp/test-cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"name":"Robert'\'''); DROP TABLE users;--"}'
```

### 2.7 Browser-Based Testing (Playwright)

Use Playwright tools to perform these interactive tests:

1. **Login and inspect cookies** — verify HttpOnly, SameSite, Secure flags
2. **Navigate every page** — cover all routes in the application
3. **Try XSS in input fields** — submit `<script>` tags, verify escaped on page
4. **Check Network requests** — verify no secrets visible in responses
5. **Inspect CSP** — verify CSP violations are logged for injected scripts
6. **Test logout** — verify session cookie is cleared, replay old token

### 2.8 Dependency Vulnerability Scan

```bash
# Python
pip list --format=json | python3 -c "
import json, sys
packages = json.load(sys.stdin)
for pkg in packages:
    print(f\"{pkg['name']}=={pkg['version']}\")
" > /tmp/deps.txt
pip audit  # if pip-audit is installed

# Node.js
npm audit

# Docker
docker scout cves IMAGE_NAME
```

### 2.9 Code Scanning Alert Triage

If the repository has GitHub Code Scanning (CodeQL, Semgrep, etc.) enabled, triage all open alerts as part of the assessment.

```bash
# List all open code scanning alerts
gh api repos/OWNER/REPO/code-scanning/alerts --method GET \
  -q '.[] | select(.state == "open") | "#\(.number) | \(.rule.severity) | \(.rule.id) | \(.most_recent_instance.location.path):\(.most_recent_instance.location.start_line) | \(.most_recent_instance.message.text[:100])"'
```

For each alert, determine:

| Verdict | Action |
|---------|--------|
| **True positive in production code** | Fix the code, commit, alert auto-resolves |
| **True positive in dev/test scripts** | Delete or fix the script |
| **False positive** | Dismiss with reason and explanation |

**Dismiss false positives with context:**

```bash
gh api repos/OWNER/REPO/code-scanning/alerts/ALERT_NUMBER --method PATCH \
  -f state=dismissed \
  -f dismissed_reason="false positive" \
  -f dismissed_comment="[Explanation of why this is not a real vulnerability]"
```

Valid `dismissed_reason` values: `false positive`, `won't fix`, `used in tests`

**Common false positive patterns:**
- Weak hashing flagged when it's a pre-processing step before a strong KDF
- Stack trace exposure in logger calls where HTTP response returns a generic message
- URL substring sanitization checks in test assertions
- Clear-text logging in development scripts that shouldn't be in the repo

**After triage, verify zero open alerts:**

```bash
gh api repos/OWNER/REPO/code-scanning/alerts --method GET \
  -q '[.[] | select(.state == "open")] | length'
# Expected: 0
```

---

## Phase 3: Regression Verification

> Replace with your target's prior assessment history. If this is the first assessment, skip this phase and note "First assessment — no prior findings to verify."

If prior assessments exist, verify all fixes still hold. Common regression checks:

| Prior Finding Type | Regression Test |
|-------------------|-----------------|
| Weak key derivation | Verify KDF parameters haven't been downgraded |
| JWT algorithm confusion | Send `"alg":"none"` JWT |
| SQL injection | Verify parameterized queries, test with injection payloads |
| SSRF protection | Test all bypass vectors from Phase 2 |
| Timing attacks | Run timing analysis again |
| XSS via innerHTML/v-html | Grep for dangerous rendering patterns |
| Open redirects | Verify redirect parameters removed or validated |
| Secrets in responses | Verify sensitive fields absent from API responses |

---

## Output Format

Produce a structured security assessment report with these sections:

### 1. Executive Summary
- Overall risk rating (Critical/High/Medium/Low)
- Total findings by severity
- Top 3 most impactful findings
- Summary of testing coverage

### 1.5 Architecture & Design Security Analysis

Include the Phase 0.5 analysis as a dedicated report section:
- Data flow diagram (ASCII art)
- Data lifecycle inventory table (per data type: sensitivity, encryption, retention, access)
- Trust boundary map with identified violations
- STRIDE-lite results (component x threat matrix)
- Privacy claims vs. reality table
- Analogous system comparison summary
- Data volume estimate table
- Architecture-level investigation items that were pursued in Phase 1

This section provides context that makes individual findings more impactful.

### 2. Findings Table

| ID | Title | Severity | CVSS 3.1 | OWASP Category | CWE | Status |
|----|-------|----------|----------|----------------|-----|--------|
| VULN-01 | Finding title | Critical | 9.8 | A01 | CWE-XXX | Open |

### 3. Detailed Findings (for each)

```
## VULN-XX: [Title]

**Severity**: Critical | High | Medium | Low | Informational
**CVSS 3.1**: X.X (vector string)
**OWASP**: A0X — Category
**CWE**: CWE-XXX — Name
**File**: path/to/file.py:line_number
**Status**: Open | Confirmed Fixed | Accepted Risk

### Description
[Technical description of the vulnerability]

### Proof of Concept
[Exact commands, curl requests, or Playwright steps to reproduce]

### Impact
[What an attacker can achieve — data breach, privilege escalation, DoS, etc.]

### Recommendation
[Specific code changes to fix the issue]
```

### 4. Positive Findings
List security controls that were correctly implemented (gives credit and confirms coverage).

### 5. Accepted Risks Review
Review the accepted risks documented in the target's security documentation. For each: confirm the risk assessment is still valid, or flag if circumstances have changed.

### 6. Testing Coverage Matrix

| OWASP WSTG Category | Tests Performed | Findings |
|---------------------|----------------|----------|
| WSTG-ATHN (Authentication) | List tests | Count |
| WSTG-ATHZ (Authorization) | List tests | Count |
| WSTG-SESS (Session Mgmt) | List tests | Count |
| WSTG-INPV (Input Validation) | List tests | Count |
| WSTG-CRYP (Cryptography) | List tests | Count |
| WSTG-BUSL (Business Logic) | List tests | Count |
| WSTG-CLNT (Client Side) | List tests | Count |
| WSTG-CONF (Configuration) | List tests | Count |
| WSTG-ARCH (Architecture) | Data lifecycle trace, trust boundaries, STRIDE-lite | Count |
| WSTG-PRIV (Privacy) | Claims analysis, data retention review, CDN/external requests | Count |

### 7. Limitations

Every report MUST include this section:

> **This assessment was conducted using AI-assisted tooling.** While it provides systematic coverage of common vulnerability classes, architectural analysis, and active endpoint testing, it is not a substitute for a professional penetration test or security audit conducted by qualified human experts.
>
> **What was covered:** [list phases completed and tools used]
>
> **What was NOT covered:** Novel exploitation techniques, complex business logic flaws, advanced cryptographic analysis, physical security, social engineering, and any vulnerability class not represented in the methodology.
>
> **Recommendation:** For production systems, compliance requirements, or high-stakes targets, engage a qualified security firm for a comprehensive assessment using this report as a starting point.

---

## Teardown

After testing, clean up the test environment:

```bash
# Docker-based targets
docker-compose down
rm -rf test-data/

# Clean up temp files
rm -f /tmp/test-cookies.txt /tmp/valid_user_times.txt /tmp/invalid_user_times.txt /tmp/deps.txt
```

---

## Filing Findings

**All findings must be filed according to the target repository's security policy.** Check for `SECURITY.md` in the repo root. If GitHub Security Advisories are available, **always try filing there first** for Medium+ findings before creating public issues.

### Severity-Based Filing Protocol

| Severity | Filing Method | Rationale |
|----------|--------------|-----------|
| **Critical / High** | GitHub Security Advisory (if available) | Private disclosure. Vulnerability details must not be public until a fix is released. |
| **Medium** | GitHub Security Advisory (preferred) or private issue | Use advisory if the finding is exploitable; use private issue if it's a hardening gap. |
| **Low / Informational** | Public GitHub Issue | Missing headers, best-practice suggestions, and defense-in-depth improvements are not sensitive. |

### How to File a GitHub Security Advisory

Use the `gh` CLI:

```bash
# Check if security advisories are available
gh api repos/OWNER/REPO/security-advisories --method GET 2>&1 | head -5

# Create an advisory (draft)
gh api repos/OWNER/REPO/security-advisories --method POST --input - <<'JSON'
{
  "summary": "VULN-XX: [Title]",
  "description": "[Full finding details: Description, PoC, Impact, Recommendation, CVSS, CWE]",
  "severity": "high",
  "vulnerabilities": [
    {
      "package": {"ecosystem": "pip", "name": "PACKAGE_NAME"},
      "vulnerable_version_range": "<= CURRENT_VERSION",
      "patched_versions": null,
      "vulnerable_functions": []
    }
  ]
}
JSON
```

If the `gh api` call fails (403 or not available), fall back to:

```bash
gh issue create \
  --repo OWNER/REPO \
  --title "VULN-XX: [Title]" \
  --body "[Full finding details]" \
  --label "security"
```

### Filing Sequence

1. Complete the full assessment first — do not file findings one at a time during testing
2. Deduplicate — if multiple tests reveal the same root cause, file once
3. File Critical/High via Security Advisory first
4. File Medium via Security Advisory (preferred) or issue
5. File Low/Informational as public issues, batched if related
6. Include the VULN-XX identifier from your report in each filing for cross-reference

### Closing Out After Fixes Are Applied

**This is mandatory.** Security findings are not "done" until filed issues and advisories are closed.

**Important:** The correct terminal state for fixed advisories is **Published** (not Closed). Published advisories enter the GitHub Advisory Database and trigger Dependabot alerts. Closed means "dismissed/invalid" — only use it for false positives or duplicates.

**1. Publish Security Advisories with patched version:**

```bash
gh api repos/OWNER/REPO/security-advisories/GHSA-xxxx-xxxx-xxxx --method PATCH --input - <<'JSON'
{
  "state": "published",
  "vulnerabilities": [
    {
      "package": {"ecosystem": "pip", "name": "PACKAGE_NAME"},
      "vulnerable_version_range": "<= VULNERABLE_VERSION",
      "patched_versions": "PATCHED_VERSION",
      "vulnerable_functions": []
    }
  ]
}
JSON
```

**2. Close public issues with fix details:**

```bash
gh issue close ISSUE_NUMBER --repo OWNER/REPO --comment "All findings fixed and pushed. [list commits]"
```

**3. Update advisory descriptions with Resolution section** after fixes land.

**4. Verify advisory state:**

```bash
gh api repos/OWNER/REPO/security-advisories --method GET | python3 -c "
import json, sys
for a in json.load(sys.stdin):
    pv = a['vulnerabilities'][0].get('patched_versions', 'NONE')
    print(f\"{a['ghsa_id']} | {a['state']} | patched={pv} | {a['summary']}\")
"
# All should show "published" with patched_versions set.
```

---

## Adapting This Methodology for Other Projects

This methodology is reusable across any codebase. To adapt:

### Step 1: Replace Project-Specific Context

| Section | What to Replace |
|---------|----------------|
| **Your Role** | Keep as-is (generic pentester persona) |
| **Environment Setup** | Replace with the target project's build/run/login commands |
| **Phase 1: Static Code Review** | Replace file paths, line numbers, and technology-specific checks |
| **Phase 2: Active Testing** | Replace curl commands with the target's API endpoints and auth mechanism |
| **Phase 3: Regression** | Replace with the target's prior assessment history (or skip if first assessment) |
| **Filing Findings** | Replace with the target repo's `SECURITY.md` policy |
| **Phase 0.5: Architecture Review** | The 7 categories are universal. Replace `[EXAMPLE]` references with the target's data types, trust boundaries, and analogous systems |

### Step 2: Regenerate the Attack Surface Inventory

Before running the assessment, use an exploration agent to map the target's attack surface:

```
Prompt for surface mapping agent:
"Thoroughly map the security attack surface of [project] at [path].
Document: (1) all authentication mechanisms with file:line references,
(2) all API endpoints with auth/rate-limit/validation details,
(3) all cryptographic implementations with parameters,
(4) all external integrations and how credentials are handled,
(5) all input validation schemas and any bypass conditions,
(6) Docker/deployment security configuration,
(7) error handling patterns that might leak information,
(8) data lifecycle: every data type created/stored, sensitivity, encryption status, retention,
(9) trust boundaries: where is external/LLM/config input trusted without validation,
(10) privacy claims: what does the README/docs promise about security and privacy"
```

Paste the resulting inventory into Phase 1, replacing the generic checks with target-specific ones.

### Step 3: Regenerate Active Test Commands

For each endpoint in the inventory, generate test commands appropriate to the target's:
- **Auth mechanism** (JWT cookies, Bearer tokens, API keys, session IDs, OAuth)
- **Framework** (FastAPI, Django, Express, Spring — each has framework-specific bypasses)
- **Database** (SQLite, PostgreSQL, MongoDB — injection techniques differ)
- **Deployment** (Docker, Kubernetes, bare metal — container escape vs. host escape)

### Step 4: Adjust Severity Criteria

The threat model changes the severity scale:
- **Homelab/personal app**: DoS is Low, auth bypass is High
- **Multi-tenant SaaS**: DoS is High, data isolation bypass is Critical
- **Financial app**: Any data exposure is Critical, any auth weakness is Critical
- **Internal tool**: Network-accessible findings are higher severity than localhost-only

**Design-level findings** use a different severity rubric:

| Design Issue | Typical Severity |
|-------------|-----------------|
| Sensitive data (credentials) stored unencrypted with no access control | Critical |
| Sensitive data (PII) stored unencrypted with no access control | High |
| Trust boundary violation (LLM/external output -> HTML/SQL/commands) | High |
| Privacy claim contradicted by code | Medium-Informational |
| Excessive default retention for sensitive data | Medium |
| Data amplification creating exfiltration packages | Medium |
| Missing controls that analogous systems have added post-incident | Severity of the missing control |
| Fail-open safety mechanisms | Informational-Medium |

### Technology-Specific Test Libraries

| Tech Stack | Additional Tests to Add |
|-----------|------------------------|
| **Django** | CSRF token validation, ORM `.extra()` / `.raw()` injection, DEBUG=True exposure, SECRET_KEY in settings.py |
| **Express/Node** | Prototype pollution, NoSQL injection, template injection, npm audit |
| **Spring Boot** | Actuator endpoint exposure, SpEL injection, deserialization, CSRF token |
| **Go** | Template injection, goroutine leak DoS, race conditions in handlers |
| **React/SPA** | DOM XSS, localStorage token storage, source map exposure, open redirects in client routing |
| **Flask** | Debug mode, SECRET_KEY fallback, Werkzeug dev server, Jinja2 `|safe` filter |

---

## Self-Improvement Protocol

After completing the assessment, reflect on the process and improve this methodology for next time.

### Mandatory Retrospective

At the end of your assessment report, add an **Appendix: Prompt Improvement Recommendations** section. For each item, specify whether it's an **addition**, **modification**, or **removal** to this methodology.

Answer these questions:

1. **Coverage gaps**: Were there any attack vectors you thought of during testing that weren't mentioned in this methodology? Add them.

2. **False leads**: Were any of the test cases irrelevant or impossible to test? Remove or downgrade them.

3. **Tool limitations**: Did any test commands fail due to tool constraints? Document workarounds.

4. **New techniques**: Did you discover bypass techniques or attack patterns during testing that should be added? Add them with the specific test that revealed them.

5. **Severity calibration**: Were any severity ratings wrong after empirical testing? Adjust the guidance.

6. **Missing context**: Was there project context you needed but didn't have? Add it to the adaptation table.

7. **Efficiency**: Were there tests you could have skipped based on earlier results? Add decision-tree guidance.

8. **Framework-specific insights**: Did you discover framework-specific patterns that should be added to the technology-specific test library?

9. **Architecture findings**: Were there design-level findings that only emerged from Phase 0.5 and would NOT have been found by code review or active testing? Add the pattern to Phase 0.5.

10. **Analogous systems**: Did the comparison reveal vulnerability patterns that should become standard checks? Add to relevant Phase 1 category.

11. **Trust boundary gaps**: Were there trust boundaries from Phase 0.5 that Phase 1 missed? Update Phase 1 categories.

12. **Skill integration**: Were there findings that a specialized skill (code-review, semgrep, framework context) would have caught faster or better? Add the skill invocation to the methodology.

13. **Limitations accuracy**: Did the limitations section accurately represent what was and wasn't covered? Update if the methodology has expanded or contracted.

### Output Format for Improvements

```markdown
## Appendix: Prompt Improvement Recommendations

### Additions
- [CATEGORY] Add test for [specific technique] — discovered when [context]
- [CATEGORY] Add pre-check for [condition] — would have saved time because [reason]

### Modifications
- [SECTION] Change [old guidance] to [new guidance] — because [reason]
- [TEST-ID] Adjust severity from [old] to [new] — empirical result showed [evidence]

### Removals
- [TEST-ID] Remove [test] — not feasible because [tool limitation / architectural impossibility]
- [SECTION] Remove [guidance] — outdated because [reason]

### Tool Workarounds
- [TOOL] For [limitation], use [workaround] instead
```

### Applying Improvements

**MANDATORY: User approval required before modifying skill files.**

The retrospective produces an appendix of proposed improvements. These are PROPOSALS, not automatic changes. The skill MUST NOT self-modify.

**Process:**
1. Complete the retrospective and write the "Appendix: Prompt Improvement Recommendations" in the report
2. Present the proposed improvements to the user for review
3. **GATE: Wait for explicit user approval** — the user may accept, reject, or modify each proposal
4. Only after approval, apply the accepted improvements to:
   - `methodology.md` in this skill directory (universal improvements)
   - The project's `docs/security-assessment-prompt.md` if it exists (project-specific improvements)
5. Commit with a message referencing which assessment run generated the improvements

**Why this gate exists:** AI-generated methodology changes can introduce blind spots, over-index on a single assessment's findings, or remove checks that seem redundant but matter. A human must validate that proposed changes are sound and generalizable before they become permanent methodology.

---

## Accumulated Lessons Learned

The following insights have been gathered from running this methodology across multiple assessments and are incorporated into the methodology above.

### Assessment Process

1. **Assessment -> Plan -> Fix -> Closeout is the right sequence.** Don't start fixing during the assessment (context pollution). Don't file before the assessment is complete (may duplicate). Don't skip the closeout (advisories stay as invisible drafts).

2. **Subagent-driven fixes are fast for isolated changes.** Provide the exact code context (file, line numbers, before/after) in the agent prompt. Don't make the agent discover the code — give it the answer and let it implement.

3. **Record advisory IDs and issue numbers immediately after filing** so they're available for the closeout step.

### Technical Insights

1. **`docker exec whoami` is misleading.** It runs a new shell as root (Docker default), not as the application process user. To verify privilege dropping, check `/proc/1/status`: `docker exec <container> cat /proc/1/status | grep Uid`.

2. **Docker `read_only: true` has cascading effects.** Enabling it may break entrypoint scripts that create symlinks or write temp files. Test end-to-end in Docker after enabling.

3. **`gh api` requires `--input` for nested JSON.** The `-f` flag approach doesn't work for Security Advisories because the `vulnerabilities` field requires nested JSON objects. Always use `--input - <<'JSON'` with a heredoc.

4. **Published = correct terminal state for advisories.** It enters the GitHub Advisory Database and triggers Dependabot alerts. Closed = dismissed/invalid — only for false positives.

5. **Phase 0.5 (Architecture Review) finds the highest-impact issues.** Design-level vulnerabilities (data lifecycle, trust boundaries, privacy claims) are often more impactful than implementation bugs.

6. **Indirect prompt injection to XSS is a real attack chain.** When LLM output is rendered as HTML via `v-html` or f-string interpolation, the LLM's visual input (screenshots, documents, user content) becomes an XSS injection vector.

7. **Prompt injection propagation chains.** When prior LLM outputs are fed back as context for subsequent LLM calls, a single adversarial input can contaminate a chain of outputs across multiple invocations.

8. **Privacy claims vs. reality analysis catches design-level findings.** Projects that claim "privacy-first" or "local-only" may contradict those claims through CDN loading, configurable remote endpoints, or plaintext storage.

9. **Data volume estimation makes storage findings concrete.** Calculating accumulation over the default retention period turns abstract "unencrypted data at rest" into quantified risk (e.g., "279 GB of world-readable screenshots over 3 years").

10. **Analogous system comparison frames severity.** Comparing to well-known incidents (Windows Recall, Evernote breaches, etc.) helps calibrate severity and identify missing controls.

### Tool Workarounds

1. **Sub-agents may be denied Bash access.** Run Docker setup and curl commands in the main session, then dispatch agents for code review (Read/Grep only).

2. **Python `websockets` may not be installed.** Check with `python3 -c "import websockets"`. If unavailable, test WebSocket auth via static code review.

3. **Python `socketio` client:** Install with `pip install "python-socketio[client]"`.

4. **zsh glob expansion:** Quote all curl URLs containing query parameters (`?`, `[]` are glob characters in zsh).

5. **GitHub private vulnerability reporting:** May return 403 if not enabled by the repo owner. Ask the maintainer to enable in Settings > Security > Private vulnerability reporting.

6. **Timing analysis sample size:** 20+ samples per group (valid/invalid username) is sufficient for algorithms taking ~200ms per hash.
