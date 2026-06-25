# Agent Client Protocol (ACP) — Comprehensive Guide for NotebookLM Podcast

**Focus topics for this source document:**
- Supported scenarios (local vs remote, transports, workspace models, auth)
- Hosting models and the Agent Harness concept
- Tool invocation and use (lifecycle, permissions, file system, terminals, MCP)
- Skills / capabilities ownership (what the Agent owns vs what the Client provides)
- Related core mechanics: Prompt Turn, Session lifecycle, Capabilities negotiation, Extensibility

**Source:** Curated and synthesized from the official `docs/` folder of https://github.com/agentclientprotocol/agent-client-protocol (primarily stable v1 protocol documentation).

**Best companion resource:** https://agentclientprotocol.com/ (interactive docs)  
**LLM-friendly index:** https://agentclientprotocol.com/llms.txt

> This document is intentionally detailed and structured for NotebookLM podcast generation. It emphasizes clarity, responsibilities of each side (Client vs Agent), concrete examples, and the mental model of how ACP enables safe, capable AI coding agents.

---

## 1. High-Level Mental Model

The Agent Client Protocol (ACP) is a **JSON-RPC 2.0-based wire protocol** that decouples code editors (Clients) from AI coding agents (Agents).

**Core idea:**  
The editor stays in control of the user’s environment (files, terminals, permissions, UI). The Agent is a specialized AI-powered process that can request actions and receive rich context, but never directly touches the user’s machine without going through the Client.

This creates a clean separation:
- **Client** = owns the environment, user interface, permissions, and execution surface.
- **Agent** = owns the intelligence, planning, tool-use reasoning, and LLM orchestration.

ACP standardizes how these two sides talk so any ACP-compatible editor can work with any ACP-compatible agent (and vice versa).

---

## 2. Supported Scenarios

### 2.1 Local Agents (Most Mature)
- Agent runs as a **child process** of the editor.
- Communication: **JSON-RPC over stdio** (standard input/output).
- Very low latency, simple to launch, no network involved.
- Ideal for agents that need tight integration with the local filesystem and tools the editor already has access to.
- The editor (Client) is responsible for spawning the agent process with the correct environment and arguments.

### 2.2 Remote Agents (In Progress / Partial Support)
- Agent runs on a different machine, in the cloud, or in a container.
- Planned transports: **HTTP** and **WebSocket**.
- A working group is actively standardizing remote transports.
- Challenges being addressed: authentication, streaming large responses, reconnect/retry semantics, and secure tunneling of file/terminal operations.

**Current state (v1):** Remote is possible in principle via custom transports, but the official "remote story" (especially file system and terminal proxying) is still maturing.

### 2.3 Workspace & Session Models
- **Single workspace root** is the baseline (the `cwd` or primary directory passed at session creation).
- **Additional directories** support is being stabilized (`additionalDirectories` capability). This allows agents to work across multiple folders in a monorepo or multi-root workspace.
- Sessions are **isolated conversation threads**. Each session has its own history, state, and can be resumed (if the agent supports `loadSession`).

### 2.4 Authentication Scenarios
- Agents can declare supported auth methods during `initialize`.
- Common pattern: Agent requires authentication before creating sessions (e.g., API keys, OAuth, enterprise SSO).
- `logout` capability allows ending an authenticated session cleanly.
- Authentication is negotiated once at connection time, not per-session in the basic model.

---

## 3. Hosting Models & The Agent Harness

### 3.1 What is an "Agent Harness"?

An **Agent Harness** is the runtime layer that turns a raw LLM + tool-calling loop into a proper ACP-speaking server.

It is responsible for:
- Implementing the ACP server side (listening on stdio or HTTP).
- Handling the `initialize` handshake and capability negotiation.
- Managing session lifecycle (`session/new`, `session/prompt`, `session/update` notifications, cancellation, etc.).
- Orchestrating the LLM calls, tool execution, and state.
- Emitting rich `session/update` notifications (messages, tool calls, plans, diffs, etc.).
- Respecting permissions requested via the Client.

Official SDKs provide harness/reference implementations so developers don’t have to write the JSON-RPC boilerplate from scratch.

### 3.2 Common Hosting Patterns

| Pattern                    | Description                                                                 | Typical Use Case                     | Transport     | Maturity |
|---------------------------|-----------------------------------------------------------------------------|--------------------------------------|---------------|----------|
| **Subprocess (Local)**    | Editor spawns the agent binary/process                                      | Desktop IDE agents                   | stdio         | Very High |
| **Long-running Server**   | Agent runs as a persistent service (local or remote)                        | Shared agents, team agents           | HTTP / WS     | Medium    |
| **Container / Cloud**     | Agent runs in Kubernetes, Fly.io, Modal, etc.                               | Scalable / multi-tenant agents       | HTTP / WS     | Emerging  |
| **Hybrid**                | Lightweight local proxy that forwards to a remote powerful agent            | Privacy + power combination          | stdio + HTTP  | Experimental |

### 3.3 SDK Harness Examples
- **Rust SDK** (`agent-client-protocol` crate): Provides both Client and Agent runtime traits. You implement the trait methods; the crate handles the wire protocol.
- **TypeScript / Python / Java / Kotlin SDKs**: Similar pattern — abstract base classes or traits for the Agent side that you extend.
- The harness usually includes helpers for:
  - Building `session/update` notifications
  - Handling permission requests
  - Streaming content blocks
  - Plan reporting
  - Cancellation tokens

**Key takeaway for podcast:** The "magic" of ACP is that the harness + Client together create a safe sandbox. The Agent never directly executes code or edits files — it only *requests* actions through well-defined ACP methods.

---

## 4. Tool Invocation and Use (The Heart of Agent Capability)

### 4.1 Philosophy
Tools are **not** owned or executed by the Agent directly.  
The **Client owns the execution surface**. The Agent only *proposes* tool calls and receives results.

This is fundamental to ACP’s security model.

### 4.2 Tool Call Lifecycle (High Level)

1. **Capability Advertisement** (at `initialize`)
   - Agent declares what it can do (`promptCapabilities`, `mcpCapabilities`, etc.).
   - Client declares what execution surfaces it offers (`fs.*`, `terminal`, etc.).

2. **Inside a Prompt Turn**
   - User sends a prompt (text + optional rich content blocks).
   - Agent reasons, possibly calls LLM multiple times.
   - When the LLM wants to use a tool, the Agent sends a `session/update` notification containing a **Tool Call** (with ID, name, arguments, status).
   - Client sees the tool call and can show it in the UI (e.g., "Agent wants to run `git status`").
   - For sensitive tools, Client shows a **permission request** (`session/request_permission`).
   - User approves or denies.
   - If approved, Client executes the tool (reads file, runs terminal command, calls MCP server, etc.).
   - Client sends the result back to the Agent via another `session/update` (or as part of the final `session/prompt` response).

3. **Rich Updates**
   - Agents can stream partial results: thinking text, message chunks, plan updates, file diffs, terminal output, etc.
   - All of this flows through `session/update` notifications.

### 4.3 Built-in Tool Surfaces

**File System (optional capability)**
- `fs/read_text_file`
- `fs/write_text_file`
- Paths must be absolute.
- Client can enforce allow-lists, show diffs before write, etc.

**Terminals (optional capability)**
- Full terminal lifecycle: `terminal/create`, `terminal/output`, `terminal/wait_for_exit`, `terminal/kill`, `terminal/release`.
- Useful for running build commands, tests, linters, etc.
- Output can be streamed back to the Agent and user.

**MCP (Model Context Protocol) Bridging**
- Agents can declare `mcpCapabilities` (http, sse).
- This allows the Agent to connect to existing MCP servers through the Client (or directly in some models).
- ACP re-uses many MCP JSON shapes for compatibility.

### 4.4 Permission Model (Critical for Trust)
- Not every tool call requires explicit user approval — depends on Client policy and sensitivity.
- `session/request_permission` is the standardized way for Agents to ask.
- Clients can batch permissions, remember decisions, or apply policies ("always allow read in this workspace").
- This is one of the biggest UX differentiators between different ACP Clients.

---

## 5. Skills / Capabilities — Who Owns What?

This is one of the most important mental models in ACP.

### 5.1 Capabilities Negotiation (`initialize`)

During `initialize`, both sides declare what they support:

**Agent advertises (examples):**
- `promptCapabilities`: Can the prompt include images? Audio? Embedded resources?
- `loadSession`: Can it resume previous sessions?
- `auth.logout`: Does it support ending auth?
- `mcpCapabilities`
- Session capabilities (`delete`, `additionalDirectories`)
- Custom capabilities via `_meta`

**Client advertises (examples):**
- `fs.readTextFile` / `fs.writeTextFile`
- `terminal` (full terminal surface)
- Custom capabilities

**Result:** After `initialize`, both sides know exactly what the other can do. No more guessing or hard-coded assumptions.

### 5.2 "Skills" Ownership Summary

| Area                        | Primarily Owned By | Notes |
|----------------------------|--------------------|-------|
| **Intelligence & Planning** | Agent             | LLM calls, reasoning, tool selection, plan generation |
| **Tool Execution**          | Client            | Actually reading/writing files, running terminals, calling external APIs |
| **Permission UI & Policy**  | Client            | Asking the user, applying allow/deny rules |
| **Rich Output (diffs, messages, plans)** | Agent     | Agent decides what to show; Client renders it |
| **Session History & State** | Both (coordinated)| Agent maintains conversational state; Client can persist/load sessions |
| **Authentication**          | Agent (with Client help) | Agent declares methods; Client may handle token storage/UI |
| **Custom Tools / Skills**   | Extensible        | Via custom methods (`_myTool`) + `_meta` or MCP bridging |

**Key Insight:**  
ACP does **not** give the Agent arbitrary "skills" that run with the Agent’s privileges. Instead, it gives the Agent a **standardized way to request** actions from a privileged Client that the user already trusts.

This is why ACP feels safer than agents that try to run with full user permissions or that require the user to paste long context manually.

---

## 6. Prompt Turn Deep Dive (Where Tools Actually Happen)

A **Prompt Turn** is the core unit of work:

1. Client sends `session/prompt` with user message + optional content blocks.
2. Agent starts processing (may involve multiple LLM calls).
3. Agent emits zero or more `session/update` notifications:
   - `content` blocks (assistant messages, thoughts, diffs)
   - `tool_call` / `tool_call_update`
   - `plan` updates
   - `available_commands` (slash commands)
   - Mode changes, etc.
4. Eventually the Agent finishes and sends the final response to `session/prompt`.
5. Client can send `session/cancel` at any time.

This streaming + update model is what enables the rich, responsive UX that modern AI coding agents need (showing plans, diffs, terminal output live, etc.).

---

## 7. Extensibility & Custom Skills

ACP is designed to grow without breaking compatibility:

- **Custom methods**: Prefix with `_` (e.g., `_myCompany/runCustomAnalysis`).
- **Custom data**: `_meta` fields on almost every object.
- **Custom capabilities**: Advertised in `initialize` under `_meta` or dedicated fields.
- **MCP bridging**: Reuse the huge existing ecosystem of MCP tools/servers.
- **Content blocks**: Can be extended with new types.

This means teams can add proprietary skills/tools while still working with any ACP Client.

---

## 8. Error Handling, Cancellation & Robustness

- Standard JSON-RPC 2.0 errors.
- `session/cancel` notification allows interrupting long-running turns.
- Agents should be resilient to cancellation and clean up resources.
- Clients should handle partial results gracefully when a turn is cancelled.

---

## 9. Summary — Mental Model for the Podcast

**ACP’s core value proposition:**

It creates a **standardized, safe, rich communication channel** between two powerful but very different systems:

- The **Client** (editor) — which has privileged access to the user’s machine and trust.
- The **Agent** (AI harness + LLM) — which has intelligence and the ability to plan and use tools.

By clearly separating **who owns execution** (Client) from **who owns reasoning** (Agent), ACP enables:

- True interoperability across editors and agents.
- A consistent, powerful UX for tool use, permissions, and rich output.
- Safe extensibility without giving agents unrestricted power.
- A path to both local high-performance agents and future remote/cloud agents.

The "Agent Harness" is the crucial piece of software that turns a raw LLM into something that speaks fluent ACP and can participate in this collaborative dance with the editor.

---

**End of consolidated source document.**

*Prepared for NotebookLM podcast generation on ACP supported scenarios, hosting, tool use, capabilities ownership, and agent harness concepts. Based on official ACP v1 documentation from the `docs/` folder.*

For the absolute latest details or to explore RFDs/v2 proposals, always cross-reference https://agentclientprotocol.com/.