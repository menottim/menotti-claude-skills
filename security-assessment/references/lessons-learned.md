# Lessons Learned from Prior Assessment Runs

> These lessons were generated through the self-improvement protocol after real assessment runs. They inform methodology improvements and tool workarounds. Read this file when adapting the methodology for a new target or when troubleshooting assessment tooling issues.

---

## v1.3.0 Assessment: Python/FastAPI/Docker Application (2026-03-05)

### Additions

1. **[DOCKER] Entrypoint privilege dropping** -- Add explicit test: `docker exec <container> whoami`. The v1.3.0 assessment found that `gosu` was installed but never invoked in the entrypoint. This was the highest-impact finding (Medium) but was NOT in the original pre-identified concerns list.

2. **[VALIDATION] Password in error responses** -- Add test: send malformed login request (e.g., `{"username": 123, "password": "real_password"}`) and check if the response `input` field contains the password. Pydantic's `RequestValidationError` includes raw input by default.

3. **[DOCKER] read_only/cap_drop as separate test** -- Distinguish between "commented out for compatibility" (documented choice) vs. "missing entirely" (oversight). Both are findings, but the former is lower severity.

4. **[SSRF] `allow_local` scope analysis** -- When testing SSRF with `allow_local=True`, test cloud metadata (169.254.x.x) SEPARATELY from private ranges (10.x, 172.16.x, 192.168.x). The `allow_local` flag may bypass the entire blocklist, not just private ranges.

5. **[CRYPTO] Decrypt failure behavior** -- Add test: what happens when `decrypt_if_needed()` encounters a value encrypted with a different key? Silent fallback to ciphertext is a finding (masks key rotation problems).

6. **[WEBSOCKET] Connection limit test** -- Attempt to open 100+ WebSocket connections and measure server memory impact.

7. **[CONFIG] Webhook URL SSRF** -- Config import may validate instance URLs but not webhook URLs. Test both independently.

### Modifications

1. **[Phase 2] Agent permissions** -- Agents dispatched for active testing may be sandboxed from running Bash commands against Docker. **Mitigation**: Either (a) run tests directly in the main session, or (b) pre-run Docker setup commands and provide cookie jar / auth tokens to agents explicitly, or (c) launch agents with clear instructions that they MUST use Bash for curl commands.

2. **[Phase 2] Static-only fallback** -- When active testing is blocked by tool permissions, the assessment can still provide high-value findings via comprehensive static code review. A static review agent found 15 findings (3 Medium) without running any live commands.

3. **[SSRF severity] `allow_local` bypass** -- Confirmed it bypasses ALL blocked networks when enabled. Upgrade from investigation-needed to confirmed Medium.

4. **[Registration race]** -- Downgrade from investigation concern to Low/Info when the dashboard endpoint already has a post-commit check and rate limiting (3/hour) makes practical exploitation infeasible.

### Tool Workarounds

1. **[Bash in sub-agents]** Sub-agents may be denied Bash tool access depending on permission settings. Workaround: run Docker setup and curl commands in the main session, then dispatch agents for code review (Read/Grep only). Pass authentication cookies as explicit strings in agent prompts.

2. **[WebSocket testing]** Python `websockets` library may not be installed in the assessment environment. Workaround: use `python3 -c "import websockets"` to check availability first. If unavailable, test WebSocket auth via static code review of the relevant handler file.

---

## Screen-Capture Application Assessment (2026-03-06)

This assessment led to the creation of Phase 0.5 (Architecture & Design Review).

### Additions

1. **[ARCHITECTURE] Phase 0.5** -- The 5 most impactful findings were design-level issues that no code-level grep or endpoint test would have found. Added as a mandatory phase with 7 categories.

2. **[AI/LLM] Indirect prompt injection to XSS** -- When LLM output is rendered as HTML via `v-html` or f-string interpolation, the LLM's visual input becomes an XSS injection vector.

3. **[AI/LLM] Prompt injection propagation chains** -- When prior LLM outputs are fed back as context for subsequent LLM calls, a single adversarial input can contaminate a chain of outputs.

4. **[PRIVACY] Claims-vs-reality analysis** -- Projects that claim "privacy-first" may contradict those claims (CDN loading, configurable remote endpoints).

5. **[ARCHITECTURE] Data volume estimation** -- Calculating storage accumulation over the default retention period makes "unencrypted data at rest" findings concrete.

6. **[ARCHITECTURE] Analogous system comparison** -- Comparing to known incidents (e.g., Windows Recall) framed the severity and identified missing controls.

7. **[FILING] Private vulnerability reporting** -- GitHub's private vulnerability reporting must be enabled by the repo owner. The `gh api repos/OWNER/REPO/security-advisories/reports` endpoint is correct for filing as a non-maintainer.

8. **[FILING] Comprehensive + individual advisories** -- Filing one comprehensive advisory plus individual advisories per finding provides both big picture and per-finding tracking.

### Modifications

1. **[Phase 0] Non-Docker/non-git targets** -- For PyPI packages: download with `pip download --no-deps`, unzip wheel, create test venv. Phase 0.5 does NOT require a running target.

2. **[SEVERITY] Localhost-only services** -- Use CVSS Modified Attack Vector = Local for localhost-bound services, but still report findings.

3. **[Phase 1] Architecture informs code review** -- Phase 1 agent prompts should reference Phase 0.5 findings to focus code review.

### Tool Workarounds

1. **[socketio]** Install with: `pip install "python-socketio[client]"`
2. **[zsh]** Quote all curl URLs containing query parameters (zsh glob expansion).
3. **[Private vuln reporting]** May return 403 if not enabled. Ask maintainer to enable in Settings > Security > Private vulnerability reporting.
4. **[Timing analysis]** Running 100+ timed requests from `curl` can be slow. Workaround: reduce to 20 samples per group (valid/invalid username). Statistical significance requires ~20+ samples for Argon2 timing (~200ms per hash).

---

## Process Insights (from v1.3.0 fix + closeout cycle)

These are lessons about the full lifecycle: assessment -> filing -> fixing -> closeout.

1. **Advisories must be published, not just created.** Draft advisories are invisible to the public and don't trigger GitHub's dependency alert system. After fixes land, PATCH each advisory to `state: "published"` with `patched_versions` set.

2. **Issues need fix-to-commit mapping.** When closing a batched issue (multiple findings in one issue), the close comment should map each finding to its fix commit hash. This creates an audit trail from finding -> fix -> verification.

3. **`gh api` requires `--input` for nested JSON.** The `-f` flag approach doesn't work for Security Advisories because the `vulnerabilities` field requires nested JSON objects. Always use `--input - <<'JSON'` with a heredoc.

4. **Docker `read_only: true` has cascading effects.** Enabling `read_only` can break entrypoint scripts. Fixes that enable security hardening must be tested end-to-end in Docker, not just verified in code review. Include a "Docker rebuild + smoke test" step after Docker-related fixes.

5. **`docker exec whoami` is misleading.** It runs a new shell as root (Docker default), not as the application process user. To verify privilege dropping, check `/proc/1/status` for the actual UID: `docker exec <container> cat /proc/1/status | grep Uid`.

6. **Subagent-driven fixes are fast for isolated changes.** Provide the exact code context (file, line numbers, before/after) in the agent prompt. Don't make the agent discover the code.

7. **Assessment -> Plan -> Fix -> Closeout is the right sequence.** Don't start fixing during the assessment (context pollution). Don't file before the assessment is complete (may duplicate). Don't skip the closeout (advisories stay as invisible drafts).

8. **Record advisory IDs and issue numbers immediately after filing** so they're available for the closeout step without re-querying the API.
