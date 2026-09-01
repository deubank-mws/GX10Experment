# GX10 Experiment

A hands-on experiment building a two-node local AI system from two ASUS GX10 / NVIDIA DGX Spark systems.

The goal is to learn what is practical with compact Blackwell systems: high-speed GPU networking, distributed inference, very large open-weight models, local APIs, gateways, observability, and low-maintenance operations.

## Current milestone — August 31, 2026

The experiment has progressed from distributed model inference to a usable local AI service with a strong focus on making it behave more like an appliance than a fragile lab environment.

Current capabilities include:

- Two ASUS GX10 / NVIDIA DGX Spark systems, each with an NVIDIA GB10 and 128 GB unified memory
- DGX Spark 7.5.0, NVIDIA driver 580.173.02, CUDA 13.0
- Docker 29.2.1 with working GPU containers
- Direct ConnectX-7 RoCE links between the systems at 200 Gb/s
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
- NVIDIA DCGM Exporter running on both systems
- Prometheus scraping both GPU exporters plus LiteLLM metrics
- Grafana backed by persistent storage and file-based provisioning

## Current service path

```text
Browser
  |
  +--> Forge Fast
  |
  +--> Forge Deep
          |
          v
      Open WebUI
          |
          v
      LiteLLM
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

Both user-facing modes intentionally use the same Qwen backend: a fast/default interaction mode and a deeper reasoning mode. This keeps the system simple rather than multiplying model runtimes and maintenance work.

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

## Reboot / recovery lesson — August 31, 2026

A real reboot exposed an important operational distinction: **a working distributed inference stack is not automatically a reboot-durable service**.

The model cache, GPU networking, and supporting services survived, but the distributed model launch still depended on state and processes that had originally been created interactively.

The runtime containers were rebuilt as persistent named containers with restart policies, cluster SSH and MPI were revalidated, and the model was relaunched from the existing local checkpoint rather than downloaded again.

After repair, the complete service chain was validated again:

```text
Qwen backend              OK
LiteLLM model route       OK
Real gateway inference    OK
Open WebUI                OK
Grafana                   OK
Prometheus                OK
Both GPU metric targets   UP
Gateway metric target     UP
```

Grafana configuration was also moved toward file-based provisioning so monitoring is less dependent on manual dashboard setup.

## Operations goal

The project is now optimizing for simplicity rather than adding infrastructure for its own sake.

The desired experience is:

```text
Power on both systems
        |
        v
wait for services/model
        |
        v
open the browser
        |
        v
choose Fast or Deep
```

Normal daily operation should require **zero terminal commands**.

The next engineering milestone is therefore supervised automatic startup and recovery for the distributed Qwen service, followed by a full reboot test requiring no SSH/manual intervention.

## Inference baseline

```text
Wall-clock time:     46.118 seconds
Prompt tokens:       45
Completion tokens:   700
Total tokens:        745
Approx. throughput:  15.2 completion tokens/sec end-to-end
Short-prompt TTFT:   ~0.25 seconds
```

## What comes next

1. Supervise distributed Qwen startup/recovery automatically.
2. Add simple status/start/stop/repair commands.
3. Complete a dual-node reboot test with zero manual recovery.
4. Keep the browser model picker intentionally limited to Fast + Deep.
5. Only then move back to useful research/status workflows and integrations.

## Security

No passwords, SSH private keys, Hugging Face tokens, API keys, LiteLLM master/virtual keys, database credentials, or other secrets belong in this repository. Any organizational or regulated data should only be used where the environment and workflow have been explicitly approved for that data.

---

This public repository intentionally documents architecture, milestones, lessons learned, and benchmark results without exposing private configuration, internal addressing, user accounts, or credentials.
