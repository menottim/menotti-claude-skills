# Design: Architecture & Design Security Review Phase

**Date:** 2026-03-06
**Context:** Lessons from chronometry-ai assessment — the most impactful findings (VULN-18 through VULN-22) came from architectural analysis, not code review or pentesting.

## Problem

The current methodology has three phases: static code review, active penetration testing, and regression verification. All three focus on *implementation* — finding bugs in code and verifying them at runtime.

The chronometry-ai assessment revealed that the most critical findings were *design-level*:
- URL credentials captured and stored for 3 years without sanitization (architectural data flow issue)
- LLM output trusted as safe HTML (trust boundary issue)
- Prior annotations fed back into prompts creating injection chains (data flow feedback loop)
- Timeline HTML embedding full-res base64 screenshots creating 800 MB exfiltration packages (data amplification)
- Privacy claims contradicted by CDN loading (claims-vs-reality gap)
- Comparison to Windows Recall framed the severity and missing controls (analogous system analysis)

None of these would be found by grepping for `innerHTML` or testing SSRF bypasses.

## Design

### New Phase: "Phase 0.5: Architecture & Design Review"

Sits between Phase 0 (Environment Setup) and Phase 1 (Static Code Review). Mandatory — not skippable. Produces findings that feed into Phase 1 review priorities and generates its own architecture-level findings in the report.

### Updated Flow

```
Phase 0:   Setup environment
Phase 0.5: Architecture & design review  ← NEW
Phase 1:   Static code review (informed by 0.5)
Phase 2:   Active penetration testing
Phase 3:   Regression verification
```

### Phase 0.5 Categories

#### 1. Data Lifecycle Trace
- Map every data type that enters, is created by, or flows through the system
- For each: source, format, sensitivity level, storage location, encryption status, retention, access controls, API exposure
- Produce a data flow diagram (ASCII art in the report)
- Estimate storage volume over time (per day, per year, at max retention)
- Identify data amplification points (e.g., base64 encoding inflating size)

#### 2. Trust Boundary Analysis
- Identify all trust boundaries: user input → app, LLM output → app, config → runtime, network → app, filesystem → app
- For each boundary: what is trusted implicitly that shouldn't be?
- Key question: "If this input were adversarial, what could an attacker achieve?"
- Specific to LLM-integrated apps: is LLM output treated as trusted? Can it influence control flow or rendered HTML?

#### 3. Threat Modeling (STRIDE-lite)
- Lightweight STRIDE pass against the architecture — not a full threat model document
- For each component: can it be Spoofed? Can data be Tampered? Is there Repudiation risk? Information Disclosure? Denial of Service? Elevation of Privilege?
- Focus on generating questions for Phase 1, not exhaustive analysis
- Output: prioritized list of "investigate in Phase 1" items

#### 4. Privacy & Claims Analysis
- Read README, FAQ, SECURITY.md, and any privacy documentation
- Extract every security/privacy claim the project makes
- Compare each claim against what the code actually does
- Document contradictions as findings (like CDN loading contradicting "nothing leaves your machine")
- Check scope exclusions in SECURITY.md — are they reasonable given the actual risk?

#### 5. Analogous System Comparison
- Identify systems that solve the same problem or have similar architecture
- Research known vulnerabilities, public incidents, or security research on those systems
- Map which known issues apply to the target
- Use WebSearch / WebFetch for this — look for blog posts, CVEs, security research
- Example: chronometry → Windows Recall → Beaumont's TotalRecall research → plaintext storage finding

#### 6. Supply Chain & Deployment
- Review dependency list: are all dependencies mainstream? Any obscure packages?
- CDN resources: are they loaded with integrity hashes?
- Auto-update mechanisms: can they be hijacked?
- Deployment configuration: launchd plists, systemd units, Docker configs, Kubernetes manifests
- Package provenance: is the PyPI/npm package built from the public repo? Any discrepancies?

#### 7. Data Volume & Accumulation
- Calculate storage growth rate based on default configuration
- Project storage at 1 day, 1 month, 1 year, max retention
- Identify what percentage is sensitive vs. non-sensitive
- Assess whether the retention default is proportionate to the use case
- Consider: what does an attacker get if they exfiltrate the entire data directory?

### Report Integration

The architecture review produces:
1. A dedicated **"Architecture & Design Security Analysis"** section in the report, placed before the findings table
2. Architecture-level findings (VULN-XX) in the same findings table, tagged with a `[Design]` prefix to distinguish from implementation bugs
3. A prioritized list of investigation items that feed into Phase 1 agent prompts

### Agent Dispatch

Phase 0.5 is partially parallelizable:
- **Categories 1-3** (data lifecycle, trust boundaries, STRIDE) require reading the codebase — dispatch as a single Explore agent or run in the main session
- **Category 4** (privacy & claims) can run in parallel — reads README/docs, compares against code
- **Category 5** (analogous systems) can run in parallel — uses WebSearch/WebFetch, no code reading needed
- **Categories 6-7** (supply chain, data volume) can run in parallel — focused scoping

### Severity Calibration for Design Findings

Design findings use a different severity rubric than implementation bugs:

| Design Issue | Typical Severity |
|-------------|-----------------|
| Sensitive data stored unencrypted with no access control | High-Critical (depending on data type) |
| Trust boundary violation (LLM output → HTML) | High |
| Privacy claim contradicted by code | Medium-Informational |
| Excessive default retention | Medium |
| Data amplification creating exfiltration packages | Medium |
| Missing analogous-system controls | Severity of the missing control |
| Fail-open safety mechanisms | Informational-Medium |

### Changes to methodology.md

1. Add Phase 0.5 section with all 7 categories, each with "what to investigate" guidance
2. Update the process flow diagram to include Phase 0.5
3. Add "Architecture & Design Review" to the Quick Reference table
4. Update the Output Format to include the architecture section before findings
5. Add Phase 0.5 categories to the Testing Coverage Matrix
6. Add architecture-specific entries to Red Flags table
7. Update the "Adapting This Methodology" section to include architecture mapping
8. Add to Self-Improvement Protocol: "Were there design-level findings that only emerged from architecture review?"

### Changes to SKILL.md

1. Update overview to mention four phases (not three)
2. Add Phase 0.5 to the process flow diagram
3. Add Phase 0.5 to the Quick Reference table
4. Add architecture-specific red flags
5. Update error handling for Phase 0.5 (e.g., "No README/docs available — skip claims analysis, note gap")
