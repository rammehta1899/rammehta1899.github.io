---
title: "Designing Asynchronous Agent Runtimes: Stateful Queues and Decoupled Execution"
description: "Explore the architectural shift to asynchronous AI agent runtimes with stateful checkpointing, background daemons, and explicit security boundaries."
categories: ["Engineering", "AI/ML"]
tags: ["platform engineering", "agentic workflows", "langgraph", "software architecture"]
---
We have built a collective habit of treating AI agents like glorified chatbot sessions. You type a prompt, watch the token stream, and wait. If you close the browser window, disconnect from the network, or experience a minor timeout, your execution state vaporizes. That is a fragile approach to engineering. For complex, long-running engineering tasks, agents must operate in the background as durable services. Building this resilience requires decoupling the execution runtime from the user interface.

Recently, we have seen excellent practical patterns emerge in this space. A prominent example is [Pizza Bot](https://github.com/pizza-bot-app/pizza-bot), an open-source local-first inbox for long-running AI work developed at Amazon and released under the Apache 2.0 license. By structuring the runtime around a continuous background daemon, stateful checkpointing, and strict execution queues, we can move away from fragile synchronous sessions. This transition shifts the agent paradigm from an active conversation to a persistent workflow.

## The Architecture of a Decoupled Agent Daemon

To build an agent that survives client disconnects, you must split the execution engine from the interface. In a typical web app, the backend state lives in a remote database, but local-first agent runtimes need a different topology. Pizza Bot implements this by running an independent `api-server` process in the background.

The desktop application, which is built with Electron, forks and supervises this `api-server` sub-process. If the desktop UI closes, the daemon remains running, continuing its operations in the background. The user interface (whether it is the Electron shell, a standard web browser, or a terminal command-line interface) communicates with this background server using standard HTTP and Server-Sent Events (SSE). 

This process model introduces a fascinating engineering tradeoff. By running a local-first API server, you avoid the latency and cost of a remote cloud runtime. However, process supervision becomes a critical point of failure. If the parent Electron shell crashes, it must gracefully hand off or safely terminate the daemon to avoid orphaned processes consuming system resources. Developers building these systems must spend significant effort on local inter-process communication (IPC) and robust signal handling to ensure the background daemon does not leak memory or CPU cycles.

## Stateful Checkpointing and Headless Triggers

How does an agent resume work when you reconnect? The runtime cannot simply keep everything in active memory. It requires a stateful orchestration layer. By building on top of frameworks like DeepAgents and LangGraph, Pizza Bot utilizes durable stateful checkpointing.

Every step of the agent's decision tree, tool execution, and planning phase is written to a persistent state store. If the client disconnects, the agent keeps executing its current step. Once the client reconnects, the UI queries the `api-server` over SSE, reads the checkpoints, and hydrates the interface. This design allows the system to support headless task triggers. You do not need an open interactive conversation to start a job. Instead, cron schedules or incoming webhooks can kick off agent workflows in the background.

This approach stands in contrast to collaborative canvas-based environments. For instance, [Whiteboard](https://github.com/devdotfast/whiteboard) provides an open-source canvas for humans and agents to architect software together, directly linking visual sequence diagrams to underlying code. While Whiteboard focuses on a highly interactive, visual IDE-like experience, asynchronous runtimes focus on background execution where the developer is intentionally absent. These represent two distinct directions for agent tooling: the active workbench and the asynchronous inbox.

## The Dual-Queue Inbox: Unread vs. Action

If agents run in the background, we need a way to manage their output without being overwhelmed by log streams. Watching an agent run thirty tool executions is a waste of developer time. The solution is an inbox-style dual-queue state management system.

Pizza Bot routes its outputs into two primary queues: `Unread` and `Action`.

The `Unread` queue holds completed runs. When an agent finishes a multi-step task, such as generating a report or refactoring a module, the final output lands here. You review it when you are ready, just like an email.

The `Action` queue handles durable human-in-the-loop approval prompts. If an agent needs to execute a potentially destructive tool or make a consequential decision, it pauses execution, writes a checkpoint, and pushes an approval request to the `Action` queue. This request persists across restarts and client sessions. The agent waits indefinitely until you click approve or deny. This pattern is essential for running agents safely in production environments.

## Enforcing Explicit Permission Boundaries

Giving a local background agent access to your system is highly dangerous. An LLM executing tools with arbitrary shell access can easily delete files or leak sensitive credentials. Typical agent frameworks run with the host user's full permissions, which is an unacceptable risk for enterprise adoption.

An asynchronous agent runtime must enforce strict security boundaries. Pizza Bot addresses this by restricting tool filesystem access by default. It receives zero home-directory access out of the box.

Instead, the user must explicitly grant read-only or writable directory paths under the application settings. If the agent attempts to run a tool that reads or writes outside these whitelisted paths, the runtime blocks the execution before it ever reaches the operating system. When evaluating general-purpose agent control planes, such as the [Agno](https://agno.link/gh) runtime designed to turn agents into production-ready microservices, establishing these explicit local security boundaries is paramount. You cannot trust an LLM-driven tool to police itself; sandboxing must be enforced at the runtime level.

## Real-World Tradeoffs of Local Async Runtimes

While the asynchronous inbox pattern is incredibly powerful, it introduces several complex architectural challenges. First, keeping local state synchronized across Electron, web browsers, and CLI clients requires a bulletproof synchronization layer. If a user acts on a prompt in the CLI, the Electron UI must immediately reflect that state transition without causing race conditions in the underlying LangGraph checkpoint database.

Second, running local background services demands significant system resources. Requiring Node.js 24 or newer as a baseline dependency means developers must manage runtime environments on their local machines. If an agent is running a heavy local model via Ollama alongside the background `api-server`, local CPU and memory usage can spike quickly, degrading the overall developer experience.

Finally, debugging non-deterministic agent workflows that run asynchronously is a massive headache. When a synchronous chat agent fails, the developer sees the error immediately in the chat window. When an asynchronous background agent fails halfway through a cron-triggered task, finding the exact checkpoint where the failure occurred requires robust telemetry and structured logging. We are still in the early stages of building standard diagnostic tools for these background state machines.

## References

- [Pizza Bot Repository on GitHub](https://github.com/pizza-bot-app/pizza-bot)
- [Whiteboard IDE on GitHub](https://github.com/devdotfast/whiteboard)
- [Agno Runtime and Control Plane](https://agno.link/gh)