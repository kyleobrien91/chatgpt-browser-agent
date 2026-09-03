# Roadmap: ChatGPT Browser Agent MCP

## Delivery strategy

Build from the outside in: first prove that an existing browser session can be safely controlled; then make the ChatGPT adapter reliable; then expose the MCP contract; finally harden autonomous multi-turn workflows.

The critical path is **connect → discover → select → send → detect generation → wait → read**. Everything else is secondary until this loop is reliable.

---

# Phase 0 — Foundation

**Goal:** Establish the project and engineering guardrails.

### Deliverables

- TypeScript / Node.js 20+ project.
- ESM / NodeNext configuration.
- Strict TypeScript settings.
- MCP SDK dependency.
- Playwright dependency.
- Configuration loader/validator.
- Structured logger with redaction.
- Repository documentation structure.
- Basic unit-test harness.

### Exit criteria

- Project installs cleanly.
- TypeScript builds successfully.
- Test runner executes in CI/local development.
- No secrets are required merely to build/test.

---

# Phase 1 — Existing Browser Connectivity

**Goal:** Attach to a browser the user already has open.

### Deliverables

- Browser session abstraction.
- CDP connector.
- Extension connector boundary.
- Tab discovery.
- Stable internal tab IDs.
- Explicit tab selection.
- Connection/disconnection detection.

### Acceptance tests

- Discover multiple browser tabs.
- Identify `chatgpt.com` tabs.
- Select a ChatGPT tab.
- Detect browser disconnect.
- Reacquire tabs after reload/reconnect.

### Exit criteria

The MCP can reliably identify and hold a reference to the intended ChatGPT `Page` without launching a second browser.

---

# Phase 2 — ChatGPT Readiness & State Detection

**Goal:** Establish reliable knowledge of the ChatGPT page state.

### Deliverables

- `ChatGPTState` model.
- Readiness detection.
- Generation-state detection.
- Blocked/error detection.
- Selector/heuristics module.
- Diagnostic status object.

### Acceptance tests

Correctly classify at minimum:

- loading page;
- ready composer;
- active generation;
- completed generation;
- browser/page error;
- user-intervention-required state.

### Exit criteria

State detection does not rely on a single DOM selector or arbitrary fixed sleep.

---

# Phase 3 — Core Conversation Operations

**Goal:** Make the basic ChatGPT conversation loop work.

### Deliverables

- semantic composer detection;
- `chatgpt_send`;
- `chatgpt_wait`;
- `chatgpt_read`;
- new-message boundary detection;
- duplicate-send protection;
- configurable timeouts.

### Acceptance test

A live test can:

1. select ChatGPT;
2. send a message;
3. observe generation;
4. wait for completion;
5. retrieve the exact latest response;
6. repeat successfully for a second turn.

### Exit criteria

The core five-step loop is reliable:

```text
select → send → wait → read → repeat
```

---

# Phase 4 — Conversation Isolation

**Goal:** Prevent autonomous work from contaminating unrelated user conversations.

### Deliverables

- `chatgpt_new_conversation`;
- conversation identity tracking;
- safe selection rules;
- navigation/reload handling;
- detection of obviously unrelated active conversations.

### Exit criteria

A workflow can start a dedicated ChatGPT conversation and maintain it across multiple turns.

---

# Phase 5 — Agent Skill

**Goal:** Turn the low-level MCP into a useful autonomous workflow.

### Deliverables

`skill/SKILL.md` covering:

- objective framing;
- success criteria;
- multi-turn iteration;
- completion evaluation;
- incomplete/incorrect responses;
- blocked state;
- stalled state;
- finite round limits;
- recovery strategy.

### Acceptance test

Given a deliberately incomplete ChatGPT response, a test agent can identify the gap and issue a targeted follow-up instead of stopping.

### Exit criteria

A supported agent can execute a real goal through several ChatGPT turns without requiring human orchestration between turns.

---

# Phase 6 — Reliability & Resilience

**Goal:** Harden against real-world browser/UI instability.

### Deliverables

- selector fallback hierarchy;
- mutation/event-based waiting;
- duplicate-submit recovery;
- browser reload recovery;
- stale-page detection;
- retry policy with safe/idempotent boundaries;
- better diagnostics;
- integration test fixtures.

### Failure scenarios to test

- ChatGPT response takes unusually long;
- page reload during generation;
- browser disconnect after send;
- transient locator failure;
- ChatGPT error response;
- generation already completed when `wait` starts;
- response contains long code blocks;
- conversation contains multiple assistant messages;
- UI controls change during a response.

### Exit criteria

The MCP fails explicitly and recoverably rather than silently returning stale or partial data.

---

# Phase 7 — Packaging & Developer Experience

**Goal:** Make the project straightforward to install and use.

### Deliverables

- npm package/application packaging;
- CLI entrypoint;
- configuration documentation;
- browser connection setup guides;
- example MCP configurations;
- troubleshooting guide;
- versioning/release process.

### Example target experience

```bash
npm install
npx chatgpt-browser-agent
```

Then configure the chosen browser connection mode in the user's MCP client.

### Exit criteria

A technically competent user can install, connect a browser and perform a first successful multi-turn task using only repository documentation.

---

# Phase 8 — Production Hardening

**Goal:** Make the project suitable for sustained personal or team use.

### Candidate work

- richer diagnostics;
- health/readiness endpoint where appropriate;
- configurable per-operation timeouts;
- concurrency/session locking;
- graceful shutdown;
- memory-leak testing;
- extended browser matrix;
- more comprehensive UI regression fixtures.

### Exit criteria

Long-running usage does not accumulate stale browser references or leave orphaned resources.

---

# Phase 9 — Advanced Capabilities

Only begin after the core conversational loop has demonstrated stable behaviour.

Potential additions:

- model selection;
- conversation selection/search;
- attachment upload;
- streaming responses;
- richer message metadata;
- multiple concurrent ChatGPT sessions;
- screenshots for diagnostics;
- generic conversational-provider abstraction.

These are explicitly lower priority than reliability of the core loop.

---

# Future Architecture — Provider Abstraction

If the project proves useful beyond ChatGPT, evolve the adapter boundary toward:

```text
ConversationProvider
├── ChatGPTProvider
├── ClaudeWebProvider
├── GeminiWebProvider
└── GenericWebChatProvider
```

with a common semantic contract such as:

```ts
interface ConversationProvider {
  status(): Promise<ProviderStatus>;
  listConversations(): Promise<Conversation[]>;
  newConversation(): Promise<Conversation>;
  send(message: string): Promise<void>;
  waitForResponse(timeoutMs?: number): Promise<Message>;
  readLatest(): Promise<Message | null>;
}
```

Do not introduce this abstraction prematurely. The ChatGPT adapter should first prove which operations are genuinely common.

---

# Milestone Definition

## MVP

Phases 0–5.

MVP means:

> A local MCP-compatible agent can attach to an existing authenticated ChatGPT browser session, send an initial task, conduct multiple follow-up exchanges, inspect responses, and stop based on explicit completion criteria.

## V1

Phases 0–7.

V1 adds recovery, polished installation, documentation and a stable developer experience.

## V2

Phases 8–9.

V2 explores production hardening and additional capabilities after the core architecture has earned the complexity.

---

# Engineering Priorities

When roadmap items compete, prioritise in this order:

1. correctness;
2. duplicate-send prevention;
3. response completion detection;
4. recovery after browser instability;
5. UI resilience;
6. security/privacy;
7. developer ergonomics;
8. advanced features.

The project should optimise for **trustworthy automation**, not maximum browser feature count.

---

# Definition of Done for the Project

The project is genuinely successful when a user can hand an MCP-compatible autonomous agent a goal such as:

> Review this implementation thoroughly, identify all significant issues, ask ChatGPT to revise its analysis where necessary, and stop only when the analysis is demonstrably complete.

and the agent can execute the workflow through the existing ChatGPT browser session without:

- manual copy/paste;
- API credentials;
- repeated human prompting;
- infinite loops;
- duplicate submissions;
- reading incomplete generations;
- or falsely declaring completion merely because ChatGPT said it was done.
