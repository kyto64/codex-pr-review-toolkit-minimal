# Instructions

## Repository

This is a **minimal PR review skill pack** for Codex CLI. There is no install script, no autonomous runner, no hooks, and no MCP auto-configuration.

### Layout

| Path | Role |
|------|------|
| `plugins/pr-review-toolkit/skills/*/SKILL.md` | Source of truth for review instructions |
| `plugins/pr-review-toolkit/.codex-plugin/plugin.json` | Public plugin manifest (`interface` metadata) |
| `.agents/plugins/marketplace.json` | Local marketplace; `source.path` is `./plugins/pr-review-toolkit` |
| `.codex/agents/*.toml` | Generated subagent definitions (6 perspectives only) |
| `scripts/generate_subagents.py` | Regenerate `.codex/agents/` and `.codex/config.toml` from `SKILL.md` |
| `scripts/validate_plugin.py` | Validate plugin packaging |

Do not add `install.sh`, `.mcp.json`, hook runners, or `codex-autonomous` / `codex-account` artifacts.

## Subagents (issue #14)

**Decision (2026-06):** Adopt optional **parallel subagent mode** for `pr-review`, with sequential single-session review as the fallback.

- Custom agents live in `.codex/agents/` (generated from `plugins/pr-review-toolkit/skills/*/SKILL.md`)
- Regenerate after skill edits: `python3 scripts/generate_subagents.py`
- `pr-review` orchestrates spawns explicitly; Codex only spawns subagents when asked
- Perspective selection (#13) limits parallel spawns to control token cost
- Subagents inherit parent sandbox policy; do not set `sandbox_mode` to bypass

## Skills

| Skill | When to use |
|-------|-------------|
| `pr-review` | Unified PR review with P0–P3 findings |
| `code-reviewer` | Correctness, security, maintainability |
| `silent-failure-hunter` | Error handling and silent failures |
| `pr-test-analyzer` | Test coverage quality |
| `comment-analyzer` | Comment accuracy |
| `type-design-analyzer` | Type invariants and encapsulation |
| `code-simplifier` | Complexity review (no behavior change) |

## Validation

- Plugin manifest: `plugins/pr-review-toolkit/.codex-plugin/plugin.json` (includes required `interface` metadata)
- Skill UI metadata: `plugins/pr-review-toolkit/skills/<name>/agents/openai.yaml`
- Marketplace entry: `.agents/plugins/marketplace.json`
- Subagent TOML: `.codex/agents/*.toml` (generated; source of truth is each `SKILL.md`)
- Run validation:

```bash
python3 scripts/validate_plugin.py plugins/pr-review-toolkit
python3 scripts/generate_subagents.py --check
```

Requires Python 3 and PyYAML (`pip install pyyaml` if missing).

## Safety

- Do not add sandbox bypass or approval-skipping workflows
- Do not reintroduce `install.sh`, hook runners, or autonomous subagent runners
- Subagent TOML under `.codex/agents/` is allowed; keep `sandbox_mode` unset so the parent policy applies
