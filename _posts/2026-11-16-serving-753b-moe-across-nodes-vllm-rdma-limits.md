---
title: "Serving 753B MoE Across Nodes: vLLM & RDMA Limits"
description: "Evaluating multi-node tensor parallelism for GLM-5.3 NVFP4 across DGX Stations reveals severe throughput cliffs past concurrency 16."
categories: ["AI/ML", "Engineering"]
tags: ["vllm", "moe", "rdma", "dgx", "llm-serving"]
labels: ["llm-serving", "caching"]
---

When model footprint exceeds single-host GPU memory, platform engineers face unforgiving network synchronization constraints. The NVIDIA GLM-5.3 NVFP4 checkpoint, derived from Z.ai's mixture-of-experts model, illustrates this memory bottleneck clearly. Storing its 753 billion parameters requires roughly 464 GB of VRAM. Although sparse gating activates only 40 billion parameters per token, every expert parameter must remain resident in VRAM because routing decisions change token by token. A single NVIDIA GB300 GPU provides 252 GB of HBM3e capacity, making single-card hosting impossible without severe CPU offloading penalties. 

To bypass local memory limits while keeping inference fast, deployments split model execution across separate physical hosts. As covered in the [NVIDIA Technical Blog deployment guide](https://developer.nvidia.com/blog/when-one-gpu-isnt-enough-serving-glm-5-3-across-two-nvidia-dgx-stations-with-vllm/), running GLM-5.3 NVFP4 across two NVIDIA DGX Stations connected over ConnectX-8 RDMA enables tensor parallelism (TP=2) across nodes. Each GB300 GPU holds approximately half the checkpoint tensors. The vLLM engine manages distributed execution, presenting a unified OpenAI-compatible API endpoint hosted on the primary station while both GPUs evaluate matrix operations simultaneously.

## Hardware Topology and Inter-Node Synchronization

Splitting tensor parallelism across distinct workstations alters the underlying communications model. Inside a single chassis, high-bandwidth interconnects like NVLink handle layer-by-layer all-reduce operations with minimal latency. Moving tensor parallelism across two physical DGX Stations pushes all intermediate activations onto network rails. 

The software stack requires Linux environments with Docker, the NVIDIA Container Toolkit, and validated GPU-memory RDMA connectivity across both network interface cards. Disabling CPU weight offload ensures all parameter lookups execute within HBM3e. When an incoming prompt hits the shared endpoint, vLLM broadcasts token state variables across the ConnectX-8 interconnect. Every layer requires synchronization. If network latency fluctuates or queue contention builds on the RDMA link, generation stalls across the entire model pipeline.

Smaller, dense models avoid this inter-node communication complexity entirely. For instance, according to the [Unsloth Qwen3.8 documentation](https://unsloth.ai/docs/models/qwen3.8), a 27B parameter model like Qwen3.8-27B fits into 17 GB of system memory using 4-bit Dynamic V3.1 quants, achieving agentic execution on standard desktop setups. However, when application requirements demand frontier-scale capabilities, distributed MoE architectures become unavoidable.

## Memory Optimizations: FP8 Cache, Prefixing, and MTP

Fitting a 753B parameter checkpoint onto dual GB300 nodes leaves tight memory margins for request context. To maximize active request capacity within the remaining HBM3e space, vLLM applies three concurrent runtime optimizations:

* **FP8 KV-Cache Storage:** Quantizing key-value memory from standard floating-point precision down to FP8 cuts context memory overhead in half, freeing VRAM for longer sequence lengths and higher concurrency.
* **Prefix Caching:** Shared prompt fragments, common system instructions, and multi-turn chat headers are cached in VRAM. Subsequent requests referencing matching prefix tokens bypass redundant prefill computations.
* **Multi-Token Prediction (MTP):** Speculative decoding mechanisms predict multiple candidate tokens per forward pass, increasing output generation efficiency per network synchronization step.

These memory techniques optimize memory usage, but they do not eliminate the physical throughput limits imposed by cross-node network interfaces under high concurrent load.

## The Concurrency Cliff in AIPerf Benchmarking

Benchmark results using AIPerf on the SPEED-Bench Coding dataset highlight the performance limits of cross-node tensor parallelism over ConnectX-8 RDMA. At moderate load, the system delivers high processing speeds, but performance drops sharply as concurrent request volume scales up.

| Concurrency Level | Output Throughput (tok/s) | Mean Request Latency (s) | System Performance Profile |
| :--- | :--- | :--- | :--- |
| **Concurrency 16** | 607.8 | Baseline | Peak operational throughput |
| **Concurrency 32** | 385.0 | 453.5 | Severe saturation cliff |

At concurrency 16, the system achieves its peak output throughput of 607.8 tokens per second. Inter-node RDMA transfers remain within bandwidth limits, allowing vLLM to schedule prefill and decode phases efficiently across both DGX Stations.

Pushing concurrency to 32 breaks this operational balance. Output throughput drops by 36.6 percent down to 385.0 tokens per second, while mean request latency surges to 453.5 seconds. 

This throughput cliff occurs because cross-node tensor parallelism requires synchronous all-reduce transfers at every network layer. As context buffers fill and request queues grow, inter-node message frequency saturates the ConnectX-8 RDMA interface. The system spends more time waiting for cross-node buffer clearing than computing matrix multiplications.

## Operational Takeaways for Platform Engineers

Serving frontier MoE models across physical host boundaries demands precise load regulation rather than simple autoscaling.

Distributed algorithms in other domain spaces rely heavily on structural boundaries to maintain processing speeds. A clear parallel appears in spatial image processing; the lightweight algorithm detailed in the [Harmonic Contour Integration repository](https://github.com/RenderBear/haci) relies on localized gain control and neighborhood pixel structures to avoid costly global state exchanges. In distributed LLM serving, managing communication boundaries is equally critical.

When deploying multi-node vLLM setups for production endpoints, engineering teams should implement three concrete system guardrails:

First, place hard concurrency throttles in front of cross-node endpoints. Allowing concurrent requests to double from 16 to 32 does not produce graceful queue degradation. It causes a complete performance collapse where users experience multi-minute wait times for single completions.

Second, separate prefill and decode workloads across hardware boundaries where possible. High-concurrency prefill phases generate massive, bursty network payloads during intermediate matrix passes, directly disrupting token decode latency on shared tensor-parallel nodes.

Third, run synthetic workload sweeps up to point-of-failure limits before establishing production SLAs. Theoretical network bandwidth figures never align perfectly with real-world layer synchronization overhead.

## References

* [NVIDIA Technical Blog: Serving GLM-5.3 Across Two NVIDIA DGX Stations with vLLM](https://developer.nvidia.com/blog/when-one-gpu-isnt-enough-serving-glm-5-3-across-two-nvidia-dgx-stations-with-vllm/)
* [Unsloth Documentation: Qwen3.8-27B Local Deployment](https://unsloth.ai/docs/models/qwen3.8)
* [GitHub: RenderBear / haci (Harmonic Contour Integration)](https://github.com/RenderBear/haci)