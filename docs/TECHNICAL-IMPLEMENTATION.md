# Technical Implementation: ChatGPT Browser Agent MCP

**Status:** Proposed  
**Target:** Node.js 20+ / TypeScript / ESM  
**Browser automation:** Playwright  
**Protocol:** MCP

## 1. Architecture

The implementation separates three concerns:

```text
Agent Runtime
  |
  | Agent Skill + MCP
  v
ChatGPT MCP Server
  |
  +-- Tool layer
  +-- ChatGPT adapter
  +-- Browser/session manager
  +-- State detector
  +-- Conversation reader
  |
  v
Playwright
  |
  v
Existing Chromium browser
  |
  v
chatgpt.com
```

The agent owns reasoning, goal evaluation and turn selection. The MCP owns browser mechanics and exposes deterministic semantic operations. ChatGPT owns conversation context.

## 2. Technology Choices

### Runtime

- Node.js 20+.
- TypeScript with strict mode.
- ESM / NodeNext module resolution.
- `tsx` or equivalent for development execution.

### Browser

Playwright is the browser automation layer.

The implementation should support two connection modes:

```text
extension
cdp
```

Extension mode is preferred for a normal end-user workflow because it can attach to an existing browser. CDP mode is useful for controlled local automation environments.

### MCP

Use the official MCP TypeScript SDK and stdio transport for the initial implementation. HTTP transport can be added later if deployment requires it.

## 3. Repository Structure

```text
chatgpt-browser-agent/
├── README.md
├── package.json
├── tsconfig.json
├── LICENSE
├── docs/
│   ├── PRD.md
│   ├── TECHNICAL-IMPLEMENTATION.md
│   └── ROADMAP.md
├── skill/
│   └── SKILL.md
├── src/
│   ├── index.ts
│   ├── server.ts
│   ├── config.ts
│   ├── types.ts
│   ├── browser/
│   │   ├── browser.ts
│   │   ├── cdp.ts
│   │   ├── extension.ts
│   │   └── tabs.ts
│   ├── chatgpt/
│   │   ├── adapter.ts
│   │   ├── composer.ts
│   │   ├── conversation.ts
│   │   ├── response.ts
│   │   ├── selectors.ts
│   │   └── state.ts
│   └── tools/
│       ├── status.ts
│       ├── tabs.ts
│       ├── select-tab.ts
│       ├── new-conversation.ts
│       ├── send.ts
│       ├── wait.ts
│       └── read.ts
└── tests/
    ├── unit/
    ├── integration/
    └── fixtures/
```

## 4. Core Interfaces

### Browser session

```ts
interface BrowserTab {
  id: string;
  page: Page;
  url: string;
  title: string;
  isChatGPT: boolean;
}

interface BrowserSession {
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  listTabs(): Promise<BrowserTab[]>;
  selectTab(id: string): Promise<void>;
  getSelectedTab(): BrowserTab | undefined;
}
```

### ChatGPT adapter

```ts
interface ChatGPTAdapter {
  isChatGPT(page: Page): Promise<boolean>;
  getState(page: Page): Promise<ChatGPTState>;
  send(page: Page, message: string): Promise<void>;
  waitForResponse(page: Page, timeoutMs: number): Promise<ChatGPTMessage>;
  readLatest(page: Page): Promise<ChatGPTMessage | null>;
  readConversation(page: Page): Promise<ChatGPTMessage[]>;
  newConversation(page: Page): Promise<void>;
}
```

### State

```ts
type ChatGPTState =
  | 'unknown'
  | 'loading'
  | 'ready'
  | 'generating'
  | 'error'
  | 'blocked'
  | 'disconnected';
```

## 5. MCP Tools

### `chatgpt_status`

Returns selected-tab and ChatGPT state information.

```ts
{
  connected: boolean;
  tab?: {
    id: string;
    url: string;
    title: string;
  };
  state: ChatGPTState;
  conversation?: {
    id?: string;
    ready: boolean;
    generating: boolean;
  };
}
```

### `chatgpt_tabs`

Lists browser tabs and marks ChatGPT candidates.

### `chatgpt_select_tab`

Input:

```ts
{ tab_id: string }
```

The server should reject non-ChatGPT tabs unless a future generic-provider mode explicitly permits them.

### `chatgpt_new_conversation`

Starts a fresh conversation in the selected ChatGPT tab and returns the resulting conversation identity where discoverable.

### `chatgpt_send`

Input:

```ts
{ message: string }
```

The implementation MUST verify the composer is available and the send operation changed the conversation state before returning success.

### `chatgpt_wait`

Input:

```ts
{ timeout_ms?: number }
```

The implementation should subscribe to observable page state where feasible, rather than polling with long fixed sleeps.

### `chatgpt_read`

Input:

```ts
{
  latest?: boolean;
  limit?: number;
}
```

Default to the latest assistant message. Full-history retrieval is opt-in.

## 6. Browser Connection Layer

### CDP

The CDP connector should accept a local endpoint such as:

```text
http://127.0.0.1:9222
```

Use Playwright's Chromium connection support. Do not expose the CDP endpoint to the downstream agent as a tool argument.

### Extension

The extension adapter should use Playwright's supported existing-browser/extension connection mechanism. Its job is to discover tabs and expose the selected `Page` to the same browser-session interface used by CDP.

The rest of the application MUST NOT care which connection mode is active.

## 7. ChatGPT UI Adapter

The adapter is the anti-corruption layer between the volatile ChatGPT frontend and the stable MCP interface.

### Selector strategy

Use this order:

1. accessibility role/name;
2. stable semantic attributes;
3. stable application-specific attributes discovered during testing;
4. structural DOM heuristics;
5. last-resort coordinate/visual techniques.

Do not make one CSS selector the sole critical dependency.

The selectors module should centralise UI-specific heuristics so frontend changes remain isolated.

## 8. Response Lifecycle

The response lifecycle should be modelled explicitly:

```text
READY
  |
  | submit
  v
GENERATING
  |
  | generation completes
  v
READY
```

Error branches:

```text
GENERATING -> ERROR
READY      -> BLOCKED
ANY        -> DISCONNECTED
```

`waitForResponse()` should establish the expected new assistant-message boundary before or immediately after submission to avoid accidentally returning an older response.

## 9. Completion Detection

Completion detection is the most important reliability mechanism in the project.

The implementation should use multiple signals rather than a single DOM element:

- appearance of a new assistant message;
- generation controls changing state;
- composer becoming available again;
- stop-generation control disappearing;
- observable DOM mutations slowing/ending;
- page navigation or error indicators.

A response should be considered complete only when the expected new assistant message exists and the page has returned to a stable, input-capable state, subject to ChatGPT UI behaviour discovered by tests.

## 10. Duplicate Submission Protection

The send operation must be idempotency-aware.

Before submitting:

1. capture a conversation fingerprint/message count if available;
2. submit;
3. verify that the user message appeared;
4. if an exception occurs after submission, inspect the conversation before retrying.

Never blindly retry a send operation whose outcome is ambiguous.

## 11. Conversation Identity

Where the UI exposes a stable conversation identifier through the URL, navigation state or page metadata, the adapter should capture it.

Where no stable ID is available, the MCP can use a session-scoped synthetic identity and retain the selected page as the authoritative conversation context.

## 12. Agent Skill Design

`skill/SKILL.md` should be deliberately independent of DOM details.

It should teach:

- initial goal framing;
- success-criteria extraction;
- turn counting;
- response evaluation;
- follow-up generation;
- blocked-state handling;
- stalled-conversation handling;
- maximum-round termination;
- completion verification.

The skill must not tell the agent to click selectors or emulate keyboard events directly. Those responsibilities belong to MCP tools.

## 13. Configuration

Example:

```text
BROWSER_MODE=cdp
CDP_ENDPOINT=http://127.0.0.1:9222
CHATGPT_HOST=chatgpt.com
DEFAULT_TIMEOUT_MS=120000
MAX_ROUNDS=8
LOG_LEVEL=info
```

Configuration should be validated at startup and surfaced as actionable errors.

## 14. Logging

Use structured logging. Default level: `info`.

Example:

```text
INFO browser.connected
INFO tabs.discovered count=4
INFO tab.selected id=tab-1
INFO chatgpt.state state=ready
INFO chatgpt.message.sent
INFO chatgpt.generation.started
INFO chatgpt.generation.completed elapsed_ms=38217
```

Never log message content, cookies, credentials or browser storage by default.

## 15. Testing Strategy

### Unit

Mock browser pages and verify:

- state transitions;
- selector fallback ordering;
- timeout behaviour;
- duplicate-send handling;
- error classification;
- configuration validation.

### Integration

Run against a real Chromium session and a real ChatGPT account in a controlled environment. Tests should cover:

1. browser connection;
2. tab discovery;
3. ChatGPT selection;
4. new conversation;
5. single-turn send/read;
6. multi-turn interaction;
7. long responses;
8. reload/reconnect;
9. generation timeout;
10. page error/block states.

Tests that require a live account should be opt-in and never run in ordinary CI.

## 16. Reliability Targets

Initial targets for MVP:

- message submission success: >= 99% in supported UI states;
- response completion detection: >= 99% in supported UI states;
- duplicate submission rate: 0 in tested failure scenarios;
- no credential/session-token leakage in logs;
- deterministic timeout/error classification.

These should be measured rather than treated as guarantees.

## 17. Security Model

The browser session is privileged because it contains the user's authenticated ChatGPT account.

Therefore:

- only explicitly selected tabs are controllable;
- credentials and session cookies remain inside the browser;
- CDP should bind to loopback unless the operator explicitly opts into another topology;
- tool inputs should be treated as untrusted agent-generated text;
- logs must redact sensitive data;
- no authentication bypass or CAPTCHA automation is permitted.

## 18. Implementation Sequence

The recommended engineering order is:

1. configuration and project skeleton;
2. browser connection abstraction;
3. tab discovery/selection;
4. ChatGPT readiness state detection;
5. semantic composer detection;
6. send implementation;
7. response completion detection;
8. response reading;
9. new-conversation support;
10. MCP registration;
11. Agent Skill;
12. integration tests;
13. resilience/reconnect hardening;
14. documentation and packaging.

Do not build advanced features before the core send/wait/read loop is reliable.
