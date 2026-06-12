# Third-Party Notices

This file clarifies attribution for public distribution. The repository license is [Apache License 2.0](./LICENSE).

## Anthropic `pr-review-toolkit` (Derivative Work)

Six review perspective skills are adapted from [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) `plugins/pr-review-toolkit`:

| Local Skill | Upstream Source |
|-------------|-----------------|
| `plugins/pr-review-toolkit/skills/code-reviewer/` | `plugins/pr-review-toolkit/agents/code-reviewer.md` |
| `plugins/pr-review-toolkit/skills/code-simplifier/` | `plugins/pr-review-toolkit/agents/code-simplifier.md` |
| `plugins/pr-review-toolkit/skills/comment-analyzer/` | `plugins/pr-review-toolkit/agents/comment-analyzer.md` |
| `plugins/pr-review-toolkit/skills/pr-test-analyzer/` | `plugins/pr-review-toolkit/agents/pr-test-analyzer.md` |
| `plugins/pr-review-toolkit/skills/silent-failure-hunter/` | `plugins/pr-review-toolkit/agents/silent-failure-hunter.md` |
| `plugins/pr-review-toolkit/skills/type-design-analyzer/` | `plugins/pr-review-toolkit/agents/type-design-analyzer.md` |

- **Upstream license:** Apache-2.0 ([upstream LICENSE](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/pr-review-toolkit/LICENSE))
- **Upstream NOTICE:** None (confirmed: no `NOTICE` file in upstream `plugins/pr-review-toolkit/`)
- **Local LICENSE:** Identical Apache-2.0 full text as upstream plugin LICENSE

Adaptations include converting Claude agent definitions to Codex `SKILL.md`, removing autonomous/subagent-runner dependencies, generalizing project-specific rules to `AGENTS.md`, and review-only constraints (no auto-fix, no sandbox bypass).

## Generated Derivative Artifacts

`.codex/agents/*.toml` files are **generated** by `scripts/generate_subagents.py` from the six adapted perspective `SKILL.md` files listed above. Each TOML embeds `developer_instructions` derived from the corresponding skill content (for example, `.codex/agents/code-reviewer.toml` from `plugins/pr-review-toolkit/skills/code-reviewer/SKILL.md`). They are derivative of the Anthropic-adapted skills, not original authored work. Regenerate after skill edits; do not edit by hand.

| Artifact | Source |
|----------|--------|
| `.codex/agents/*.toml` | Generated from adapted `plugins/pr-review-toolkit/skills/*/SKILL.md` (six perspectives) |

## Original Work in This Repository

The following are original to [kyto64/codex-pr-review-toolkit-minimal](https://github.com/kyto64/codex-pr-review-toolkit-minimal) and are also licensed under Apache-2.0 unless noted otherwise:

| Component | Notes |
|-----------|-------|
| `plugins/pr-review-toolkit/skills/pr-review/` | Orchestrator skill; not present in upstream |
| `plugins/pr-review-toolkit/.codex-plugin/plugin.json` | Codex plugin manifest and interface metadata |
| `plugins/pr-review-toolkit/skills/*/agents/openai.yaml` | Skill UI metadata for Codex |
| `.agents/plugins/marketplace.json` | Local marketplace entry |
| `.codex/config.toml` | Subagent configuration |
| `scripts/generate_subagents.py` | Generates `.codex/agents/` from adapted `SKILL.md` |
| `scripts/validate_plugin.py` | Plugin validation script |
| `docs/sample-review.md` | Sample review output |
| `README.md`, `AGENTS.md`, `EXTERNAL_SOURCES.md` | Repository documentation |
