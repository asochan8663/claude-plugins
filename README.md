# @asochan-ai — Claude Code Skills

Production-grade skills for quality, security, and workflow automation.

**Works on 35+ platforms**: Claude Code, Codex CLI, Cursor, Gemini CLI, GitHub Copilot, and any tool that reads SKILL.md.

## Quick Start

```bash
# Add this marketplace
/plugin marketplace add asochan8663/claude-plugins

# Install skills
/plugin install auto-test@asochan-ai
/plugin install quality-rules@asochan-ai
/plugin install pre-check@asochan-ai
/plugin install security-audit@asochan-ai
```

## Skills

| Skill | What it does |
|-------|-------------|
| **auto-test** | Auto-detect test frameworks (pytest, vitest, jest, go test, cargo test) and run tests |
| **quality-rules** | Enforce quality rules during implementation — write it right the first time |
| **pre-check** | Map blast radius, quality rules, and past incidents before starting any task |
| **security-audit** | 3-stage security audit: pattern scan + dependency check + AI attacker analysis |

## Philosophy: Zero Leak

These skills implement a layered defense system:

```
L1 pre-check    → Map the battlefield before you start
L2 quality-rules → Build it right the first time
L3 (your tests)  → Catch what slipped through
```

## License

MIT
