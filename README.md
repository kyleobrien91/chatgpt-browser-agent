# ChatGPT Browser Agent

Use an existing authenticated `chatgpt.com` browser tab as a conversational backend for MCP-compatible agents.

The project consists of two layers:

- **MCP server** — deterministic browser/session control through Playwright.
- **Agent Skill** — teaches an agent how to conduct, evaluate and iterate a multi-turn conversation with ChatGPT until a user-defined goal is achieved.

## Documentation

- [Product Requirements Document](docs/PRD.md)
- [Technical Implementation](docs/TECHNICAL-IMPLEMENTATION.md)
- [Roadmap](docs/ROADMAP.md)

## Planned MCP surface

- `chatgpt_status`
- `chatgpt_tabs`
- `chatgpt_select_tab`
- `chatgpt_new_conversation`
- `chatgpt_send`
- `chatgpt_wait`
- `chatgpt_read`

## Design principle

The **agent owns reasoning and goal evaluation**. The **MCP owns browser mechanics and ChatGPT UI interaction**. ChatGPT owns conversational context.

## Security boundary

The project uses the user's existing authenticated browser session. It does not automate login, expose credentials or cookies, bypass CAPTCHA, or circumvent access controls.

## Status

Documentation-first. Implementation follows the roadmap in `docs/ROADMAP.md`.
