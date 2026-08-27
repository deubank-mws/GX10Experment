# GX10 Experiment

A hands-on experiment building a two-node local AI system from two ASUS GX10 / NVIDIA DGX Spark systems.

The goal is to learn what is practical with compact Blackwell systems: high-speed GPU networking, distributed inference, very large open-weight models, local APIs, agents, and observability.

## Current milestone — August 27, 2026

The two-node system now has a working browser-to-model path:

- Two ASUS GX10 / NVIDIA DGX Spark systems, each with an NVIDIA GB10 and 128 GB unified memory
- DGX Spark 7.5.0, NVIDIA driver 580.173.02, CUDA 13.0
- Docker 29.2.1 with working GPU containers
- Two direct ConnectX-7 RoCE links between the systems, each negotiating at 200 Gb/s
- NCCL 2.30.7-1 and OpenMPI 4.1.6
- Successful two-node 16 GiB NCCL `all_gather` validation with zero errors
- TensorRT-LLM multi-node containers with working container-to-container MPI/SSH
- `nvidia/Qwen3-235B-A22B-FP4` downloaded and verified on both nodes
- Qwen running with TP=2 across both GB10s
- OpenAI-compatible `/v1/models` and `/v1/chat/completions` endpoints validated
- 700-token benchmark: 46.118 seconds, about 15.2 completion tokens/sec end-to-end
- Short-prompt streaming TTFT baseline: about 0.25 seconds
- Open WebUI running on the head node and successfully serving browser chat from a Windows workstation

## Runtime lesson learned

The first deployment attempt used TensorRT-LLM `1.3.0rc13`. The two-node model launch segfaulted in the multi-node leader/worker processes. Because networking, NCCL, MPI, and the model checkpoint had already been validated independently, the runtime was rolled back rather than rebuilding the cluster.

Using TensorRT-LLM `1.2.0rc6`, the same Qwen TP=2 deployment loaded successfully and served the first completion.

This is a useful reminder that in distributed AI systems the newest runtime is not always the best runtime for a specific model/hardware combination.

## Why Qwen3-235B-A22B-FP4?

The experiment intentionally uses a model large enough to make meaningful use of both systems rather than treating the nodes as separate small-model hosts. The NVIDIA FP4 checkpoint is well matched to Blackwell hardware and provides a practical test of distributed inference on two DGX Spark systems.

## Networking validation

Both active ConnectX interfaces report RDMA `ACTIVE`, physical link `LINK_UP`, and 200000 Mb/s link speed. A 16 GiB NCCL `all_gather` completed successfully across the two GB10 GPUs with zero wrong or out-of-bounds values before large-model inference was attempted.

## Inference and benchmark milestone

The TensorRT-LLM server exposes an OpenAI-compatible API on the head node. The end-to-end path is:

```text
Client request
   -> TensorRT-LLM
   -> Qwen3-235B-A22B-FP4
   -> TP=2 across two NVIDIA GB10 systems
   -> generated response
```

Initial benchmark:

```text
Wall-clock time:     46.118 seconds
Prompt tokens:       45
Completion tokens:   700
Total tokens:        745
Approx. throughput:  15.2 completion tokens/sec end-to-end
```

A separate short-prompt streaming test measured roughly **0.25 seconds time-to-first-token**. Larger-context TTFT remains a future benchmark.

## Browser milestone

Open WebUI was deployed on the head node with persistent local storage and connected to the TensorRT-LLM OpenAI-compatible endpoint. A Windows browser successfully selected the Qwen model and returned a chat response.

The experiment now works through a normal browser rather than only `curl`:

```text
Windows browser
   -> Open WebUI
   -> TensorRT-LLM API
   -> Qwen3-235B-A22B-FP4
   -> dual GX10 cluster
   -> browser response
```

## What comes next

1. Create a cleaner curated model identity such as **Forge 235B** in Open WebUI.
2. Confirm persistent chat history and browser UX.
3. Add API-key organization and per-workload usage monitoring.
4. Add GPU, memory, container, and network observability.
5. Experiment with research and status-oriented agents.
6. Add tool integrations where appropriate and authorized.
7. Compare additional open-weight models and runtime versions on the same hardware.

## Security

No passwords, SSH private keys, Hugging Face tokens, API keys, or other credentials belong in this repository. Any organizational or regulated data should only be used where the environment and workflow have been explicitly approved for that data.

---

This public repository intentionally documents architecture, milestones, lessons learned, and benchmark results without exposing private configuration or credentials.