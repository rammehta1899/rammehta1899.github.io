---
title: Building Least-Privilege Filesystem Sandboxing for Autonomous Agent Runtimes
description: How Pizza Bot isolates background AI agent execution using explicit directory whitelisting, process separation, and tool-scoped subagents.
categories: ["Engineering", "AI/ML"]
tags: [platform engineering, agent runtime, security, sandbox, architecture]
---
Local AI agents promise to execute multi-step software engineering tasks in the background while you focus on higher-level system design. You trigger a background run, switch contexts, and collect the output later. But when local agent runtimes transition from passive prompt generation to active workspace modification, security defaults become a pressing concern. Most local agent frameworks run with unrestricted access to the developer's home directory. They inherit whichever credentials, SSH keys, and configuration files belong to the user account launching the process.

Granting unconstrained filesystem permissions to an autonomous background process is dangerous. A prompt injection inside a third-party repository or a misconfigured file-edit tool can corrupt local projects or exfiltrate private credentials. Protecting host systems requires moving away from implicit system privileges toward strict, deny-by-default access control models.

A pragmatic implementation of this security model can be seen in [Pizza Bot](https://github.com/pizza-bot-app/pizza-bot), an open source inbox for background AI work originally developed at Amazon and released under the Apache 2.0 license. Pizza Bot demonstrates how to pair zero-default directory permissions with process isolation and specialized subagent architectures.

## Deny-By-Default Directory Permissioning

Instead of assuming full access to the home directory, a secure agent runtime must implement zero default filesystem permissions across local environments. Pizza Bot revokes default home-directory access entirely. When starting a local runtime instance, the application cannot read or write to any local path until a human operator explicitly configures permission targets under Settings > Files.

Paths are explicitly granted as either read-only or writable directories. If an agent attempts to open SSH credentials or access repositories outside the approved list, the runtime interceptor drops the file operation immediately. This design isolates execution boundaries. An agent assigned to edit a frontend component inside a web application repository cannot traverse up into adjacent directories or inspect system secrets unless that specific folder was explicitly whitelisted in settings.

Crucially, these access boundaries apply regardless of which LLM provider fuels the agent logic. Whether the user configures Amazon Bedrock, Anthropic, Google Gemini, OpenAI, OpenRouter, or a local model via Ollama under Settings > Providers, the underlying filesystem constraints remain identical. Model providers handle reasoning, but local application code enforces the authorization boundaries.

## Process Isolation: Decoupling UI from Execution

Enforcing permissions cleanly requires structural process separation. Combining UI rendering, IPC event loops, and arbitrary tool execution into a single application process creates an unmaintainable attack surface. Pizza Bot structures its local desktop architecture by separating the front-end user experience from the core runtime engine.

The Electron desktop application shell forks and supervises a completely separate api-server child process. The React user interface running in Electron, the web browser interface, and the terminal CLI all communicate with this background api-server process using HTTP and Server-Sent Events (SSE). You can inspect this architecture directly in the [Pizza Bot repository](https://github.com/pizza-bot-app/pizza-bot) open-source codebase.

Supervising the backend execution engine as a distinct sub-process provides several operational benefits. It allows the agent runtime to maintain stateful execution even when the user closes the desktop UI or disconnects a terminal session. The system relies on a stateful DeepAgents and LangGraph runtime to drive background work.

When long-running agent tasks complete or hit human-in-the-loop approval requests, state checkpoints persist across client disconnects. Completed tasks automatically move to the Unread queue, while pending approval requests collect in Action. Crucially, persistent worker processes write checkpoints to designated application state folders without requiring root access or elevated operating system privileges.

## Tool-Scoped Subagents and Capability Isolation

Directory sandboxing addresses path access, but it does not address tool abuse. If a monolithic agent loop possesses file-writing tools, shell execution tools, network utilities, and browser automation simultaneously, any failure in prompt evaluation grants full capability access to whatever input triggered the error.

To limit this blast radius, skills are architected as tool-scoped subagents rather than global agent capabilities. Instead of giving a top-level agent unrestricted access to every registered tool, capabilities are isolated into specialized sub-routines.

When a primary agent determines that a specific task is needed, it delegates execution to a subagent that only carries the specific tools required for that single skill. Progress for these subagents streams directly into the UI Activity panel. If a documentation-parsing skill is invoked, its subagent environment lacks write tools and shell execution primitives entirely. Even if injected text attempts to force a command execution payload, the subagent runtime literally lacks the capability handle to invoke it.

## Practical Security Tradeoffs and Engineering Realities

While application-level path permissioning and subagent tool isolation represent significant steps forward for local developer tooling, engineering leaders must evaluate the remaining trade-offs.

First, application-level path checks in a managed runtime like Node.js (which requires Node 24 or newer in Pizza Bot's build scripts) are only as strong as the process boundary enclosing them. If an agent tool invokes arbitrary shell scripts or compiles native binaries, path validation code inside JavaScript can be bypassed unless tied directly to OS-level kernel sandboxing such as Linux namespaces, cgroups, bubblewrap, or macOS App Sandbox rules. Application path checks prevent accidental file destruction and stop standard path traversal vectors, but they do not replace true operating system containerization for untrusted code execution.

Second, explicit permission models introduce user friction. Engineers expect local CLI tooling to execute immediately against the current working directory. Demanding that users open Settings > Files to explicitly whitelist directories before background work can begin adds setup latency. If permissioning mechanisms are too cumbersome, developers will look for bypasses or default back to running unsandboxed root agents.

The next milestone for autonomous local runtimes will be pairing soft application-level path permissions with transparent, zero-overhead OS sandboxing. Until then, enforcing explicit directory boundaries and tool-scoped subagents remains the most practical baseline for running autonomous agents safely on developer machines.

## References

* [Pizza Bot GitHub Repository](https://github.com/pizza-bot-app/pizza-bot)