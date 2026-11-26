---
title: "GPUWorks Dual-Scheduler Architecture for Kubernetes"
description: "How GPUWorks uses dual-scheduler awareness to share GPU clusters between Kubernetes and inference engines without hardware lock-in."
categories: ["Engineering", "AI/ML"]
tags: ["kubernetes", "gpus", "llm", "infrastructure", "vllm"]
labels: ["llm-orchestration", "llm-serving"]
---
Managing high-density GPU infrastructure usually forces platform teams to make an uncomfortable trade-off. You either surrender your entire cluster to heavy Kubernetes operators and complex custom resource definitions, or you drift into managing fragmented bare-metal boxes through ad-hoc SSH scripts. The former introduces massive control plane overhead, while the latter leaves expensive silicon sitting completely idle for long stretches of time.

[GPUWorks](https://github.com/flitai-io/gpuworks) presents a compelling alternative to this binary choice. Packaged as a single ~40 MB binary download that includes its entire control plane and web user interface, it offers a lightweight Model-as-a-Service platform designed to coexist directly with existing infrastructure. Rather than forcing you to rip out your operational tooling, GPUWorks embeds into running clusters without replacing the native scheduler or device drivers.

## The mechanics of dual-scheduler coordination

The central architectural challenge when running inference workloads alongside standard cloud-native applications is device contention. Standard Kubernetes scheduling views accelerators through static device plugin allocations. A pod requests `nvidia.com/gpu: 1`, and the K8s scheduler locks that card for the pod's lifespan, regardless of whether memory or compute pipelines are active.

GPUWorks addresses this bottleneck by establishing dual-scheduler visibility between its internal engine and the host cluster. You join an existing Kubernetes cluster by applying a single manifest. The system automatically inspects each node's hardware, detecting accelerators across vendors while preserving whichever device plugins the cluster already uses. It works directly alongside GPU sharing layers like HAMi and Alibaba Cloud ACK GPU sharing.

The coordination mechanism relies on continuous state tracking. The Kubernetes scheduler will not attempt to place pods onto GPUs currently claimed by GPUWorks. Conversely, the GPUWorks engine monitors cluster state and avoids allocating cards that have active Kubernetes pod assignments. When inference traffic spikes, GPUWorks dynamically borrows idle accelerator capacity from the pool. As soon as inference tasks complete, it frees those cards and returns them to the primary Kubernetes scheduler.

## Standardizing heterogeneous hardware pools

Heterogeneity is another operational hurdle in modern AI infrastructure. Enterprise teams rarely run identical hardware across every cluster. They often end up with a mix of data center GPUs, desktop accelerators, and specialized AI chips spread across public and private clouds.

The GPUWorks control plane unifies these disparate hardware targets into a single managed pool. The scheduler handles nodes powered by NVIDIA, AMD Instinct, AMD Ryzen AI Max, Huawei Ascend, T-Head PPU, Hygon DCU, and Apple Silicon side by side. It also aggregates instances fetched across public providers like RunPod, CompShare, AutoDL, Vast.ai, DigitalOcean, and Alibaba Cloud, as well as private cloud virtual machines running on ZStack Cloud.

To handle execution, GPUWorks manages engines like vLLM, SGLang, Strata, and ninfer behind a single OpenAI-compatible HTTP endpoint. To avoid testing complex launch flags for each target manually, the platform bundles a catalog containing 1,295 verified launch specs spanning 404 models. This includes tested configurations for Qwen, DeepSeek, GLM, Kimi, MiniMax, Llama, Gemma, and gpt-oss. The catalog pins explicit engine parameters, container images, launch flags, and weight revisions for each specific GPU architecture.

## Real-world trade-offs and operational edge cases

Despite the elegance of a lightweight single binary, dual-scheduler coexistence introduces distinct failure modes that platform engineers must prepare for.

First, race conditions are inherently harder to eliminate when two independent schedulers negotiate for the same physical resources. If a Kubernetes deployment undergoes a rolling update while GPUWorks is in the middle of dynamically borrowing idle capacity, state synchronization lag can cause temporary allocation conflicts. If the K8s scheduler decides a node needs to host a high-priority API pod, GPUWorks must rapidly yield the hardware. Evicting a large language model with gigabytes of allocated KV cache inside vLLM is far from instantaneous.

Second, while lightweight runtimes focus heavily on host-level device management, specialized edge workloads often require completely different algorithmic tradeoffs. For example, in edge or computer vision contexts, engineers sometimes deploy specialized lightweight algorithms like [HaCI](https://github.com/RenderBear/haci) to bypass heavy neural network dependencies entirely. When orchestrating hybrid inference pools across public cloud providers like RunPod alongside private ZStack Cloud VMs, inter-node network latency creates a severe bottleneck for distributed tensor parallelism. GPUWorks solves local node resource mapping, but cross-datacenter pipeline parallelism still faces basic physical latency constraints.

Finally, shared GPU mechanisms like HAMi or ACK GPU sharing mitigate memory fragmentation, but they do not solve noisy-neighbor issues on the memory bus. If a Kubernetes workload runs compute-heavy matrix operations alongside a GPUWorks inference container on a shared card, PCIe bandwidth contention can tank request latency even if VRAM is safely partitioned.

## Evaluating dual-scheduler architectures

Dual-scheduler coordination offers a practical middle ground between monolithic Kubernetes operator management and unmanaged bare-metal execution. By treating Kubernetes as a partner rather than a system to replace, tools like [gpuworks](https://github.com/flitai-io/gpuworks) demonstrate that control planes can remain compact without sacrificing multi-cloud or multi-vendor capabilities.

The key question to monitor as these engines evolve is how effectively they handle fine-tuning and training jobs across heterogeneous hardware without introducing complex distributed consensus overhead. Engineering leaders evaluating these architectures should test rapid eviction scenarios and memory bus saturation on their specific hardware mixes before routing production traffic.

## Further reading

* [GPUWorks GitHub Repository](https://github.com/flitai-io/gpuworks)
* [HaCI Edge Detection Repository](https://github.com/RenderBear/haci)