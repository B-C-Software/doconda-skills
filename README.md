# Doconda skills

Agent Skills for [Doconda](https://doconda.com), the API that creates, reviews and edits Word, PowerPoint, Excel and PDF
documents for apps and AI agents. With the skill installed, your coding agent (Claude Code, Cursor, Codex…) knows how to
use the Doconda API, the `@doconda/sdk` TypeScript SDK and the MCP server: which request to send, how to check the
result, what each error means.

## Install

Any agent that supports skills:

```bash
npx skills add B-C-Software/doconda-skills
```

Claude Code, as a plugin (the skill plus the Doconda MCP server):

```
/plugin marketplace add B-C-Software/doconda-skills
/plugin install doconda@doconda
```

The MCP server reads your key from `DOCONDA_API_KEY` (`ak_eu_…` or `ak_us_…`, from the dashboard).

Or by hand: copy `skills/doconda` into `~/.claude/skills/` (all projects) or `.claude/skills/` (one project).

## Contents

- `skills/doconda/SKILL.md`: how to integrate Doconda.
- `skills/doconda/reference.md`: endpoints, request fields, errors, review codes and events.

Docs: https://docs.doconda.com · MIT licence.
