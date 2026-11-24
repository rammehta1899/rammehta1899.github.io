---
title: "HaCI Apple M4 Metal Benchmark: Edge Vision Latency"
description: "Benchmarking bio-inspired edge vision processing with HaCI on Apple M4 hardware using custom Metal kernels, profiling memory scaling and execution latency."
categories: ["Engineering", "AI/ML"]
tags: ["apple-silicon", "metal", "edge-computing", "computer-vision", "benchmarks"]
labels: ["agent-runtimes", "modularity-and-modeling"]
---

As platform engineering teams push perception pipelines closer to the physical capture source, offloading computer vision tasks from central clusters to local runtime hardware has become an essential architectural pattern. When building local intelligence runtimes, raw compute throughput is only half the battle. Memory allocation curves, host-to-device transfer overhead, and hardware kernel efficiency determine whether an edge workload is economically and operationally viable. 

To evaluate how bio-inspired edge detection scales on unified memory architectures, I benchmarked [HaCI (Harmonic Contour Integration)](https://github.com/RenderBear/haci) on Apple M4 hardware using custom Metal performance kernels.

HaCI takes a bio-inspired approach to visual contour extraction, replacing standard single-pass convolutional edge detectors with a structured four-stage processing pipeline. Rather than treating boundary detection as a simple pixel-intensity gradient problem, the system mimics early visual cortical processing to preserve structural continuity across noisy regions.

## The HaCI Processing Pipeline

Understanding the computational bottleneck requires examining how HaCI structures its frame transformations:

1. Pixel-level Gain Control (`haci/L0.py`): Normalizes local dynamic contrast and stabilizes luminance across input RGB frames.
2. Cell-level Moments (`haci/L1.py`): Computes local $z_2$ moments to extract orientation angles ($\theta$), energy ($E$), and contrast metrics ($C$).
3. Seed Association Fields (`haci/seed.py`): Applies non-linear response suppression, same-orientation surround modulation, and flank suppression to determine cell splat densities ($\rho$).
4. Ridge Projection Rendering (`haci/renderer.py`): Uses a learned ridge projection renderer to generate the final contour map.

When invoked with the `--device mps` flag, PyTorch dispatches the renderer's custom splatting operations through Metal shaders implemented in `haci/splat_metal.py`. Moving these custom splatting loops from PyTorch native primitives to hand-tuned Metal kernels prevents expensive CPU-GPU serialization roundtrips during vector scatter operations.

## Apple M4 Performance Breakdown

Profiling execution speed and memory consumption across varying frame resolutions reveals sharp non-linear scaling behaviors. Tests were run on Apple M4 hardware running Python 3.12 via PyTorch Metal Performance Shaders (MPS), executing `baselines/speed.py`.

At the baseline evaluation resolution of 481×321 pixels, HaCI processes frames in 35 ms while drawing 123 MB of host RAM and occupying 1.1 GB of GPU VRAM.

Scaling to 720p (1280×720 pixels) increases total frame latency to 201 ms. Host RAM consumption grows modestly to 171 MB, while GPU memory allocation remains static at 1.1 GB.

At 1080p (1920×1080 pixels), execution time reaches 457 ms per frame. Host RAM footprint rises to 233 MB, while GPU VRAM usage doubles to 2.2 GB.

The memory footprint data reveals an important systemic operational detail. While host memory scales predictably alongside raw frame buffers (123 MB to 233 MB), GPU VRAM exhibits step-function behavior. The Metal runtime allocates a fixed 1.1 GB base footprint for shader state, scratch buffers, and parameter tensors at 481×321 and 720p. Crossing the threshold to 1080p forces the PyTorch MPS memory allocator to double its slab reservation to 2.2 GB to accommodate intermediate ridge projection splat tensors.

Latency scaling presents an even steeper penalty. Moving from 481×321 to 1080p represents roughly a 13.4x increase in frame area, but total execution time increases by 13.05x (from 35 ms to 457 ms). At 457 ms per frame, un-downscaled 1080p processing caps throughput at roughly 2.1 frames per second. For real-time frame rates (30 FPS), operating HaCI directly on full HD frames without spatial tiling or resolution downscaling is non-viable on desktop class edge runtimes. 

At 481×321, however, 35 ms latency sits just under the 33.3 ms window required for 30 FPS stream processing. That makes low-resolution downscaling prior to spatial contour extraction mandatory for real-time video streams.

## Quality Metrics on the BRIND Benchmark

Processing speed must be weighed against mathematical boundary precision. Evaluated against the BRIND test set (200 evaluation images, with models trained on the 300-image BRIND training set) using the standard Berkeley boundary benchmark protocol (99 thresholds, maximum boundary distance matching parameter `maxDist` = 0.0075), HaCI yields strong structural boundary performance:

* Optimal Dataset Scale (ODS): 0.7160 at a cut threshold of 0.74
* Precision: 0.6841
* Recall: 0.7509
* Optimal Image Scale (OIS): 0.7383
* Average Precision (AP): 0.7598

For raw soft edge maps (`c_eval`) prior to non-maximum suppression (NMS) thinning (`s_eval`), the algorithm reaches an ODS of 0.7136.

The precision-recall spread (0.6841 Precision vs 0.7509 Recall) highlights the core tradeoff of bio-inspired seed association fields. The flank suppression and orientation fields excel at stitching together low-contrast, broken contours that standard gradient filters drop. This boosts recall significantly. However, in heavily textured background areas, those same association fields occasionally over-propagate weak signals, pulling down overall precision.

## Edge Runtimes and Cluster Orchestration

From a platform infrastructure perspective, running dedicated edge computer vision models locally on Apple Silicon changes how we think about heterogeneous compute clusters. Rather than routing raw high-bitrate camera feeds back to centralized Kubernetes worker pools running heavy NVIDIA GPUs, lightweight execution layers can perform local edge extraction.

Modern cluster tools like [GPUWorks](https://github.com/flitai-io/gpuworks) showcase how lightweight management layers are evolving to control these mixed environments. Systems that unify NVIDIA, AMD Instinct, and Apple Silicon nodes under a shared scheduling control plane allow platform engineers to route lightweight vision jobs to unified memory workstation nodes while reserving high-powered discrete cards for large-scale model inference. 

Knowing the exact memory step-functions of models like HaCI (such as its jump to 2.2 GB VRAM at 1080p) is essential when configuring resource limits for local containerized agents or sidecars. A worker node configured to run multiple edge tasks side by side will experience unexpected OOM evictions if frame resolution dynamically shifts from 720p to 1080p without pre-allocated headroom.

Custom Metal kernels (`haci/splat_metal.py`) demonstrate that bio-inspired visual algorithms can achieve acceptable real-time performance on commodity unified memory hardware, provided image scaling stays bounded. The remaining challenge for platform teams is managing the trade-off between host memory headroom, VRAM step allocations, and resolution scaling when embedding these perception loops into production pipelines.

## References

* [HaCI GitHub Repository](https://github.com/RenderBear/haci) - Harmonic Contour Integration source code, PyTorch implementation, and MPS Metal splat kernels.
* [GPUWorks Repository](https://github.com/flitai-io/gpuworks) - Lightweight GPU cluster management and heterogenous accelerator orchestration platform.