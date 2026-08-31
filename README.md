# GX10 Experiment

A hands-on experiment building a two-node local AI system from two ASUS GX10 / NVIDIA DGX Spark systems.

The goal is to learn what is practical with compact Blackwell systems: high-speed GPU networking, distributed inference, very large open-weight models, local APIs, gateways, agents, observability, and operational recovery.

## Current milestone — August 31, 2026

The experiment has progressed from distributed model inference to a usable, monitored local AI service:

- Two ASUS GX10 / NVIDIA DGX Spark systems, each with an NVIDIA GB10 and 128 GB unified memory
- DGX Spark 7.5.0, NVIDIA driver 580.173.02, CUDA 13.0
- Docker 29.2.1 with working GPU containers
- Two direct ConnectX-7 RoCE links between the systems, each negotiating at 200 Gb/s
- Successful two-node NCCL validation with zero errors
- TensorRT-LLM 1.2.0rc6 multi-node runtime
- `nvidia/Qwen3-235B-A22B-FP4` cached and verified on both nodes
- Qwen running with tensor parallelism across both GB10s
- OpenAI-compatible model API validated
- 700-token benchmark: 46.118 seconds, about 15.2 completion tokens/sec end-to-end
- Short-prompt streaming TTFT baseline: about 0.25 seconds
- Open WebUI serving browser chat
- LiteLLM acting as the monitored OpenAI-compatible gateway
- Dedicated virtual-key attribution working for browser chat
- Qwen fast/default `/no_think` behavior and on-demand `/think` behavior validated
- NVIDIA DCGM Exporter running on both systems
- Prometheus scraping both GPU exporters plus LiteLLM metrics
- Grafana connected with initial dual-node GPU panels working

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

Observability:

```text
GPU exporter (node 1) --+
                         |
GPU exporter (node 2) --+--> Prometheus --> Grafana
                         |
LiteLLM metrics ---------+
```

## Runtime lesson learned

The first deployment attempt used TensorRT-LLM `1.3.0rc13`. The two-node model launch segfaulted in the multi-node leader/worker processes. Because networking, NCCL, MPI, and the model checkpoint had already been validated independently, the runtime was rolled back rather than rebuilding the cluster.

Using TensorRT-LLM `1.2.0rc6`, the same Qwen TP=2 deployment loaded successfully and served completions.

This is a useful reminder that in distributed AI systems the newest runtime is not always the best runtime for a specific model/hardware combination.

## Reboot / recovery lesson — August 31, 2026

A controlled real-world reboot exposed a second operational lesson: **a working distributed inference stack is not the same thing as a reboot-durable service**.

The browser, gateway, database, and monitoring services restarted normally, but the distributed TensorRT/Qwen backend did not automatically return.

Investigation showed that:

- the TensorRT image and full model cache were still present on both systems;
- RDMA devices and persistent ConnectX networking survived;
- the original multi-node runtime containers had been disposable rather than durable service containers;
- the model server itself had been launched interactively, so restarting a helper/SSH container alone could not recreate the Qwen process;
- some MPI/SSH/model runtime configuration lived only inside the old containers and had to be restored.

The multi-node containers were recreated as named containers with Docker restart policies, cluster SSH was revalidated, and MPI again returned both nodes successfully. The large model was then relaunched from the existing local checkpoint rather than downloaded again.

The next infrastructure milestone is to eliminate the manual recovery steps entirely.

## Why Qwen3-235B-A22B-FP4?

The experiment intentionally uses a model large enough to make meaningful use of both systems rather than treating the nodes as separate small-model hosts. The NVIDIA FP4 checkpoint is well matched to Blackwell hardware and provides a practical test of distributed inference on two DGX Spark systems.

## Inference and interaction behavior

Initial benchmark:

```text
Wall-clock time:     46.118 seconds
Prompt tokens:       45
Completion tokens:   700
Total tokens:        745
Approx. throughput:  15.2 completion tokens/sec end-to-end
```

A separate short-prompt streaming test measured roughly **0.25 seconds time-to-first-token**. A later monitored large-context request also showed sub-second TTFT.

Open WebUI provides the normal browser experience. A curated model wrapper routes through the gateway rather than directly to the backend server.

For simple chat, Qwen prefers `/no_think`, reducing unnecessary reasoning delay. Explicit `/think` still enables deeper reasoning when requested.

## API gateway and observability

LiteLLM sits between Open WebUI and TensorRT-LLM. A dedicated browser/UI virtual key was validated in request logs, providing a foundation for future workload separation such as research, status, GitHub, automation, and testing.

LiteLLM exposes Prometheus metrics for requests, failures, tokens, latency, time-to-first-token, and gateway overhead.

NVIDIA DCGM Exporter runs on both GB10 systems. Prometheus successfully scrapes both GPU exporters plus LiteLLM metrics, and Grafana provides the first dual-node GPU panels for temperature, utilization, and power.

## What comes next

The highest-priority work is now **startup and recovery automation**, not adding more AI features.

The target experience is:

```text
Power on both systems
        |
        v
network + interconnect ready
        |
        v
runtime containers start automatically
        |
        v
preflight verifies node reachability, RDMA, SSH, and MPI
        |
        v
Qwen model service starts automatically
        |
        v
health endpoint becomes ready
        |
        v
browser AI service is usable
```

Planned improvements:

1. Persist MPI, SSH, and model runtime configuration outside the containers.
2. Add simple start/stop/status/preflight commands.
3. Supervise the worker/runtime and model launcher with systemd.
4. Remove interactive authentication from normal startup using protected local credentials or a tested offline-cache workflow.
5. Add health checks and conservative self-recovery.
6. Perform a full controlled reboot test with no manual SSH recovery.
7. Finish Grafana request/token/latency panels.
8. Then resume research/status agents and additional integrations.

## Security

No passwords, SSH private keys, Hugging Face tokens, API keys, LiteLLM master/virtual keys, database credentials, or other secrets belong in this repository. Any organizational or regulated data should only be used where the environment and workflow have been explicitly approved for that data.

---

This public repository intentionally documents architecture, milestones, lessons learned, and benchmark results without exposing private configuration, internal addressing, user accounts, or credentials.
