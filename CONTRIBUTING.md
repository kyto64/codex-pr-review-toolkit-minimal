# Contributing

Thank you for improving this minimal PR review toolkit.

## Ways to Contribute

- Report bugs or unclear documentation via [GitHub Issues](https://github.com/kyto64/codex-pr-review-toolkit-minimal/issues)
- Suggest review-quality improvements to Skill instructions
- Fix packaging, attribution, or README accuracy

## Skill Changes

Review instructions live in `plugins/pr-review-toolkit/skills/*/SKILL.md`.

1. Edit the relevant `SKILL.md`
2. If you changed one of the six perspective skills, regenerate subagents:

```bash
python3 scripts/generate_subagents.py
```

3. Validate packaging:

```bash
python3 scripts/validate_plugin.py plugins/pr-review-toolkit
python3 scripts/generate_subagents.py --check
```

4. Open a pull request with a short summary of the review behavior change

`pr-review` is the orchestrator skill and is not generated into `.codex/agents/`.

## Scope

Please keep contributions aligned with the repository goals:

- PR review only
- No `install.sh`, hook runners, MCP auto-configuration, or sandbox bypass workflows
- Minimal diff; match existing tone and structure

See [AGENTS.md](./AGENTS.md) for maintainer conventions.

## License

By contributing, you agree that your contributions are licensed under the repository [Apache License 2.0](./LICENSE), consistent with upstream `pr-review-toolkit` adaptations documented in [THIRD_PARTY_NOTICES.md](./THIRD_PARTY_NOTICES.md).
