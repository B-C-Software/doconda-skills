# Doconda skills

[![Validate](https://github.com/B-C-Software/doconda-skills/actions/workflows/validate.yml/badge.svg)](https://github.com/B-C-Software/doconda-skills/actions/workflows/validate.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

The [Agent Skill](https://docs.doconda.com/en/skill) for [Doconda](https://doconda.com), the API that creates, reviews
and edits Word, PowerPoint, Excel and PDF documents for apps and AI agents. With it installed, your coding agent
(Claude Code, Cursor, Codex…) writes Doconda integrations correctly: which request to send, how to check the result,
what each error means and what not to do.

## What your agent learns

- One endpoint for everything (`POST /v1/documents`): from a sentence or from your Markdown, to `docx`, `pdf`, `pptx`,
  `xlsx` or an `artifact` (a web page with its own data).
- How to check every result: `status`, `style_unsupported` and the report, instead of assuming it worked.
- Quality levels and their time and price, copying a template's design, files and sources, retention, links that
  expire, idempotency, streaming, webhooks and errors (`DocondaError` codes).
- How to give Doconda to an AI agent: MCP, or `doconda.tools()` for function calling.

The full reference (endpoints, request fields, error, review and event codes) is in
[`skills/doconda/reference.md`](skills/doconda/reference.md).

## Install

Any agent that supports skills:

```bash
npx skills add B-C-Software/doconda-skills
```

Claude Code, as a plugin (the skill plus Doconda's MCP server):

```
/plugin marketplace add B-C-Software/doconda-skills
/plugin install doconda@doconda
```

The MCP server reads your key from `DOCONDA_API_KEY` (`ak_eu_…`, from the Doconda dashboard; new accounts get free
credit). To add only the MCP server to Cursor, Codex or another client, see
[MCP server](https://docs.doconda.com/en/agents/mcp).

By hand: copy `skills/doconda` into `~/.claude/skills/` (all projects) or `.claude/skills/` (one project).

## Try it

Ask your agent, for example:

- "Add an endpoint that turns this invoice JSON into a PDF with Doconda."
- "Our LLM writes reports in Markdown: deliver them as Word files with our brand colours."
- "Review every .docx in `contracts/` with Doconda and list the issues it can't fix."
- "Give my support agent a tool to create documents with Doconda."

## Links

- [Docs](https://docs.doconda.com/en) · [TypeScript SDK](https://www.npmjs.com/package/@doconda/sdk) ·
  [Python SDK](https://pypi.org/project/doconda/) · [MCP server](https://www.npmjs.com/package/@doconda/mcp)
- [Runnable examples](https://github.com/B-C-Software/doconda-examples) for Bedrock, Vertex AI, OpenAI, Claude,
  LangGraph, the Vercel AI SDK, Mastra and more

## Contributing

This repository is published from Doconda's main repository, so changes are made there. Found something wrong or out of
date? [Open an issue](https://github.com/B-C-Software/doconda-skills/issues).

## License

[MIT](LICENSE)
