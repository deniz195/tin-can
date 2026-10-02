# tin-can: existing solutions

| | |
|---|---|
| Status | Research snapshot, 2026-10-02 |
| Method | Web research by a subagent; summarized here |

Each claim carries a tag:

- **[V]** read on an official or primary page
- **[R]** reported by a secondary, community or search-snippet source
- **[I]** inference

The research was blocked from the GitHub API, so the star counts and dates below came from fetched page text. Smithery and PulseMCP were not searched directly. Before relying on a claim, test it by hand; §6 lists the gaps.

---

## 1. Summary

1. **Nothing does exactly what tin-can does, but there are close neighbours.**
   - Four hosted or remote-MCP "mailbox / room" services overlap heavily: **AgentDM**, **Agent Room**, **Cross-Claude MCP / CrossWire** and **A2AWire agent-inbox**. There are also about 15 small local or self-hosted bridges.
   - All of them deal with "agents can't be woken" the same way. The receiving agent calls a tool that polls or blocks until a message arrives (`read_messages`, `room_listen`, `bridge_wait`, `wait_for_reply`). Some add Claude Code hooks or channels.
   - None combines all of these: a hosted relay, a pairing code that works in chat apps, a way to answer without a live session, and setup with OAuth (an industry-standard sign-in protocol) that works for ChatGPT and claude.ai.
2. **The closest competitors are Agent Room and AgentDM.**
   - **Agent Room** is the closest on pairing. It is hosted, joined with a 9-character room code, and listens with a long-poll tool. It can also wake agents through webhooks or a Claude Code Stop-hook trick. Nothing shows it working from ChatGPT or claude.ai web, and it has no authentication: anyone with the code can join.
   - **AgentDM** is a hosted grid where agents message each other by `@alias`. It also bridges to A2A and Slack. It is closed-source and in Early Access. Its own Claude-to-GPT demo uses the OpenAI Responses API rather than the ChatGPT app.
3. **Anthropic is covering the Claude Code ↔ Claude Code case itself.** Claude Code now ships cross-session messaging, agent teams and channels (a research preview). All three work only between Claude Code sessions; none reach claude.ai chat, the Office add-ins or ChatGPT. **tin-can's value is connecting different apps and vendors.**
4. **The existing protocols don't solve this.**
   - **A2A** is mature (v1.0, Linux Foundation, 150+ organizations). But it assumes every agent is an HTTP server.
   - ChatGPT and claude.ai are MCP clients. Nothing shows either one acting as an A2A client, and neither OpenAI nor Anthropic is on the Linux Foundation's list of A2A supporters.
   - Today, nothing can push a new turn into a chat client.
5. **Measured limits that shape the design:**
   - ChatGPT allows **60 s** per tool call. The limit is not configurable, and progress events don't reset it.
   - claude.ai and Claude Desktop allow **240 s**.
   - Claude Code has no practical per-call limit, but times out after 5 minutes without activity.
   - Claude Desktop reportedly stops a turn after about 20 tool calls.
6. **Claude in PowerPoint supports connectors**, including custom ones the organization has enabled. Two caveats: tools reportedly didn't load on the first turn (a May 2026 bug), and on Team and Enterprise plans only Owners can add custom connectors.
7. **Gaps tin-can could fill:**
   - Pairing that a non-technical user can complete in PowerPoint or ChatGPT: a short code plus OAuth.
   - A proper listening setup for the coding side: a channel plugin, plus fallbacks to the Monitor tool or a Stop hook, plus `claude -p` started on demand to answer.
   - A relay built around ChatGPT's 60-second cap.
   - A security model that treats the other agent's messages as untrusted and makes the person who opened a line approve whoever joins.
8. **"Consult another LLM" servers are a different category.** Examples are PAL and consult-llm. They call model APIs or start new CLI processes. They don't connect two sessions that are already running.

---

## 2. Comparison table

| # | Name | Type | Connects two running sessions across vendors? | How the receiver gets messages | Maturity | Link |
|---|---|---|---|---|---|---|
| 1 | AgentDM | Hosted remote MCP messaging grid; bridges MCP ↔ A2A | Yes, any MCP client by `@alias`. No evidence of use from the ChatGPT app; the demo uses the Responses API | Polls `read_messages`; mirrors to Slack; nothing wakes the receiver | Early Access, closed-source. Registry entry v2.0.0 (2026-05-10). Show HN | https://agentdm.ai/ |
| 2 | Agent Room | Hosted or self-hosted MCP room, joined with a 9-character code | Claude Code, Desktop, Cursor, Windsurf, Codex, Gemini and others. ChatGPT and claude.ai web not mentioned | Long-poll `room_listen`; signed (HMAC) webhook wake-up; Stop hook for Claude Code | 74 stars, MIT, 171 commits. Blog post 2026-05-16 | https://github.com/agent-room-alkl/agent-room |
| 3 | Cross-Claude MCP / CrossWire | Hosted (CrossWire) or self-hosted | Claims Claude, ChatGPT (through an OpenAPI description imported as Custom GPT Actions), Gemini, Perplexity | Polls with `wait_for_reply` (90 s, every 5 s) until a `done` signal | MIT; stars and dates unknown | https://www.shieldyourbody.com/cross-claude-mcp/ |
| 4 | A2AWire agent-inbox | Hosted remote MCP mailbox | Any client that speaks Streamable HTTP | Polls `mailbox_check`; claims a message for 300 s; acknowledges it | Listed on mcp.so 2026-09-23; free; very new | https://github.com/chatmcp/mcpso/issues/4322 |
| 5 | MCP Agent Mail | Local or HTTP FastMCP server (Git + SQLite) with inboxes and file locks | Claude Code, Codex, Gemini CLI, Cline and others working on one project | Polls the inbox, plus hooks; tracks acknowledgements | 2.2k stars, 228 forks, last commit 2026-09-29 | https://github.com/Dicklesworthstone/mcp_agent_mail |
| 6 | Claude Code cross-session messaging | Vendor feature | No, Claude Code only (local, cloud, and other machines via Remote Control) | An idle session gets a new turn; a busy session reads the message between tool calls | Shipped in v2.1.224+, on by default | https://code.claude.com/docs/en/cross-session-messaging |
| 7 | Claude Code Channels, and community bridges built on it | Vendor feature plus plugins: network-claude-peers-mcp, intermcp, osteele/agent-mail | No, the receiver must be Claude Code | A `notifications/claude/channel` message is pushed into the running session | Research preview with an allowlist; the bridges have 0–3 stars | https://code.claude.com/docs/en/channels-reference |
| 8 | Claude Code agent teams | Vendor feature | No, works only within one session | File mailboxes, delivered automatically | Experimental, off by default | https://code.claude.com/docs/en/agent-teams |
| 9 | Small local and self-hosted bridges | Claude-Bridge, agent-bridge, AgentBus, cross-agent_mcp, ccbridge, mcp-dispatch, claude-intercom, mcp-relay | Mostly Claude Code and Codex, plus Grok or Gemini via MCP. None show ChatGPT support | Blocking `*_wait` or `recv` tools; polling transcripts; file watching with `asyncRewake`; a supervisor that starts agents | 0–11 stars; hobby projects | see §3.9 |
| 10 | A2A (IBM's ACP merged into it) | Protocol | Only between A2A servers. Copilot Studio acts as a client; ChatGPT and claude.ai don't | SSE streaming; webhook push notifications; tasks that can wait for input | v1.0 (Jan 2026); 26k stars; 150+ organizations; Linux Foundation | https://github.com/a2aproject/A2A |
| 11 | MCP Tasks extension and roadmap | Protocol work | n/a | Clients poll `tasks/get` at a suggested `pollIntervalMs`; push via `subscriptions/listen` | SEP-2663 is Final; roadmap dated 2026-08-22 | https://modelcontextprotocol.io/seps/2663-tasks-extension |
| 12 | ANP and AGNTCY | Protocols and infrastructure | Planned, not yet real | ANP: messaging between decentralized identities (DIDs). AGNTCY: publish/subscribe (SLIM) | ANP proposal under review 2026-09-24; AGNTCY at the Linux Foundation with 75+ companies | https://github.com/aaif/project-proposals/issues/46 |
| 13 | "Consult another LLM" servers | PAL MCP (11.8k stars), consult-llm-mcp (138), steipete/claude-code-mcp (1.3k, archived 2026-05-15) | No. They call APIs or start new CLI processes rather than join running sessions | A synchronous call, or a background job that is polled | Popular | https://github.com/BeehiveInnovations/pal-mcp-server |
| 14 | Pieces for a headless responder | Routines `fire` API, Managed Agents events API, `claude -p`, Agent SDK | No, Anthropic only, and they start new agents | Routines: each fire starts a new cloud session. Managed Agents: POST events to an existing session | Routines are experimental | https://platform.claude.com/docs/en/api/claude-code/routines-fire |
| 15 | Copilot Studio A2A | Vendor product | Copilot Studio can call outside A2A agents | A2A `message/send` and `message:stream` | Docs updated 2026-08-26 | https://learn.microsoft.com/en-us/microsoft-copilot-studio/add-agent-agent-to-agent |

Related but not compared above:

- MCP servers that share memory or hand a conversation over, such as Muninn and conversation-handoff-mcp. They share notes, not live messages.
- Setups where agents talk through Slack.

---

## 3. Detailed notes

### 3.1 AgentDM: the closest hosted competitor

https://agentdm.ai/

- **What it is.** A hosted "grid" where agents send each other direct messages by `@alias`. [V homepage]
  - Tools: `send_message`, `read_messages`, `message_status`.
  - Setup is `npx agentdm init` or about 5 lines of JSON.
  - Limits are 1,000 agents and 10M messages a month, free during Early Access.
- **Extras.** A bridge to Slack channels (blog post 2026-04-10), filters on message content, and a server-side bridge between MCP and A2A. [V blog titles; R summaries]
- **How it compares with tin-can.**
  - It is vendor-neutral and connects sessions that already exist.
  - The homepage says ChatGPT works "through A2A bridge". But its own post "How Claude and GPT Can Send Messages" (2026-09-08) used Claude Code / Desktop and the OpenAI Responses API with a polling loop, not the ChatGPT app. [V]
  - Each agent needs its own API key, and there's no step where a human pairs two agents.
- **Waking.** Nothing wakes the receiver; both sides poll. AgentDM's own comparison with A2A admits it lacks A2A's task lifecycle and push notifications. [V]
- **Maturity.** The official registry lists `https://api.agentdm.ai/mcp/v1/grid` at v2.0.0, updated 2026-05-10. [V] Don't confuse it with github.com/quietpublish/agentdm, an unrelated local tool with the same name.
- **Worth copying:** `@alias` addressing, delivery receipts through `message_status`, and a Slack bridge so humans can watch.
- **Watch out for:** nothing wakes the receiver, so a message to an idle chat session just sits there.

### 3.2 Agent Room: the closest on pairing

https://github.com/agent-room-alkl/agent-room

- **What it is.** An MCP server, hosted at `https://www.agent-room.com/mcp` or self-hosted (Vercel + Upstash Redis).
  - Agents join a room with a 9-character code (`room_join`).
  - They post tagged messages: `[DECISION]`, `[TODO]`, `[STATUS]`, `[RESULT]`.
  - They listen with `room_listen`, a long-poll that also tracks who is present.
  - Extras: a task board where completing a task requires evidence (`room_task`), and a transcript export (`room_minutes`).
  - The author describes it as "not a router, not an orchestrator… a shared message log with presence". [V]
- **Three ways to wake an agent.** Agents loop on `room_listen`. Claude Code hooks fetch messages at turn boundaries. Signed (HMAC) webhooks POST to registered endpoints. [V]
- **The Stop-hook trick.**
  - When Claude Code tries to end its turn, the hook returns `{decision:"block", reason:"<message>"}`, which forces another turn.
  - The hook long-polls Redis for 30 s at a time.
  - It gives up after `MAX_BLOCKS_PER_CYCLE` = 60 rounds, about 30 minutes of listening.
  - It checks `stop_hook_active` to avoid an endless loop.
  - Versions exist for Cursor, Codex and Gemini CLI. [V, dev.to article]
- **Maturity and caveats.**
  - 74 stars, 23 forks, MIT licence, 171 commits.
  - Listed clients are Claude Code, Desktop, Cursor, Windsurf, Codex, Antigravity, OpenClaw and Hermes. ChatGPT and claude.ai web are not mentioned.
  - The docs say: "Anyone with the 9-character code can join". There is no authentication. [V]
- **Worth copying:** the short-code UX, structured message tags, presence, and the webhook and Stop-hook wake paths.
- **Watch out for:** an unauthenticated code serves as both identity and secret.

### 3.3 Cross-Claude MCP / CrossWire

https://www.shieldyourbody.com/cross-claude-mcp/

- **What it is.** A message server, hosted (CrossWire) or self-hosted (Node with SQLite or Postgres). [V]
  - It generates an `/openapi.json` automatically, so ChatGPT can import it as Custom GPT Actions.
  - It also claims to work over MCP with Gemini and Perplexity.
- **Waking.** `wait_for_reply` polls (90 s by default, every 5 s) until the other side sends a `done` signal. [V]
- **An honest warning from the page.** Without explicit rules, the AI agents forget to send `done`, stuff huge amounts of data into messages, or re-register partway through a conversation. [V]
- **Worth copying:** an OpenAPI / REST interface as a fallback for ChatGPT, and an explicit "I'm done" signal.

### 3.4 A2AWire agent-inbox

https://github.com/chatmcp/mcpso/issues/4322

- **What it is.** A hosted MCP mailbox at `https://a2awire.com/mcp/connectors/agent-inbox/http`, over Streamable HTTP. [V]
  - Agents register an identity and send messages to named agents.
  - `mailbox_check` reads messages page by page with a cursor.
  - `mailbox_claim` takes a message for 300 s.
  - `mailbox_ack` confirms it; repeating the call is safe.
- **Waking.** Polling only.
- **Context.** The wider A2AWire site is a marketplace for an "agent economy" that uses test USDC (a cryptocurrency). [V]
- **Worth copying:** the cursor, claim and acknowledge pattern.

### 3.5 MCP Agent Mail

https://github.com/Dicklesworthstone/mcp_agent_mail

- **What it is.** "Gmail for coding agents". [V]
  - Each agent has a persistent identity, an inbox and an outbox, and threads can be searched.
  - Agents can take advisory locks on files.
  - It runs as a FastMCP HTTP server (port 8765 by default, bearer or JWT auth). Git stores the audit trail and SQLite the index.
- **Vendors.** Claude Code, Codex, Gemini CLI, Factory Droid and Cline, all working on the same project and usually on the same machine. There's also a Rust port. [V]
- **Waking.** Agents poll `fetch_inbox`, plus hooks and acknowledgements. Nothing is pushed from the server. [V]
- **Maturity.** 2.2k stars, 228 forks, 870 commits; latest commit 2026-09-29, earliest visible 2026-06-04. [V]
- **Worth copying:** thread ids, acknowledgement tracking, and filters for unread or urgent messages to save tokens.
- **Watch out for:** its identity model assumes one trusted user and one project.

### 3.6 Claude Code cross-session messaging (vendor)

https://code.claude.com/docs/en/cross-session-messaging

- **What it is.** [V]
  - Requires v2.1.224+ on macOS, Linux and WSL2, and v2.1.234+ on native Windows.
  - Claude uses `ListAgents` and `SendMessage` to message your other sessions.
  - On the same machine, messages travel over a Unix socket per session. To other machines or the cloud, they go through Anthropic's servers via Remote Control.
  - It is on by default.
- **How messages arrive.** [V]
  - A session that is running reads the message between tool calls.
  - An idle session gets a new turn that starts with the message.
  - `notify_when_idle` sends one notice when a session next goes idle; the request expires after 12 hours.
  - A `claude -p` worker opens an inbox socket (except in `--bare` mode). It accepts messages if `crossSessionInbound` is set to `accept`.
- **Safety model.** [V]
  - Messages are plain text, up to about 1M characters.
  - Messages are labelled as coming from another session.
  - A message can't approve a permission or change settings.
  - Slash commands in a message don't run.
  - `crossSessionInbound` can be set to accept, hold or refuse.
  - The receiving session limits how often each sender can send and drops identical repeats.
  - It queues at most 50 messages, so a loop between two agents stops on its own.
  - Held messages expire after 5 minutes by default.
- **How it differs.** Claude only. A non-Claude agent could join only by speaking the undocumented socket protocol; one project, aeriondyseti/claude-message-mcp, does this (Windows only, 0 stars). [V]
- **Worth copying:** the whole design for handling inbound messages.

### 3.7 Claude Code Channels (vendor)

https://code.claude.com/docs/en/channels-reference

- **What it is.** [V]
  - A channel is a local MCP server that Claude Code starts as a subprocess and talks to over stdio.
  - The server declares `capabilities.experimental['claude/channel']` and sends `notifications/claude/channel` with `content` and `meta`.
  - Each event shows up in the session as a `<channel source=…>` tag.
  - A two-way channel also offers a normal MCP `reply` tool and can pass permission prompts through.
- **Constraints.** [V]
  - It is a research preview, and it requires signing in with a claude.ai or Console account.
  - `--channels` loads only allowlisted plugins. Anything else needs `--dangerously-load-development-channels`.
  - On Team and Enterprise plans, the organization must turn it on.
  - Events are dropped silently if the channel isn't registered, and they are never acknowledged.
  - Events that arrive while Claude is busy are queued and delivered together on the next turn.
  - On the v2 runtime, a server that negotiates MCP revision 2026-07-28 isn't registered as a channel.
- **A pairing precedent.** [V]
  - The Telegram and Discord plugins pair like this: the bot replies with a code, you approve it in the session, and your sender id goes on an allowlist.
  - The docs advise checking who sent a message, not which room it came from.
- **Community bridges built on channels.** All are tiny:
  - JasonDictos/network-claude-peers-mcp: a broker daemon that keeps messages for 7 days. 0 stars, MIT. [V]
  - Guo-astro/intermcp: 0 stars. [V]
  - osteele/agent-mail: 198 commits, 3 stars, MIT. [V]
    - It's the only one seen with delivery-receipt states: spooled, pending, held, pushed, push-unreachable, read, refused, expired.
    - Only Claude Code gets messages pushed; other agents fetch them with `check_inbox`.
- **What this means for tin-can.** A tin-can channel plugin could poll the relay over HTTPS and push questions into the running session. Today that only works within the preview's constraints. [I]

### 3.8 Claude Code agent teams

https://code.claude.com/docs/en/agent-teams

- **What it is.** [V]
  - Experimental and off by default; turned on with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.
  - Each agent gets a JSON mailbox under `~/.claude/teams/…`.
  - A session has exactly one team, so teams can't be shared across sessions.
  - Sessions run non-interactively with `-p` or through the SDK can't start teammates.
- **Relevance.** It doesn't apply to independent user sessions. But its combination of mailbox, idle notification and shared task list is a good model.

### 3.9 Small local and self-hosted bridges

All of these are small projects.

- **Claude-Bridge** (constripacity): https://github.com/constripacity/claude-bridge [V]
  - A self-hosted relay on SQLite, served over Streamable HTTP at `/mcp`.
  - Named channels; long-polls with `bridge_wait`; claims messages for a time with `bridge_claim`.
  - Documented for Claude Code and Codex. 11 stars, MIT, 58 commits.
- **agent-bridge** (Prakshal-Jain): https://github.com/Prakshal-Jain/agent-bridge [V]
  - A local relay over stdio.
  - Its README says: "an MCP server is request/response: it can't push a new turn into an idle CLI session".
  - Its answer: clients sit in `bridge_wait`, which returns when a message arrives, or a daemon (`watch.js`) starts an agent to answer each new message. 0 stars.
- **AgentBus** (oznotes): https://github.com/oznotes/AgentCommBus [V]
  - A daemon with a blocking `recv` tool and a live dashboard.
  - Lists OpenCode, Kimi, Claude Code, Gemini CLI, Cursor and Codex.
- **cross-agent_mcp** (whooperlove): https://github.com/whooperlove/cross-agent_mcp [V]
  - Connects Claude Code, Codex and Grok on one machine. It finds sessions by scanning their transcript files.
  - Polls the transcripts every 15 s for up to 15 minutes, with guards against loops. 2 stars, MIT.
- **ccbridge** (sourabhnirvani): https://github.com/sourabhnirvani/ccbridge [V]
  - Connects two people's Claude Code agents through a Cloudflare tunnel.
  - You join with an invite line that carries a room token. Each session has its own identity key, so nobody can pose as someone else.
  - Messages are text only and are cleaned both at the relay and on delivery. Random tags stop forged prompts.
  - Limits: 8 KB per message, 60 messages per minute, 7-day expiry. Receivers poll with `get_messages`. 0 stars.
- **mcp-dispatch** (justinstimatze): https://github.com/justinstimatze/mcp-dispatch [V]
  - A relay that uses the file system. Pending messages ride along on the responses to other calls; agents can also wait with `dispatch-wait`.
  - Desktop notifications.
  - The `dispatch-supervise` daemon starts an agent when mail arrives for one that's offline.
  - Can optionally sync between machines over git. 1 star.
- **claude-intercom** (sanztheo): https://github.com/sanztheo/claude-intercom [V]
  - A file watcher exits with code 2, which makes Claude Code's `asyncRewake` start a new turn.
  - Requires Bun and Claude Code 2.1+. 3 stars.
- **mcp-relay** (mhcoen): https://github.com/mhcoen/mcp-relay [V]
  - Claude Desktop ↔ Claude Code only. Messages are buffered in SQLite and fetched with an explicit `/get`.
  - Holds up to 20 messages of 8 KB each. 4 stars.
- **claude-message-mcp:** a `wait_for_messages` long-poll and an `--on-message` shell hook. The hook waits for messages to settle before firing and by default wakes the agent at most 30 times an hour. [V]

None of these show any sign of working from ChatGPT or claude.ai web. They are local, use stdio, or only work with Claude Code.

### 3.10 A2A, with ACP merged into it

https://github.com/a2aproject/A2A

- **Status.**
  - Google announced it in April 2025 and gave it to the Linux Foundation in June 2025. [V Wikipedia]
  - The Linux Foundation's April 2026 press release reports 150+ organizations, more than 22k stars, SDKs in 5 languages, and v1.0 with signed Agent Cards. The repo now shows 26k stars, Apache-2.0 licence. [V]
  - Listed supporters: AWS, Cisco, Google, IBM, Microsoft, Salesforce, SAP, ServiceNow. **OpenAI and Anthropic are not listed.** [V]
  - IBM/BeeAI's ACP merged into A2A in August 2025. [V LF AI & Data blog title]
  - Releases: v1.0.0 in January 2026 and v1.0.1 in May 2026. [R AIwire]
- **Finding agents.** Each agent publishes an Agent Card at `https://{domain}/.well-known/agent-card.json`. Agents are found through curated registries or direct configuration; the spec doesn't define a registry API. [V]
- **Async work.** Responses can stream over SSE (`SendStreamingMessage`), and webhook notifications can be set up per task (`TaskPushNotificationConfig`). A task can pause in `input-required` or `auth-required` states. [V]
- **Why it doesn't solve tin-can's problem.**
  - It assumes every agent is an HTTP service with its own address.
  - Pairing isn't part of the protocol. One academic survey says MCP, A2A and ACP can't express delegation, consent, human approval or pairing between agents. [R, arXiv 2606.31498, as summarized by the fetcher]
  - A2A is mostly used between enterprise platforms: SAP, Salesforce, Microsoft Agent 365 and Google Agentspace. [R]
- **A possible angle.** AgentDM already bridges MCP and A2A on the server. tin-can could present a line as an A2A agent with its own Agent Card, so Copilot Studio and similar products could reach it. [I]

### 3.11 MCP spec work

https://blog.modelcontextprotocol.io/posts/mcp-roadmap/

- **The 2026-07-28 revision makes the protocol core stateless.** [R: WorkOS and other blogs; the Claude Code docs confirm the revision exists]
  - Protocol sessions, the `initialize` handshake and stream resumption are removed. Each request carries its version and capabilities in `_meta`.
  - Server push moves to `subscriptions/listen`.
  - Tasks become an extension.
- **SEP-2663, the Tasks extension (Final).** [V]
  - A server can answer `tools/call` with a task handle (`resultType:"task"`) instead of a result.
  - The client then polls `tasks/get`, following `pollIntervalMs` and `ttlMs`.
  - Task states: working, input_required, completed, cancelled, failed.
  - Servers can optionally push `notifications/tasks` over `subscriptions/listen`.
- **Roadmap (2026-08-22).** [V] It lists more mature Tasks, "server-initiated events (webhooks and channels, so clients aren't left polling for results)", and identity for agents that act for users who aren't present. These are plans; no client does this yet.
- **Client support.**
  - claude.ai says it doesn't support resource subscriptions, sampling, or "advanced or draft capabilities". [V]
  - It documents OAuth specs only up to 2025-11-25. [V]
  - Nothing found shows ChatGPT or claude.ai using the 2026-07-28 revision. [I]

### 3.12 ANP and AGNTCY

- **ANP.** A stack for identity, messaging and discovery built on decentralized identifiers (`did:wba`).
  - A proposal for it to join the Agentic AI Foundation (AAIF) was filed 2026-09-24 and is "Under Review" for Sandbox status.
  - 50+ contributors. It has also been pitched to 3GPP for 6G. [V]
- **AGNTCY.** Started by Cisco and given to the Linux Foundation in July 2025, with 75+ companies backing it. [R]
  - Its parts include an Agent Directory for federated discovery and SLIM, a fast, encrypted (MLS) publish/subscribe system.
  - Repos were updated 2026-10-02.
- **Relevance.** Neither appears in ChatGPT or claude.ai. Both are long-term options for identity and discovery.

### 3.13 "Consult another LLM" servers: a different thing

- **PAL MCP** (BeehiveInnovations, formerly zen): 11.8k stars, Apache-2.0. [V]
  - Mostly calls provider APIs directly and keeps track of conversation threads.
  - Its `clink` tool starts Gemini or Codex CLIs as separate processes.
- **consult-llm-mcp** (raine): 138 stars, MIT. Calls APIs (OpenAI, Gemini, DeepSeek, Anthropic and others) or runs CLIs. [V]
- **steipete/claude-code-mcp:** runs Claude Code as an MCP server for one-shot tasks. 1.3k stars, archived 2026-05-15. xihuai18/claude-code-mcp and mkXultra/ai-cli-mcp are similar. [V]
- **mcp-chatgpt-responses** (billster45): lets Claude and ChatGPT talk through the Responses API. [R]

These start a model or call one. They can't see your ChatGPT conversation or the Claude Code session you have open. They do resemble part of what tin-can's headless-responder mode would be.

### 3.14 Pieces for a headless responder (Anthropic)

- **Routines `fire` API.** [V]
  - `POST https://api.anthropic.com/v1/claude_code/routines/{id}/fire`, with a body of `{"text": …}` up to 65,536 characters.
  - Each call starts a new cloud Claude Code session. There's no idempotency key, so a retried call starts another session.
  - It returns the session id and URL straight away. It doesn't wait for or stream the output.
  - Limits: 30 fires an hour per routine, 100 an hour per account.
  - Requires a claude.ai Pro, Max, Team or Enterprise plan with Claude Code on the web. Experimental; each routine has its own token.
  - It runs in a cloud sandbox cloned from GitHub. So it can't see a wiki repo that exists only locally. [I]
- **Managed Agents.** A hosted REST API (April 2026). `POST /v1/sessions/{id}/events` sends user messages into a session. [R, via search results; not opened]
- **`claude -p --resume`.** The approach used by claude-intercom-worker and `--on-message` hooks. [R]

### 3.15 Copilot Studio and Microsoft

- **Copilot Studio.** [V; docs updated 2026-08-26]
  - You can add an outside "A2A agent" by its endpoint URL. Copilot Studio reads the Agent Card automatically.
  - Auth can be none, an API key or OAuth2. Conversations can be multi-turn, and the full chat history goes along in the message metadata.
  - It can reach agents on-premises or in a private network (VNet) through custom connectors.
- **Microsoft Agent Framework 1.0.** Generally available since April 2026, replacing AutoGen. Its A2A support is a beta adapter. [R]
- **What this means.** Copilot Studio is an A2A client, not somewhere a user pairs a running session. Giving tin-can an A2A interface would be the way to integrate. [I]

### 3.16 Agents talking through Slack

- Claude Code's Slack channel bridge can put several bots in one channel (`ChannelPolicy.allowBotIds`). Bots are addressed by @mention and ignore their own messages. [R, search snippet only]
- Claude Tag is Anthropic's shared Slack agent, launched 2026-06-23 for Team and Enterprise. [R]
- AgentDM can mirror its channels to Slack. [V]

Slack works as a transport and as a log humans can read. But both agents then have to be Slack bots, rather than the chat sessions people already have open.

### 3.17 Multi-agent frameworks, briefly

- AutoGen (now in maintenance mode, replaced by Microsoft Agent Framework), CrewAI (supports MCP and A2A natively) and LangGraph coordinate agents inside one program or runtime. [R]
- OpenAI Agents SDK handoffs run inside a single run and process, and the agent handed to takes over the conversation. [V]
- None of these connect user sessions that run independently, so none are competitors.

---

## 4. Facts that bear on the PRD's assumptions

| Fact | Source | Confidence |
|---|---|---|
| **ChatGPT**: "the hard limit on any tool call is 1 minute" (OpenAI staff, 2026-04-27). Not configurable | https://community.openai.com/t/how-to-configure-long-mcp-tool-call-times-for-chatgpt-app/1379834 | High: forum post by OpenAI staff |
| ChatGPT / OpenAI remote MCP: support said the "~60s–1–2 minute window remains enforced" (Dec 2025), and "Progress events from your MCP do not currently reset that timeout" | https://community.openai.com/t/call-remote-mcp-server-tool-timed-out-resulting-in-error-424/1364167 | Medium-high: the thread is about the Responses API and 424 errors, but agrees with the ChatGPT statement |
| A tool call of about 60–70 s in ChatGPT was then called again, so `ask` and `send` must handle repeats safely | Same OpenAI forum thread (1379834) | Medium |
| **claude.ai / Desktop**: 240 s per tool call; tool results up to about 150,000 characters. Claude Code: set with `MCP_TOOL_TIMEOUT` | https://claude.com/docs/connectors/building | High (official). A third-party blog says 300 s; trust the official 240 s |
| **Claude Code**: `MCP_TOOL_TIMEOUT` defaults to about 28 hours and can be set per server with `"timeout"` in `.mcp.json`. HTTP, SSE and claude.ai connectors time out after 5 minutes with no response or progress notification (30 minutes for stdio). Each request must get its first byte within at least 60 s | https://code.claude.com/docs/en/mcp | High |
| **claude.ai connectors** run from Anthropic's cloud, even on Desktop and mobile. Requirements: the server is on the public internet; its hostname resolves to a public IPv4 address (an IPv6-only host fails); it doesn't redirect to another host (that drops the `Authorization` header); the token endpoint answers within 10 s; OAuth uses PKCE with S256; the client is registered through DCR, CIMD or ahead of time | https://claude.com/docs/connectors/building/troubleshooting and https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp | High |
| claude.ai does **not** support resource subscriptions, sampling or draft capabilities, so a server can't push into a chat. Streamable HTTP is recommended; the older SSE transport is being phased out | https://claude.com/docs/connectors/building | High |
| **ChatGPT developer mode**: on the web for Pro, Plus, Business, Enterprise and Education. It is a full MCP client, so tools can read and write, over SSE or streaming HTTP. Write actions need the user's confirmation by default; the confirmation can be remembered for the conversation, but a refreshed conversation asks again | https://developers.openai.com/api/docs/guides/developer-mode | High for the doc. One third-party guide says custom connectors are read-only on Plus and Pro. That contradicts the doc and is unresolved, so test it |
| ChatGPT tool annotations (`readOnlyHint`, `destructiveHint`, `openWorldHint`) change when ChatGPT asks for confirmation. They don't replace authorization checks on the server | https://developers.openai.com/apps-sdk/build/mcp-server | High |
| **Sessions**: the 2026-07-28 MCP revision removes protocol sessions; every request describes itself. Claude's docs list OAuth specs only up to 2025-11-25, and Claude Code supports 2026-07-28 only as an opt-in on its "v2 runtime" | https://workos.com/blog/mcp-stateless-spec-2026-07-28 and https://code.claude.com/docs/en/mcp | High for the spec. **Unknown** which revision claude.ai and ChatGPT use. Don't rely on `Mcp-Session-Id` |
| ChatGPT reconnecting after about 2 hours idle: instead of refreshing its tokens it tries an SSE `GET`, and gets stuck in a loop of failures. In September 2026 OpenAI support "can't confirm this reconnect issue is resolved" | https://community.openai.com/t/stateless-streamable-http-mcp-server-token-refresh-works-after-short-idle-but-reconnect-loop-after-long-idle/1367543 | Medium: one user report plus a support reply |
| **No authoritative source says** whether ChatGPT or claude.ai keep an MCP session across tool calls or conversations. A search snippet claimed ChatGPT "does not maintain open sessions or continue polling", but gave no primary source | n/a | Unverified. Assume no session is kept and design for that |
| Tool calls per turn: Claude Desktop reportedly stops after about 20 tool calls with "tool-use limit for this turn" and a Continue button (default 10 sampling iterations) | https://github.com/anthropics/claude-code/issues/33969 | Low-medium: a user report with no reply from Anthropic; the issue was marked invalid |
| **Claude in PowerPoint** supports connectors. The docs say it "supports connectors for pulling external context", "plus any custom connectors your organization has enabled", turned on "in your Claude settings". Connectors may be missing when Claude is reached through Bedrock, Vertex, Foundry or an LLM gateway | https://claude.com/docs/office-agents/powerpoint and https://claude.com/docs/office-agents/connectors-and-skills | High that connectors work there. **Medium** that a custom connector you add yourself shows up in the add-in: the custom-connector help page doesn't mention Office add-ins |
| Claude for PowerPoint is generally available on Pro, Max, Team and Enterprise. The add-in keeps chat history locally in the browser | Same PowerPoint doc | High |
| PowerPoint bug report (2026-05-13): custom MCP tools weren't there on the first turn of a new session. They appeared only after the user typed "refresh <name> MCP tools" (`refresh_mcp_connectors`). Closed as "invalid", so it's unclear whether it was fixed | https://github.com/anthropics/claude-code/issues/58727 | Medium: one Enterprise report; current status unknown |
| On Team plans only Owners and Primary Owners can add custom connectors. The Free plan allows one custom connector | https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp | High |
| **Ways to wake Claude Code**: (a) Channels (§3.7); (b) cross-session `SendMessage` into an idle session or a `-p` session (§3.6); (c) a hook with `asyncRewake:true` wakes Claude when it exits with code 2; (d) a Stop hook that returns `decision:"block"`; (e) the **Monitor tool**, which can run a background script or open a `ws://` / `wss://` connection and hand each line it receives to Claude. Monitor stops after 5 minutes by default, 30 at most | https://code.claude.com/docs/en/hooks and https://code.claude.com/docs/en/tools-reference | High that these exist. **Not checked** whether Monitor starts a new turn in a fully idle session; test it |

---

## 5. Ideas to borrow and mistakes to avoid

**Waiting and waking**

1. **Don't make `ask` block.**
   - It should return a message id right away; the agent then calls `await(id, wait_s)` to wait for the answer.
   - Cap each wait at about 45–50 s for ChatGPT (1-minute limit) and about 200 s for claude.ai (240 s limit).
   - This matches MCP Tasks (`tasks/get` with `pollIntervalMs` and `ttlMs`) and the cursor, claim and acknowledge model used by AgentDM and agent-inbox.
   - Make `ask` safe to repeat by using a message id supplied by the client. ChatGPT repeated a tool call that ran past about 60 s.
2. **Don't rely on sessions.** Pass `line_id` and a token as tool arguments, using handles the server creates. That fits the stateless 2026-07-28 spec and copes with ChatGPT's unreliable reconnects.
3. **Show presence and delivery receipts, so the asker knows what will happen.**
   - Report whether the other agent is listening, when it was last seen, whether a push can reach it, or whether the message is just queued. Precedents: osteele/agent-mail's receipt states and AgentDM's `message_status`.
   - If the other side isn't listening, don't make the asker wait. Fail fast, and suggest asking the user to nudge it or starting the headless responder.
4. **Offer three ways to listen, all behind one relay:**
   - A long-poll tool for chat agents.
   - A Claude Code channel plugin, which gets messages pushed in. Because channels are a preview with an allowlist, fall back to Monitor, a Stop hook or `asyncRewake`.
   - A headless responder started on demand (`claude -p --resume`, or Routines `fire` in the cloud), with limits on rate and on how many run at once. Good precedents: mcp-dispatch's supervisor, and claude-message-mcp's settle delay and 30-per-hour cap.
5. **Plan for the cost in tokens and turns.**
   - Polling uses up turns; this is why Agent Room built the Stop hook.
   - Chat clients limit tool calls per turn, so an agent that keeps calling `await` can run out.
   - Keep listening loops short, and make them easy to resume.

**Pairing and security**

6. **Pairing.**
   - Use a short code like Agent Room's 9 characters, but make it single-use with an expiry.
   - The person who opened the line should approve whoever joins. Claude Code's channel pairing flow is the precedent.
   - Tie each participant to its own identity key, as ccbridge does, so joining a line doesn't let you pose as someone else.
   - Avoid Agent Room's approach, where the code is both the secret and the identity.
7. **Treat peer messages as untrusted.**
   - Claude Code's approach: text only, labelled as coming from another session. A message can't approve permissions or change settings, and commands in it don't run.
   - ccbridge adds more:
     - It cleans messages at the relay and again on delivery.
     - It wraps them in random delimiters, so a message can't fake a system banner.
     - Limits: 8 KB per message, 60 messages a minute, 7-day expiry.
   - Check who sent a message, not just which room it's in.
   - The Anthropic and ccbridge approach is the right starting point for an agent that reads a wiki and might answer from sensitive files.
8. **Stop endless loops.**
   - Both agents might keep replying to each other forever. Copy Claude Code's limits on incoming messages: a rate limit per sender, dropping identical repeats, and a queue capped at 50.
   - Add an explicit `done` or end-of-turn signal (as Cross-Claude does), and something like Agent Room's turn-taking modes.
9. **Privacy.** AgentDM claims AES-256 encryption and deletion once a message is delivered. Decide how long tin-can keeps messages and whether it encrypts them, and say so publicly. This matters especially for wiki content.

**Working with each client**

10. **Put the protocol rules in the tool descriptions and the server's `instructions`.** Cross-Claude's authors found that agents forget the rules if they aren't spelled out. Use structured message tags like Agent Room's.
11. **ChatGPT friction.**
    - `send` is a write tool, so ChatGPT will ask the user to confirm it, with an option to remember the choice for that conversation.
    - Mark the tools that only read with `readOnlyHint`.
    - Keep the number of tools small; reports say ChatGPT gets worse as tools are added. [R]
12. **Hosting for claude.ai and PowerPoint.**
    - Use a hostname with a public IPv4 address, and don't redirect to another host.
    - Support OAuth with DCR or CIMD and PKCE (S256), and make the token endpoint answer within 10 s.
    - Allow Anthropic's IP range through any firewall.
    - The PowerPoint add-in may not load connector tools on the first turn. So send `tools/list_changed`, and document that users can say "refresh tin-can tools".
    - On Team and Enterprise plans, an Owner will have to add the connector.
13. **Optionally, support A2A and MCP Tasks.**
    - An A2A Agent Card, as AgentDM has, would let Copilot Studio and similar clients connect.
    - Shaping `ask` and `await` like MCP Tasks keeps tin-can compatible once clients support Tasks.

**Strategic risks**

14. **Anthropic may cover more of this.** It already ships messaging between Claude Code sessions, channels and Remote Control. Position tin-can on what the vendors are unlikely to build: connecting different vendors, chat apps and Office add-ins.
15. **Channels is a research preview with an allowlist.** Have a fallback so the Claude Code side still works without it.
16. **Check the name.** "AgentDM" and "Agent Mail" each refer to several unrelated projects. Search before launching under a name.

---

## 6. Still to verify

- Whether AgentDM, Agent Room or CrossWire actually work from the ChatGPT app or claude.ai web. Neither Agent Room's pages nor AgentDM's blog show it. Test by hand.
- Which MCP protocol revision claude.ai and ChatGPT use today, and whether either keeps a session across tool calls.
- Whether Monitor can wake a Claude Code session that is fully idle.
- Whether the PowerPoint first-turn tool-loading bug is fixed, and whether a custom connector you add yourself (not one the organization enabled) appears in the add-in.
- Last-commit dates for most of the small repos; only MCP Agent Mail's (2026-09-29) is confirmed.
- Smithery and PulseMCP weren't searched directly.
- Some A2A adoption and version details came from press and blog summaries rather than the spec itself: the v1.0.1 date and the Agentic AI Foundation.
