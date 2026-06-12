# Security Policy

This repository ships review-only Codex Skills and plugin metadata. It does not run a hosted service, execute user code autonomously, or configure MCP servers.

## Reporting a Vulnerability

If you find a security issue in this repository (for example, unsafe instructions in a Skill, packaging that could mislead users into bypassing sandbox policy, or accidental secret exposure), please report it via [GitHub Issues](https://github.com/kyto64/codex-pr-review-toolkit-minimal/issues).

For sensitive reports, you may also contact the maintainer through GitHub instead of opening a public issue.

## Scope

In scope:

- Skill instructions that encourage sandbox bypass, approval skipping, or unsafe automation
- Plugin packaging or documentation that could lead to unintended environment changes
- Committed secrets, tokens, or private logs

Out of scope:

- Findings produced by `$pr-review` on your own codebases
- Issues in upstream [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) unrelated to this adaptation

## Supported Versions

Security fixes apply to the latest release on the default branch. This project is pre-1.0; patch releases may include documentation or Skill instruction fixes.
