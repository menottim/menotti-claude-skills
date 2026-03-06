# menotti-claude-skills

A collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for specialized workflows.

## Skills

### [security-assessment](./security-assessment/)

Four-phase AI-assisted security assessment methodology for white-box application security reviews.

**Phases:**
1. **Phase 0** — Environment setup
2. **Phase 0.5** — Architecture & design review (data lifecycle, trust boundaries, privacy claims, analogous systems)
3. **Phase 1** — Static code review (auth, crypto, injection, config, AI patterns)
4. **Phase 2** — Active penetration testing (XSS, SSRF, headers, WebSocket, dependency scan)
5. **Phase 3** — Regression verification of prior fixes

**Produces:** Structured findings with CVSS 3.1 scores, CWE references, OWASP WSTG coverage, and proof-of-concept commands.

**Important:** This is an AI-assisted first pass, not a replacement for professional human security review. See the skill's [limitations section](./security-assessment/SKILL.md#limitations--read-this-first).

## Installation

To use a skill with Claude Code, copy (or symlink) the skill directory into `~/.claude/skills/`:

```bash
# Clone the repo
git clone https://github.com/menottim/menotti-claude-skills.git

# Symlink the skill you want
ln -s "$(pwd)/menotti-claude-skills/security-assessment" ~/.claude/skills/security-assessment
```

Claude Code will discover the skill automatically via its `SKILL.md` frontmatter.

## License

MIT
