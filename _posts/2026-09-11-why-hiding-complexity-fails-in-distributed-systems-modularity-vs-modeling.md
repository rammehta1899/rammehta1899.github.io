---
title: "Why Hiding Complexity Fails in Distributed Systems: Modularity vs. Modeling"
description: Platform engineering leaders must distinguish between vertical modularity and horizontal modeling abstractions to build reliable distributed systems.
categories: ["Engineering", "Infrastructure"]
tags: [distributed-systems, platform-engineering, software-architecture, formal-methods, vllm]
---

Most software engineers learn modular design in computer science courses. We learn to isolate implementation details behind clean abstract data types, draw vertical boundaries around modules, and encapsulate internal state. In a single-threaded process running on a local machine, encapsulation works. You call a function, the stack frame executes, and a result returns. Move to distributed infrastructure, however, and those vertical boundaries leak concurrency, network partitions, and subtle timing bugs directly into production code.

In a thoughtful analysis titled [The Two Abstractions of System Design](http://muratbuffalo.blogspot.com/2026/05/the-two-abstractions-of-system-design.html), computer scientist Murat Demirbas highlights a distinction that platform teams frequently miss: modularity abstraction versus modeling abstraction. Modularity draws vertical boundaries to hide implementation details and simplify consumption for callers. Modeling abstraction cuts horizontally across the system, stripping away operational mechanics to reduce system behavior to a minimal behavioral skeleton.

Confusing these two concepts creates fragile infrastructure. When platform engineers build abstractions that attempt to hide concurrency, they do not eliminate distributed edge cases. They merely blind their monitoring tools and force downstream product teams to debug mysterious cascading failures under load.

## Leaky Encapsulation and the Failure of Vertical Boundaries

Joel Spolsky articulated the Law of Leaky Abstractions decades ago. Every non-trivial abstraction leaks the mechanics beneath it. TCP promises a reliable stream of bytes, but when network congestion or dropped packets occur, IP packet retransmissions and latency spikes leak directly into the application layer. File systems present a clean tree of directories, yet block allocation policies and disk head movement dictate write throughput. SQL databases present declarative relational tables, but a missing index forces developers to learn the execution mechanics of the database planner.

In distributed systems, leaks are not minor performance nuisances. They are systemic correctness failures.

If an API hides retry loops, network timeouts, or leader election re-balances behind a synchronous RPC endpoint, it creates an illusion of local call semantics. Under partial network partitions, that clean endpoint stalls, double-writes, or times out. The vertical boundary designed to simplify life for product developers becomes a trap during incident response. Nobody knows which layer owns state consistency when the abstraction fails silently.

Vertical encapsulation attempts to hide internal state to make code easy to consume. Distributed execution continuously exposes state interleavings. When network delays shuffle message ordering, the underlying implementation details do not stay hidden; they become the dominant factor in system correctness.

## Horizontal Reduction: Exposing Interleavings to Prove Invariants

If modularity tries to hide internal state, modeling abstraction takes the opposite path. It radically reduces behavioral scope to expose fine-grained state transitions.

Formal tools like TLA+ do not care about API ergonomics or clean class interfaces. They require engineers to reduce system behavior down to state variables and atomic transitions. Instead of hiding concurrent execution, formal modeling intentionally forces every valid interleaving to surface so that invariant violations can be checked.

Consider how foundational distributed protocols achieve stability. They do not hide complexity. They discard non-essential attributes.

Lamport logical clocks discard wall-clock physical time entirely, reducing execution to a partial ordering of events to establish causality. Linearizability discards physical node replication, retry policies, and network topology, reducing the system to a single virtual execution register to evaluate consistency guarantees.

By stripping away physical realities, modeling abstractions let engineers reason about correctness across infinite possible interleavings. You cannot verify a consensus algorithm by wrapping it in a black-box class library. You verify it by slicing away non-essential execution details until only the state transition invariant remains.

## Concurrency in Modern Infrastructure: The Case of Speculative Decoding

This tension between hiding complexity and modeling explicit execution paths shows up directly in modern AI serving infrastructure. High-throughput serving engines like vLLM cannot afford black-box encapsulation when optimizing target model execution.

Autoregressive token generation in large language models is inherently memory-bandwidth bound because generation advances one committed token at a time. To bypass this bottleneck, vLLM implements speculative decoding, as detailed in their technical analysis on [speculative decoding on AMD GPUs](https://vllm.ai/blog/2026-08-23-speculative-decoding-amd-gpus). Rather than treating LLM inference as a black-box API call, speculative decoding splits generation into a draft-and-verify loop. A lightweight proposal component generates candidate future tokens, and the primary target model verifies those candidates in a single forward pass.

This approach requires exposing intermediate execution behaviors across specialized drafting methods such as EAGLE-3, DFlash, DSpark, and multi-token prediction (MTP). In testing across AMD Instinct MI300X and MI355X GPUs using the ROCm software platform, throughput gains depended heavily on draft checkpoint matching, acceptance behavior, and workload profiles.

Hiding this complexity behind a generic inference endpoint would mask how proposal lengths and draft acceptance rates interact with hardware utilization. Platform teams must model the concurrent interactions between draft models and verification steps rather than pretending inference is a simple synchronous function call.

## Systems Engineering and Platform Release Pragmatics

We see the same architectural reality reflected in production infrastructure releases. As detailed in the [vLLM v0.28.0 release notes](https://github.com/vllm-project/vllm/releases/tag/v0.28.0), platform engineers continuously expose lower-level behavioral controls to optimize distributed execution.

In vLLM v0.28.0, performance improvements like Decode Context Parallel (DCP) support, DSpark confidence-scheduled verification, and an adaptive speculative token budget delivering roughly 60 percent better DSpark time-to-first-token require explicit coordination across hardware execution layers. Default configurations like raising max_num_batched_tokens from 8192 to 16384 reflect empirical calibration of memory boundaries rather than theoretical abstraction boundaries.

If infrastructure engineering was simply about stacking modular APIs, these optimizations would be transparent wrappers. In practice, high-performance systems demand that platform engineers understand memory offloading, CUDA graph capture regions, and asynchronous draft scheduling.

When you encapsulate without modeling, you hide operational signals required to keep systems stable under load.

## Principles for Navigating Abstraction Tradeoffs

Building resilient platforms requires knowing when to encapsulate and when to model explicit reduction. Engineering leaders should enforce three practical rules across their organizations:

Use modularity for developer ergonomics, not fault domain isolation. APIs should simplify syntax and hide business logic boilerplate. They must never hide concurrency semantics, retry budgets, or consistency models from calling services.

Build behavioral models for core distributed paths. Before deploying critical coordination services, write formal TLA+ specifications or build simplified state-machine models. Verify how your system behaves during network partitions, node reboots, and out-of-order delivery.

Expose execution realities to platform operators. High-throughput systems must provide deep observability into internal state transitions. Whether managing database replication or LLM speculative verification, hiding operational execution mechanics behind magic black boxes guarantees catastrophic failures when scale limits are breached.

Vertical encapsulation makes software easy to write. Horizontal behavioral modeling makes distributed systems survive contact with production reality.

## References
- [The Two Abstractions of System Design: Hide or Reduce](http://muratbuffalo.blogspot.com/2026/05/the-two-abstractions-of-system-design.html)
- [Exploring Speculative Decoding in vLLM on AMD GPUs](https://vllm.ai/blog/2026-08-23-speculative-decoding-amd-gpus)
- [vLLM Release v0.28.0 Notes](https://github.com/vllm-project/vllm/releases/tag/v0.28.0)