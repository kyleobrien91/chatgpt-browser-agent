# PRD: ChatGPT Browser Agent MCP

**Status:** Draft  
**Version:** 1.0  
**Target:** TypeScript / Node.js 20+  
**Browser:** Chromium-based browsers  
**Protocol:** Model Context Protocol (MCP)  
**Automation:** Playwright

## 1. Product Overview

### 1.1 Problem

MCP-compatible autonomous agents can use model APIs, but they generally cannot directly leverage a user's already-authenticated ChatGPT web session. This project bridges that gap by allowing an agent to operate the normal ChatGPT messaging interface in an existing browser tab.

### 1.2 Product

The **ChatGPT Browser Agent MCP** is an MCP server that lets an agent control an already-open `chatgpt.com` tab. A companion Agent Skill teaches the agent to:

1. establish a ChatGPT browser session;
2. give ChatGPT a user-defined goal;
3. wait for a complete response;
4. inspect the response;
5. determine whether the goal has actually been achieved;
6. formulate targeted follow-ups;
7. continue the conversation;
8. stop on verified completion, blockage, failure or iteration limits.

The intended flow is:

```text
User goal
   |
   v
Agent + Skill
   |
   | MCP
   v
ChatGPT Browser MCP
   |
   | Playwright
   v
Existing authenticated browser
   |
   v
chatgpt.com
   |
   v
ChatGPT
   |
   +---- response ----> Agent evaluates -> follow-up or completion
```

## 2. Goals

### G1 — Existing tab/session

The system MUST interact with an existing browser session and SHOULD reuse the user's authenticated ChatGPT session. It MUST NOT require the agent to obtain or manage the user's credentials.

### G2 — Multi-turn operation

The agent MUST be able to conduct multiple sequential turns in the same ChatGPT conversation while preserving ChatGPT's native context.

### G3 — Goal-oriented iteration

The agent MUST be able to inspect a response, identify missing or incorrect work, and issue corrective follow-ups until success criteria are met or execution terminates.

### G4 — Reliable completion detection

The MCP MUST distinguish active generation from completed generation and from error/blocked/disconnected states. Fixed sleeps MUST NOT be the sole completion mechanism.

### G5 — Small semantic API

The MCP SHOULD expose high-level ChatGPT operations so the agent does not need to understand ChatGPT's DOM.

### G6 — UI resilience

Critical browser interactions MUST use resilient semantic/accessibility-driven strategies and SHOULD tolerate normal ChatGPT frontend changes.

## 3. Non-Goals

Version 1.0 does not include:

- OpenAI API integration as the primary transport;
- automated login or credential handling;
- CAPTCHA bypass or access-control circumvention;
- generic browser automation beyond what is required for ChatGPT;
- autonomous account/security/billing administration;
- guaranteed correctness of ChatGPT output.

## 4. Target Users

Primary users are developers running autonomous agents such as OpenClaw, OpenCode, Hermes or custom MCP-compatible runtimes who want to leverage an existing ChatGPT web session.

## 5. User Stories

- **US-001:** As an agent, discover an existing ChatGPT tab and use its authenticated session.
- **US-002:** As an agent, start or select an appropriate ChatGPT conversation.
- **US-003:** As an agent, send a message and know it was submitted.
- **US-004:** As an agent, wait until ChatGPT has finished generating.
- **US-005:** As an agent, retrieve the latest response without understanding the DOM.
- **US-006:** As an agent, provide targeted follow-ups based on the previous response.
- **US-007:** As an agent, detect browser/UI failures and recover safely.
- **US-008:** As an agent, verify the result against the original goal.

## 6. Functional Requirements

### 6.1 Browser discovery

- **FR-001:** Discover open browser tabs.
- **FR-002:** Expose tab ID, URL, title and whether it is a ChatGPT tab.
- **FR-003:** Identify ChatGPT primarily from the URL host.
- **FR-004:** Allow explicit tab selection.

### 6.2 Browser connection

Support existing Chromium through:

1. Playwright browser-extension connectivity; and
2. Chrome DevTools Protocol (CDP).

CDP SHOULD default to localhost.

### 6.3 ChatGPT state

Use an internal state model:

```text
unknown | loading | ready | generating | error | blocked | disconnected
```

State detection SHOULD combine accessibility data, DOM state, mutation observation and generation controls.

### 6.4 Sending

`chatgpt_send` MUST:

1. verify a valid selected ChatGPT tab;
2. verify the page can accept input;
3. locate the composer;
4. enter the message;
5. submit it;
6. verify submission.

It MUST avoid duplicate submission after recoverable browser failures.

### 6.5 Waiting

`chatgpt_wait` MUST wait for generation completion with a configurable timeout. It SHOULD return the completed response or a structured failure.

### 6.6 Reading

`chatgpt_read` MUST support the latest response as the default and SHOULD support conversation history for cases that genuinely need it.

### 6.7 New conversations

`chatgpt_new_conversation` SHOULD allow autonomous tasks to be isolated from unrelated user conversations.

The system MUST avoid silently injecting an autonomous task into an obviously unrelated conversation.

## 7. Agent Skill

The companion `SKILL.md` MUST teach the agent a controlled loop:

```text
goal -> status -> select/create conversation -> send -> wait -> read
        -> evaluate against success criteria
          -> complete: stop
          -> blocked: ask/stop
          -> incomplete: follow up
          -> stalled: change strategy once, then stop if still stalled
```

The skill MUST distinguish complete, incomplete, incorrect, blocked and stalled outcomes.

It MUST default to a finite iteration limit; 8 rounds is the initial default.

## 8. Completion Criteria

The agent is responsible for determining whether the user's objective is actually satisfied. A confident claim by ChatGPT is not itself proof of completion.

Where possible, completion SHOULD be checked against explicit artefacts, requirements, tests, files or other observable evidence.

## 9. Error Handling

Structured errors MUST be exposed for at least:

- `BROWSER_NOT_CONNECTED`
- `CHATGPT_TAB_NOT_FOUND`
- `INVALID_TAB`
- `PAGE_NOT_READY`
- `MESSAGE_SEND_FAILED`
- `RESPONSE_TIMEOUT`
- `GENERATION_FAILED`
- `USER_INTERVENTION_REQUIRED`
- `BROWSER_DISCONNECTED`
- `UNKNOWN_ERROR`

Errors SHOULD indicate whether retry is safe.

## 10. Security & Privacy

The MCP MUST NOT expose passwords, cookies, session tokens or browser storage to the agent.

Prompt and response content MUST NOT be logged by default. Debug logging MUST be explicit and local.

The system MUST NOT bypass CAPTCHA, authentication controls or safety mechanisms.

## 11. MCP Tool Surface

Version 1.0 SHOULD provide:

| Tool | Purpose |
|---|---|
| `chatgpt_status` | Current connection/page/conversation state |
| `chatgpt_tabs` | Discover tabs |
| `chatgpt_select_tab` | Select the active ChatGPT tab |
| `chatgpt_new_conversation` | Start an isolated conversation |
| `chatgpt_send` | Send a message |
| `chatgpt_wait` | Wait for generation completion |
| `chatgpt_read` | Read the latest response/history |

The tool surface should remain intentionally small.

## 12. Success Metric

A supported MCP-compatible agent can take a complex user-defined goal, delegate work to ChatGPT through an existing authenticated `chatgpt.com` session, conduct corrective multi-turn dialogue, and stop with a result that it has independently verified against the goal.
