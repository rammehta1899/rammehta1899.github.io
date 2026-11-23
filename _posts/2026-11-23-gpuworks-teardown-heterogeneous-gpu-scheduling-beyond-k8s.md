---
title: "GPUWorks Teardown: Heterogeneous GPU Scheduling Beyond K8s"
description: "A technical teardown of GPUWorks: scheduling heterogeneous GPUs across clouds, coexisting with Kubernetes, and skipping heavy control plane bloat."
categories: ["Engineering", "AI/ML"]
tags: ["gpu-scheduling", "infrastructure", "vllm", "sglang", "distributed-systems"]
labels: [llm-serving, llm-orchestration, agent-scheduling]
---
If you are running multi-cloud LLM inference setups, you have likely felt the operational friction of forcing every accelerator into Kubernetes. Setting up device plugins, managing driver dependencies across spot clouds like RunPod or Vast.ai, and maintaining heavy control planes just to spin up vLLM or SGLang instances creates unnecessary complexity. [GPUWorks](https://github.com/flitai-io/gpuworks) takes a different path. It is a ~45 MB single-binary Model-as-a-Service (MaaS) platform that aggregates heterogeneous accelerator pools across public clouds, private VMs, and local workstations without requiring Kubernetes as a foundation.

The core architectural premise of GPUWorks is straightforward: decouple GPU workload scheduling from container orchestration heavyweights. Written in Go, the entire control plane includes a web management interface, node health tracking, and model management within a self-contained binary. This design choice eliminates the memory footprint and operational complexity of API servers, etcd clusters, and custom resource definitions when your primary objective is executing inference engines efficiently.

For platform teams managing heterogeneous hardware, hardware lock-in is a constant headache. In a single GPUWorks cluster, NVIDIA cards sit right next to AMD Instinct, AMD Ryzen AI Max, Huawei Ascend, Hygon DCU, T-Head PPU, and Apple Silicon devices. Node registration requires a single command on the target host. Once connected, the GPUWorks scheduler queries hardware capabilities, available VRAM, and interconnect topologies automatically.

Matching specialized hardware to optimized inference engines usually demands manual configuration of flags, quantization parameters, and backend execution layers. GPUWorks tackles this by bundling a catalog of 1,235 verified launch specifications covering 392 models, including DeepSeek, Qwen, Llama, Gemma, and GLM variants. When you select a model to deploy, the scheduler picks appropriate target GPUs and launches the corresponding vLLM, SGLang, Strata, or ninfer engine with pinned flags and weight revisions tested for that hardware family.

This lightweight execution model mirrors a broader shift in edge and specialized compute workloads. For instance, algorithmic vision frameworks like [HaCI](https://github.com/RenderBear/haci), which runs edge detection algorithms across Apple Silicon MPS and CUDA environments, rely on direct hardware acceleration without the abstraction penalty of heavy container runtimes. Lightweight infrastructure tooling follows the same philosophy by exposing bare hardware capabilities directly to localized execution engines.

Where GPUWorks becomes particularly interesting for enterprise infrastructure is its approach to Kubernetes coexistence. Many platforms attempt to either replace Kubernetes entirely or run strictly as an operator inside it. GPUWorks offers a middle ground: bi-directional scheduler awareness. By deploying a single Kubernetes manifest, GPUWorks interfaces with existing clusters without replacing existing device plugins like HAMi or Alibaba Cloud ACK GPU sharing.

The mutual visibility mechanism prevents the classic split-brain state that plagues multi-scheduler environments. When Kubernetes allocates a GPU to a pod, GPUWorks detects the reservation and marks the physical card as occupied. Conversely, when GPUWorks provisions an inference engine onto a physical device, it registers a claim that prevents the Kubernetes scheduler from assigning workloads to that card. Idle cards borrowed from a Kubernetes node return to the main pool automatically once the GPUWorks task terminates.

This bi-directional borrowing mechanism is critical for teams operating hybrid setups. You can maintain your primary Kubernetes clusters for steady-state services while using GPUWorks to dynamically tap into idle capacity across bare-metal servers, spot instances on AutoDL, or developer workstations running Apple Silicon. It provides an elastic GPU fabric without forcing external cloud instances into a complex overlay network.

However, choosing a lightweight scheduler over standard Kubernetes tooling comes with concrete tradeoffs that platform leaders must evaluate. First, bypassing Kubernetes CRDs means sacrificing ecosystem integration with GitOps workflows like ArgoCD or Flux. If your deployment pipelines rely heavily on declarative Kubernetes manifests and service meshes, introducing a separate control plane creates an operational split. State management relies on GPUWorks internal datastore, requiring distinct backup and recovery procedures.

Second, while the bundled launch catalog of 1,235 specs reduces initial setup friction, it introduces a dependency on upstream spec maintenance. As vLLM or SGLang release breaking changes to CLI flags or tensor parallel options, outdated specs in the local catalog can lead to failed deployments unless team leads maintain custom launch templates.

Finally, security boundaries in non-Kubernetes environments require disciplined host management. When running the GPUWorks agent directly on bare metal or spot VMs, host-level isolation depends heavily on container runtime boundaries and OS-level network segmentation. Platform teams must ensure proper agent credential rotation and network access controls across public cloud providers like RunPod or Alibaba Cloud.

For platform teams evaluating hybrid inference infrastructure, GPUWorks offers a compelling case study in minimal control planes. It proves that multi-vendor GPU scheduling across heterogeneous hardware does not inherit mandatory Kubernetes overhead. Whether you adopt it to borrow idle GPU cycles from existing clusters or use it as a standalone inference plane across spot instances, the key metric to track is scheduling latency and card utilization under mixed workloads.

## Further reading
- [GPUWorks GitHub Repository](https://github.com/flitai-io/gpuworks)
- [HaCI Edge Detection Architecture](https://github.com/RenderBear/haci)