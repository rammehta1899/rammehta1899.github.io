---
title: "Architecting Asynchronous Agent Workflows: Checkpoints and Approvals in Pizza Bot"
description: "Learn how Pizza Bot uses stateful LangGraph runtimes, SSE, and explicit action queues to build durable, asynchronous background AI agents."
categories: ["Engineering", "AI/ML"]
tags: ["ai agents", "langgraph", "system architecture", "asynchronous", "platform engineering"]
---
Most software teams building LLM applications make the same architectural mistake early on. They design agent execution around a synchronous, uninterrupted client connection. The user types a prompt into a chat window, the frontend opens an HTTP connection or WebSocket, and the client sits idle waiting for a chain of tool calls to complete. The moment the user closes their laptop lid, switches tabs on a mobile browser, or drops Wi-Fi for three seconds, the workflow breaks. Long-running automation cannot rely on ephemeral client connections. The open-source project [Pizza Bot on GitHub](https://github.com/pizza-bot-app/pizza-bot) offers a practical alternative by decoupling client interfaces from a persistent execution runtime.

Developing production background agents requires shifting your mental model from interactive chat sessions to persistent job processing. When an agent needs five minutes to clone a repository, analyze code dependencies, run test suites, and draft a pull request, keeping an active UI connection alive is a reliability nightmare. State management must move down to the platform tier.

## Decoupling Execution from the User Interface

Pizza Bot addresses this by decoupling its frontends from its backend process model. Developed originally at Amazon and open-sourced under the Apache 2.0 license, the application uses a standalone `api-server` daemon that executes tasks independently of the user interface. Whether you interact with the system through an Electron desktop shell, a web browser, or a terminal CLI, the client acts solely as an observer and control interface. Communication happens over standard HTTP endpoints and Server-Sent Events (SSE) streams.

If you initiate a complex task in the desktop app and immediately quit the Electron shell, the underlying `api-server` continues running. It executes LLM invocations, runs tool calls, and updates state checkpoints without caring if a client is listening. When you reopen the UI hours later, the client reconnects to the local server, fetches the latest thread states over SSE, and renders the updated execution tree.

Under the hood, this requires a Node.js 24 or newer environment. When running the development environment via `npm run dev`, the Vite frontend and Electron desktop shell start together while the shell forks and supervises its own `api-server` process. This supervisor pattern mirrors how packaged desktop builds operate, ensuring local background tasks aren't tied to the browser process lifecycle.

## Durable Checkpoints and Human-in-the-Loop Queues

Executing agents asynchronously is only half the battle. The harder engineering challenge is handling human intervention when an agent reaches a high-stakes decision point. If an agent wants to delete a cloud storage bucket, apply a database migration, or post a comment to a public repository, you cannot let it proceed unchecked. Yet blocking execution while waiting for a live user to press an approval button in an active chat window destroys the benefits of background execution. The codebase in the [Pizza Bot repository](https://github.com/pizza-bot-app/pizza-bot) handles this through a stateful runtime built on DeepAgents and LangGraph.

Instead of holding an active thread in memory, the engine persists the entire execution state to disk at every workflow node. When an agent reaches a step that requires human verification, the engine writes a durable checkpoint and pauses the thread. The task isn't marked as failed or active. It is routed directly to a dedicated global queue.

Pizza Bot divides finished and pending work into two clean abstractions:
- Unread Queue: Houses runs that finished execution in the background while the user was away. You can review the step-by-step logs and output artifacts at your convenience without clogging active workspace folders.
- Action Queue: Houses durable approval requests. When an agent hits an explicit safety boundary or requires user judgment, it pauses execution and posts an item here.

This queue design changes how human-in-the-loop operates. Instead of interrupting your current focus with modal dialogs or requiring you to keep a prompt window open, approving an action becomes an asynchronous operation. You open your Action inbox, review the proposed tool payload, grant or deny permission, and the engine resumes the LangGraph thread from its saved checkpoint.

## Scheduled Triggers and Scoped Subagents

Background automation gets even more interesting when workflows don't originate from a user conversation at all. By supporting background triggers via cron schedules or inbound webhooks, agents can kick off execution based on external events. A nightly cron job can spin up an agent thread to scan codebase vulnerabilities, generate summary reports, and push actionable findings straight to your Action queue before you log in every morning.

To keep complex agent tasks manageable, Pizza Bot relies on skill-based delegation. Rather than overloading a single master system prompt with dozens of disparate tool definitions, tools are scoped into specialized subagents. When the main agent delegates work to a specific skill, that subagent executes within its own constrained context. Progress across these scoped tasks streams directly to a real-time Activity panel, giving engineers clear visibility into nested execution trees.

Flexibility at the inference layer is another operational requirement. The runtime integrates across multiple model providers, allowing teams to swap backends based on cost, latency, or compliance constraints. Under Settings > Providers, users can configure credentials for Amazon Bedrock, Anthropic, Google Gemini, OpenAI, OpenRouter, or local LLM instances via Ollama.

## Pragmatic Security and Operational Tradeoffs

Giving local AI agents access to execute commands on an engineer's workstation creates immediate security risks. Pizza Bot enforces a strict zero-trust default posture for host access. By default, the application receives zero access to your user home directory or local filesystems. You must explicitly grant access to specific read-only or writable directory paths under Settings > Files.

While this local sandboxing prevents rogue agents from sweeping through sensitive local SSH keys or configuration files, it introduces real operational friction. If an agent needs to work across multiple software repositories, engineers have to manually curate granted folder lists. Forget to add a path, and the run fails silently at a tool boundary.

There are other practical tradeoffs to weigh before adopting this pattern. Because background processing relies on the decoupled daemon model, the `api-server` process must remain running continuously on the host system. If your local machine reboots or the daemon dies under memory pressure, pending cron triggers fail to fire until the process is manually restarted. Requirements like Node.js 24+ can also complicate integration for enterprise environments standardized on older LTS releases. Packaging policies introduce deployment considerations as well: while macOS installers are signed and notarized, Linux packages attached to releases are unsigned, requiring engineering teams to manually verify downloads against provided SHA256SUMS. You can examine the complete release setup directly in the [Pizza Bot repository source](https://github.com/pizza-bot-app/pizza-bot).

## What to Watch Next

The shift toward asynchronous, queue-driven AI automation is necessary for operational reliability. However, open questions remain about how local-first decoupled architectures like Pizza Bot will bridge the gap to multi-tenant cloud environments. How do we preserve explicit folder grant security and persistent state checkpointing when shifting execution from local desktop daemons to distributed Kubernetes clusters? Watch how stateful agent frameworks evolve their checkpoint serialization specs and authorization models over the next year.

## Further Reading
* [Pizza Bot GitHub Repository](https://github.com/pizza-bot-app/pizza-bot)