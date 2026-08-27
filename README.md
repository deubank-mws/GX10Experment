# GX10 Experiment

A hands-on experiment building a two-node local AI system from two ASUS GX10 / NVIDIA DGX Spark systems.

The goal is to learn what is practical with compact Blackwell systems: high-speed GPU networking, distributed inference, very large open-weight models, local APIs, gateways, agents, and observability.

## Current milestone — August 27, 2026

The experiment has progressed from distributed model inference to a usable, monitored local AI service:

- Two ASUS GX10 / NVIDIA DGX Spark systems, each with an NVIDIA GB10 and 128 GB unified memory
- DGX Spark 7.5.0, NVIDIA driver 580.173.02, CUDA 13.0
- Docker 29.2.1 with working GPU containers
- Two direct ConnectX-7 RoCE links between the systems, each negotiating at 200 Gb/s
- NCCL 2.30.7-1 and OpenMPI 4.1.6
- Successful two-node 16 GiB NCCL `all_gather` validation with zero errors
- TensorRT-LLM multi-node containers with working container-to-container MPI/SSH
- `nvidia/Qwen3-235B-A22B-FP4` downloaded and verified on both nodes
- Qwen running with TP=2 across both GB10s
- OpenAI-compatible model API validated
- 700-token benchmark: 46.118 seconds, about 15.2 completion tokens/sec end-to-end
- Short-prompt streaming TTFT baseline: about 0.25 seconds
- Open WebUI successfully serving browser chat
- LiteLLM proxy added as the monitored OpenAI-compatible gateway
- Dedicated virtual-key attribution working for browser chat
- Qwen fast/default `/no_think` behavior and on-demand `/think` behavior validated
- NVIDIA DCGM Exporter running on both systems
- Prometheus successfully scraping both GPU exporters plus LiteLLM metrics
- Grafana connected to Prometheus with initial dual-node GPU panels working

## Current service path

```text
Browser / future clients
        |
        v
Open WebUI
        |
        v
LiteLLM gateway + virtual keys
        |
        v
TensorRT-LLM
        |
        v
Qwen3-235B-A22B-FP4
        |
        v
TP=2 across two NVIDIA GB10 systems
```

The observability path is now:

```text
GPU exporter (node 1) --+
                         |
GPU exporter (node 2) --+--> Prometheus --> Grafana
                         |
LiteLLM metrics ---------+
```

## Runtime lesson learned

The first deployment attempt used TensorRT-LLM `1.3.0rc13`. The two-node model launch segfaulted in the multi-node leader/worker processes. Because networking, NCCL, MPI, and the model checkpoint had already been validated independently, the runtime was rolled back rather than rebuilding the cluster.

Using TensorRT-LLM `1.2.0rc6`, the same Qwen TP=2 deployment loaded successfully and served the first completion.

This is a useful reminder that in distributed AI systems the newest runtime is not always the best runtime for a specific model/hardware combination.

## Why Qwen3-235B-A22B-FP4?

The experiment intentionally uses a model large enough to make meaningful use of both systems rather than treating the nodes as separate small-model hosts. The NVIDIA FP4 checkpoint is well matched to Blackwell hardware and provides a practical test of distributed inference on two DGX Spark systems.

## Networking validation

Both active ConnectX interfaces report RDMA `ACTIVE`, physical link `LINK_UP`, and 200000 Mb/s link speed. A 16 GiB NCCL `all_gather` completed successfully across the two GB10 GPUs with zero wrong or out-of-bounds values before large-model inference was attempted.

## Inference and benchmark milestone

Initial benchmark:

```text
Wall-clock time:     46.118 seconds
Prompt tokens:       45
Completion tokens:   700
Total tokens:        745
Approx. throughput:  15.2 completion tokens/sec end-to-end
```

A separate short-prompt streaming test measured roughly **0.25 seconds time-to-first-token**. A later monitored large-context request also showed sub-second TTFT, while longer Qwen reasoning phases were separately visible in the UI.

## Browser and model-behavior milestone

Open WebUI provides the normal browser experience. A curated Forge model wrapper routes through the gateway rather than directly to the backend model server.

For simple chat, Qwen is configured to prefer `/no_think`, reducing unnecessary reasoning delay on trivial requests. Explicit `/think` still enables deeper reasoning when requested.

This creates a practical fast/default + deep/on-demand interaction pattern without changing the underlying 235B model.

## API gateway and usage attribution

LiteLLM was added between Open WebUI and TensorRT-LLM. A dedicated browser/UI virtual key was created and validated in LiteLLM request logs.

This provides the foundation for future per-workload separation such as research, status, GitHub, automation, and testing without sharing one credential across every workload.

LiteLLM also exposes Prometheus metrics for requests, failures, input/output tokens, total tokens, latency, time-to-first-token, and gateway overhead.

## GPU and gateway observability

NVIDIA DCGM Exporter is running successfully on both GB10 systems. Validated telemetry includes GPU temperature, utilization, power draw, clock information, and other available DCGM fields.

Prometheus successfully scrapes all three current metric sources:

- GPU exporter on node 1
- GPU exporter on node 2
- LiteLLM metrics

Grafana is connected to Prometheus and initial panels are working for both systems:

- GPU temperature
- GPU utilization
- GPU power draw

An initial LiteLLM gateway-request panel has also been created. Completing the request/token/latency section is the next dashboard task.

## What comes next

1. Finish the Grafana AI-gateway section for requests, failures, tokens, latency, and TTFT.
2. Add CPU, RAM, disk, filesystem, container, and network telemetry for both systems.
3. Add more interconnect/RDMA visibility where useful.
4. Make the distributed model service survive a controlled restart with documented/automated recovery.
5. Create virtual keys only as additional real workloads are introduced.
6. Build the first research and status-oriented workflows.
7. Add tool integrations where appropriate and authorized.
8. Compare additional open-weight models and runtime versions on the same hardware.

## Security

No passwords, SSH private keys, Hugging Face tokens, API keys, LiteLLM master/virtual keys, database credentials, or other secrets belong in this repository. Any organizational or regulated data should only be used where the environment and workflow have been explicitly approved for that data.

---

This public repository intentionally documents architecture, milestones, lessons learned, and benchmark results without exposing private configuration, internal addressing, user accounts, or credentials.
