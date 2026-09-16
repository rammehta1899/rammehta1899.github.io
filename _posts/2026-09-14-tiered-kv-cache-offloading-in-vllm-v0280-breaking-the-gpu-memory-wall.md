---
title: "Tiered KV Cache Offloading in vLLM v0.28.0: Breaking the GPU Memory Wall"
description: How vLLM v0.28.0 breaks the GPU VRAM wall using native NVMe KV cache offloading, canonical CPU layouts, and Model Runner V2 disaggregation.
categories: ["Engineering", "AI/ML"]
tags: ["vllm", "platform-engineering", "kv-cache", "llm-serving", "gpu-memory"]
---
If you run high-throughput LLM serving infrastructure, your biggest operational bottleneck isn't FLOP availability. It's GPU memory capacity. As context windows expand toward hundreds of thousands of tokens, the Key-Value (KV) cache grows until it exhausts H100 or MI300X VRAM long before compute hardware reaches high utilization. The open source community addressed this core limitation in the [vLLM v0.28.0 release](https://github.com/vllm-project/vllm/releases/tag/v0.28.0), moving the engine from a GPU-bound memory model to a multi-tier storage hierarchy. By pairing native NVMe disk offloading with disaggregated execution, vLLM shifts how production systems handle active and cold attention states.

Every active sequence requires retaining KV tensors across all layer heads for every historical token. With long prompts or agentic multi-turn chats, the memory footprint scales linearly with sequence length and batch size. When VRAM fills up, serving engines face a bad choice: drop concurrent requests or aggressively evict cached prefixes. Eviction destroys prefix reuse, forcing costly re-prefill cycles when the next turn arrives. This architectural wall makes scaling token-heavy applications economically painful for platform engineering teams.

## Breaking the VRAM Limit with Storage Hierarchies

The core of the v0.28.0 update is native NVMe disk offloading (#49644). Instead of treating host RAM as the only secondary fallback when VRAM fills up, vLLM now supports a multi-tier storage architecture where KV blocks flow from VRAM to host CPU memory and down to local NVMe drives. For custom enterprise storage backends or proprietary distributed memory fabrics, out-of-tree secondary tier managers can be loaded dynamically at runtime via the `module_path` parameter (#51007). This decoupling means infrastructure engineers don't have to fork vLLM just to plug in a specialized NVMe-oF array or pooled CXL cache system.

Offloading state across storage tiers introduces a tricky systems problem: tensor layout mismatch. If a KV block is cached on CPU memory while running a model across four GPUs using tensor parallelism, what happens if the scheduler later reallocates that request to a single GPU or a different pipeline parallelism topology? In earlier architectures, restoring offloaded blocks across different parallel layouts required expensive re-sharding or outright cache invalidation.

## Canonical CPU Layouts and Topology-Agnostic Caching

vLLM v0.28.0 solves this by enforcing a canonical CPU layout (#48414) for offloaded blocks. Tensors are normalized into a single parallelism-agnostic format before hitting host memory or NVMe storage. When a sequence hits a cache match on the secondary tier, the engine reshapes the restored blocks to fit whatever GPU tensor or pipeline parallel split is actively serving the decode step. This unblocks dynamic scaling in large clusters, allowing cache hydration to function independently of worker node rank topologies.

Secondary storage carries latency penalties. NVMe reads, even across fast PCIe Gen 5 lanes, cannot match internal HBM bandwidth. If a cache fetch stalls waiting for disk I/O, the entire decode batch risks starvation. Addressing this reality, v0.28.0 adds explicit handling for partial secondary-tier load results (#50321). If an offloaded KV block is partially missing or delayed, the engine doesn't halt the request or fail the inference step. It falls back to computing missing attention blocks on the fly while proceeding with available cached tokens.

## Resilient Decode Loops and Observability

Operating a multi-tier cache without deep visibility is a recipe for silent throughput degradation. The release introduces dedicated tiering metrics (#48798) exposed via Prometheus endpoints. Operators can now track secondary cache hit rates, host-to-device transfer latencies, NVMe I/O saturation, and evicted block counts in real time. If your secondary cache hit rate drops while PCIe bus saturation spikes, you know your offload threshold is set too aggressively for your physical disk bandwidth.

Tiered KV storage is only one part of the optimization puzzle in vLLM v0.28.0. The release also matures Model Runner V2, bringing Encoder/Prefill/Decode (E/P/D) disaggregation (#38390) into stable production readiness. Splitting prefill workers from decode workers prevents long prefill sequences from blocking tight decode step iterations. By combining E/P/D disaggregation with dynamic weight offloading (#51413), operators can dynamically allocate GPU VRAM between active parameter weights and KV cache pools depending on current traffic spikes.

## Model Runner V2 and Execution Disaggregation

This release also strengthens speculative decoding workflows. As detailed in the team's [analysis of speculative decoding on AMD GPUs](https://vllm.ai/blog/2026-08-23-speculative-decoding-amd-gpus), draft-and-verify strategies yield significant speedups but require tight coordination between target models and proposal mechanisms. In v0.28.0, DFlash2 brings local convolution and candidate selection (#52816), while DSpark adds confidence-scheduled verification (#47808). Async scheduling is now automatically enabled for draft models (#48341), reducing draft-stage overhead during token verification steps.

Underneath the engine, control plane performance receives a major upgrade with a standalone Rust frontend and gRPC pipeline. This includes explicit data-parallel rank routing (#51178), allowing high-concurrency clusters to dispatch incoming API calls directly to target worker ranks without crossing Python GIL bottlenecks. For massive architectures like Kimi-K3, optional shared-expert sharding (#50912) saves roughly 17 GiB of memory per GPU, freeing substantial VRAM for KV block allocation. The default `max_num_batched_tokens` has been raised from 8192 to 16384 (#51726), reflecting confidence in these combined memory savings.

## Evaluating Real-World Tradeoffs and Operational Limits

Secondary tiering works best when prompt prefixes exhibit high temporal locality, such as multi-turn system prompts or fixed RAG context blocks. On workload patterns dominated by unique, unpredictable prompts, secondary cache writes generate high NVMe drive write amplification and PCIe bus overhead without delivering meaningful hit rates. Furthermore, breaking changes like migrating bitsandbytes out-of-tree (#43529), removing `calculate_kv_scales` (#49389), and enforcing Transformers 5.15.0 (#51668) require platform teams to audit custom deployment scripts before rolling out v0.28.0 into production pipelines.

The architectural direction of vLLM is clear: moving from single-device VRAM management to cluster-wide, tiered memory orchestration. As models grow and context lengths stretch further, the real test for platform engineers will be balancing NVMe drive endurance against cache hit ratios while tuning E/P/D node ratios. The full set of features and pull requests can be tracked on the [vLLM v0.28.0 release notes](https://github.com/vllm-project/vllm/releases/tag/v0.28.0) and the ongoing experimental results shared in the [vLLM speculative decoding benchmark report](https://vllm.ai/blog/2026-08-23-speculative-decoding-amd-gpus).

## Further Reading

- [vLLM v0.28.0 Release Notes](https://github.com/vllm-project/vllm/releases/tag/v0.28.0)
- [Exploring Speculative Decoding in vLLM on AMD GPUs](https://vllm.ai/blog/2026-08-23-speculative-decoding-amd-gpus)