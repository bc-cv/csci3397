# Build-Your-Own-OpenClaw — Tutorial Summary

Source: `others/repo/build-your-own-openclaw/` (reference impl: [pickle-bot](https://github.com/czl9707/pickle-bot)).

## Overview

Tutorial walks reader through building a production-grade AI agent system from scratch across 18 incremental steps. Starts with a simple LLM chat loop and progressively layers in tools, skills, persistence, compaction, event-driven architecture, multi-channel delivery, multi-agent routing, cron scheduling, agent dispatch, concurrency control, and long-term memory. Each step keeps the code runnable and calls out a single architectural shift, so the reader sees exactly which limitation motivated the next abstraction. End state is a scalable, multi-agent, multi-channel system mirroring the OpenClaw reference implementation.

## Tutorial Phases

- **Phase 1 — Capable Single Agent**: Steps 00, 01, 02, 03, 04, 05, 06
- **Phase 2 — Event-Driven Architecture**: Steps 07, 08, 09, 10
- **Phase 3 — Autonomous & Multi-Agent**: Steps 11, 12, 13, 14, 15
- **Phase 4 — Production & Scale**: Steps 16, 17

---

## Step 00: Just a Chat Loop
**Goal**: Build foundational chat loop that takes user input and returns LLM responses.

**Key concepts**:
- `ChatLoop`: user-input REPL with rich console output
- `AgentSession`: manages conversation state, passes full history to LLM each turn
- `LLMProvider`: thin wrapper over litellm `acompletion`
- Full message history sent every call (no truncation yet)

**Key files**: `src/mybot/cli/chat.py`, `src/mybot/core/agent.py`, `src/mybot/provider/llm/base.py`

**Why it matters**: Baseline. Every later abstraction extends this loop. Without it, nothing else works.

---

## Step 01: Tools
**Goal**: Give agent ability to take real actions (Read, Write, Bash).

**Key concepts**:
- `BaseTool` ABC with `execute` + `get_tool_schema` returning OpenAI-style function-calling schema
- Two LLM stop reasons: `end_turn` vs `tool_use` — chat loop runs until `end_turn`
- Tool-calling loop: LLM → tool calls → append results → LLM → repeat
- Simple tools (Read/Write/Bash) compose into surprisingly powerful behavior

**Key files**: `src/mybot/tools/base.py`, updated `src/mybot/core/agent.py`

**Why it matters**: Prior step was chat only — agent could not touch filesystem or environment. This step unlocks action.

---

## Step 02: Skills
**Goal**: Lazy-load capabilities at runtime via `SKILL.md` files.

**Key concepts**:
- `SkillDef`: id, name, description, content
- `SKILL.md` format: YAML frontmatter metadata + markdown body
- `skill` tool dynamically lists available skills in its schema; loads full content on demand
- Saves context window by not injecting all skills upfront
- Alternative: OpenClaw uses system-prompt injection + Read tool instead of a skill tool

**Key files**: `src/mybot/tools/skill_tool.py`, workspace skills directory

**Why it matters**: Tools are baked in at startup; skills let users extend the agent without code changes, and keep system prompt small.

---

## Step 03: Persistence
**Goal**: Save and restore conversation history across sessions.

**Key concepts**:
- JSONL-based storage: `.history/index.jsonl` for session metadata, `.history/sessions/{id}.jsonl` for messages
- `HistoryStore` with `create_session`, `save_message`, `get_messages`
- Each run starts a new session; messages persisted append-only
- File-based (no DB dependency) keeps it hackable

**Key files**: `src/mybot/core/history.py` (new)

**Why it matters**: Prior steps lose memory on exit. Persistence is prerequisite for resumable sessions, compaction, and cron jobs.

---

## Step 04: Slash Commands
**Goal**: Give user deterministic control via `/help`, `/skills`, `/session`.

**Key concepts**:
- `Command` ABC with async `execute`
- `CommandRegistry` registers + dispatches commands (with aliases)
- Commands run before normal chat — if input starts with `/`, dispatch and skip LLM
- Design choice: commands may or may not enter message history (user controls vs conversation content)

**Key files**: `src/mybot/core/commands/base.py`, `src/mybot/core/commands/registry.py`, updated `cli/chat.py`

**Why it matters**: LLM-mediated everything is wasteful and non-deterministic for things like "show me the session ID". Slash commands add a deterministic control plane.

---

## Step 05: Compaction
**Goal**: Keep conversations alive past context-window limits.

**Key concepts**:
- `ContextGuard` with `token_threshold` (default 160k = 80% of 200k)
- Token estimation via litellm `token_counter`
- Two-tier strategy: first truncate oversized tool results, then summarize old messages if still over budget
- Rollover into fresh session seeded with summary
- New commands: `/compact` (manual), `/context` (show usage)

**Key files**: `src/mybot/core/context_guard.py` (new), updated `core/agent.py`

**Why it matters**: Without compaction, long sessions hit hard context limits and crash. This makes long-running agents viable.

---

## Step 06: Web Tools
**Goal**: Let agent see beyond local filesystem and training cutoff.

**Key concepts**:
- `WebSearchProvider` ABC: `search(query) -> list[SearchResult]`
- `WebReadProvider` ABC: `read(url) -> ReadResult`
- `websearch` and `webread` tools wrap providers
- Provider pattern lets user swap search backends (Tavily, Brave, etc.)

**Key files**: `src/mybot/provider/web_search/`, `src/mybot/provider/web_read/`, `src/mybot/tools/websearch_tool.py`, `src/mybot/tools/webread_tool.py`

**Why it matters**: LLM training data is stale. Local tools cannot answer "what's latest in X". Web tools close that gap.

---

## Step 07: Event-Driven Architecture
**Goal**: Refactor to pub/sub event bus — decouple message sources from agent execution.

**Key concepts**:
- `EventBus` with subscribe/unsubscribe/publish; internal asyncio queue
- Event types: `InboundEvent` (incoming) and `OutboundEvent` (response)
- `Worker` base class; `SubscriberWorker` for event handlers
- `AgentWorker` subscribes to `InboundEvent`, runs session, publishes `OutboundEvent`
- No user-visible change in this step — pure refactor setting up later phases

**Key files**: `src/mybot/core/events.py`, `src/mybot/core/eventbus.py`, `src/mybot/server/agent_worker.py`

**Why it matters**: CLI-coupled agent cannot serve Telegram, Discord, WebSocket, cron simultaneously. Event bus decouples "where message comes from" from "who processes it" — the keystone for phases 3 and 4.

---

## Step 08: Config Hot Reload
**Goal**: Edit config without restarting the server.

**Key concepts**:
- `Config.reload()` re-reads + deep-merges `config.user.yaml` + `config.runtime.yaml`
- `ConfigHandler` (watchdog `FileSystemEventHandler`) watches workspace for file changes
- Two-layer config: user (human-edited) + runtime (process-written, e.g., source→session bindings)
- Deep-merge semantics let runtime override user without clobbering

**Key files**: `src/mybot/utils/config.py`

**Why it matters**: Long-running servers need config changes (routing, skills, agents) without downtime. Also needed so runtime writes from later steps (session caching, routing) persist across edits.

---

## Step 09: Channels
**Goal**: Receive messages from Telegram, Discord, and any platform.

**Key concepts**:
- `EventSource` ABC: platform-specific identifier (e.g., `platform-telegram:chat:user`)
- `Channel` ABC: `run`/`reply`/`stop` interface per platform
- `ChannelWorker` manages multiple channels, publishes `InboundEvent` to bus
- `DeliveryWorker` subscribes to `OutboundEvent`, routes to the right channel's `reply()`
- Each source maps to one session (cached in `config.runtime.yaml`)
- Outbound event persistence: write-to-disk before dispatch, `ack()` deletes on successful delivery; recovery replays pending events on startup — prevents message loss

**Key files**: `src/mybot/channel/base.py`, `src/mybot/server/channel_worker.py`, `src/mybot/server/delivery_worker.py`, updated `core/eventbus.py`

**Why it matters**: Event bus exists but had no external inputs. Channels make the agent usable from phone/group chat, and the persistence layer makes delivery reliable across crashes.

---

## Step 10: WebSocket
**Goal**: Expose programmatic real-time access via WebSocket.

**Key concepts**:
- `WebSocketWorker` auto-subscribes to `InboundEvent` + `OutboundEvent`, broadcasts to all connected clients as JSON
- FastAPI `/ws` endpoint accepts connections, hands off to worker
- Each event serialized with `type` field for client discrimination
- Works alongside channel workers — WebSocket is just another event sink

**Key files**: `src/mybot/server/websocket_worker.py`, `src/mybot/server/app.py`

**Why it matters**: Third-party UIs, browser clients, and automated integrations need a machine-consumable real-time feed. HTTP/channels not enough.

---

## Step 11: Multi-Agent Routing
**Goal**: Route messages to different specialized agents by source pattern.

**Key concepts**:
- `AgentLoader` discovers multiple `AGENT.md` definitions in workspace
- `RoutingTable` maps source regex → agent, with tiered specificity
- `Binding` auto-computes tier: 0 = exact match, 1 = specific regex, 2 = wildcard; more specific wins
- Fallback to `default_agent` when no binding matches
- New commands: `/route`, `/bindings`, `/agents`

**Key files**: `src/mybot/core/agent_loader.py`, `src/mybot/core/routing.py`, updated `channel_worker.py`

**Why it matters**: Single-agent system cannot specialize. Now Telegram can hit a chat agent while WebSocket hits a memory agent, all from the same process.

---

## Step 12: Cron + Heartbeat
**Goal**: Agent runs scheduled jobs — works while user is asleep.

**Key concepts**:
- `CronDef` in `CRON.md` files: id, agent, schedule (cron expr), prompt, one_off flag
- `CronWorker` ticks every 60s, publishes `DispatchEvent` for due jobs
- `DispatchEvent` / `DispatchResultEvent` — internal event types for non-user-originated execution
- Cron ops (create/list/delete) exposed as a **skill**, not a tool — avoids bloating tool registry
- Heartbeat (OpenClaw only): single, runs in main session at fixed interval without cron expr

**Key files**: `src/mybot/core/cron_loader.py`, `src/mybot/server/cron_worker.py`, workspace `crons/*/CRON.md`

**Why it matters**: Prior steps only react to user messages. Cron gives the agent autonomous time-triggered behavior — reminders, digests, monitoring.

---

## Step 13: Multi-Layer Prompts
**Goal**: Assemble rich system prompt from identity, personality, workspace, runtime, and channel layers.

**Key concepts**:
- `AgentDef` extended with `soul_md` (personality layer, optional)
- `PromptBuilder.build()` concatenates ordered layers:
  1. Identity (`AGENT.md`)
  2. Soul/personality (`SOUL.md`)
  3. Bootstrap context (`BOOTSTRAP.md`, `AGENTS.md`, crons list)
  4. Runtime (agent id, timestamp)
  5. Channel hint (which platform this message came from)
- Extensible: memory layer, project layer, etc. can plug in later

**Key files**: `src/mybot/core/prompt_builder.py`, updated `agent_loader.py`, workspace `AGENT.md`/`SOUL.md`/`BOOTSTRAP.md`/`AGENTS.md`

**Why it matters**: Single-string system prompt doesn't scale. Layered structure lets each concern (identity, workspace map, runtime facts) live in its own file and be composed cleanly.

---

## Step 14: Post Message Back
**Goal**: Agent can initiate messages to user, not just respond.

**Key concepts**:
- `post_message` tool publishes an `OutboundEvent` directly to the bus with `AgentEventSource`
- `DeliveryWorker` (from step 09) picks it up and delivers via the right channel
- Tool only registered in cron-job context — prevents agents from spamming during normal turns
- Enables scheduled reminders, proactive notifications

**Key files**: `src/mybot/tools/post_message_tool.py`

**Why it matters**: Cron could run, but couldn't actually reach the user. This closes the loop — scheduled autonomous behavior can now surface to the right channel.

---

## Step 15: Agent Dispatch
**Goal**: One agent delegates work to another agent.

**Key concepts**:
- `create_subagent_dispatch_tool` factory: tool schema dynamically lists dispatchable agents (excludes caller)
- Mechanism: publish `DispatchEvent` to bus, subscribe a temp `DispatchResultEvent` handler filtering by session_id, await a future, unsubscribe on completion
- `parent_session_id` tracks delegation chain
- Subagent runs as its own session — isolated context
- Alternative patterns discussed: shared task lists, tmux/screen sessions

**Key files**: `src/mybot/tools/subagent_tool.py`

**Why it matters**: Routing only dispatches external messages. For agent→agent collaboration (e.g., chat agent asks memory agent), we need an in-process dispatch via the same event bus.

---

## Step 16: Concurrency Control
**Goal**: Prevent resource exhaustion from too many parallel agent instances.

**Key concepts**:
- `AgentDef.max_concurrency` — configurable per-agent limit
- `AgentWorker` holds a dict of `asyncio.Semaphore`, one per agent id
- `async with sem` blocks when limit reached, releases on session exit
- Alternative granularities: per-source (fair-sharing), per-priority (reserved capacity)

**Key files**: updated `src/mybot/server/agent_worker.py`

**Why it matters**: Multi-channel + cron + dispatch can all fire the same agent at once, burning API quota and CPU. Semaphore gating is the simplest correct answer.

---

## Step 17: Memory
**Goal**: Long-term memory across all conversations.

**Key concepts**:
- Specialized memory agent (`cookie`) accessed via subagent dispatch
- Memory stored as markdown files in workspace: `memories/topics/`, `memories/projects/`, `memories/daily-notes/`
- Main agent (`pickle`) asks cookie via dispatch; cookie reads/writes files
- Alternative patterns: direct tools in main agent, skill-based (grep), vector DB for semantic search

**Key files**: `default_workspace/agents/cookie/AGENT.md`, memory directory convention

**Why it matters**: Compaction preserves context within a session; persistence saves per-session history. Neither gives cross-session recall. Memory agent provides durable, organized user knowledge that survives rollover and new sessions.
