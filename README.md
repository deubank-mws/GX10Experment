# GX10 Experiment

A hands-on experiment building a two-node local AI system from two ASUS GX10 / NVIDIA DGX Spark systems.

The project is now focused less on proving that distributed inference works and more on making the result behave like a **simple appliance**: power on, wait, open the browser, and use it.

## Current milestone — August 31, 2026

Current capabilities include:

- Two ASUS GX10 / NVIDIA DGX Spark systems, each with an NVIDIA GB10 and 128 GB unified memory
- DGX Spark 7.5.0, NVIDIA driver 580.173.02, CUDA 13.0
- Docker 29.2.1 with working GPU containers
- Direct ConnectX-7 RoCE links between the systems at 200 Gb/s
- Successful two-node NCCL validation with zero errors
- TensorRT-LLM 1.2.0rc6 multi-node runtime
- `nvidia/Qwen3-235B-A22B-FP4` cached and verified on both nodes
- Qwen running with tensor parallelism across both GB10s
- OpenAI-compatible model API
- LiteLLM gateway with usage attribution
- Open WebUI browser interface
- Dual-node NVIDIA DCGM telemetry
- Prometheus + Grafana monitoring
- systemd-supervised distributed model startup
- simple local health/recovery commands
- lightweight Markdown-based persistent AI memory

## Service path

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

The normal model picker is intentionally kept small. The goal is useful capability without turning the environment into a model zoo.

## Performance baseline

```text
Wall-clock time:     46.118 seconds
Prompt tokens:       45
Completion tokens:   700
Total tokens:        745
Approx. throughput:  15.2 completion tokens/sec end-to-end
Short-prompt TTFT:   ~0.25 seconds
```

## Runtime lesson learned

An early deployment with a newer TensorRT-LLM release failed in the multi-node leader/worker processes. Rolling back to the known-good 1.2.0rc6 runtime produced a stable distributed Qwen deployment.

The broader lesson: validate each layer independently and do not assume the newest runtime is automatically the best runtime for a specific model/hardware combination.

## Reboot / automation lesson

A later reboot exposed another distinction: **a working distributed model is not automatically a durable service**.

The model cache, GPU networking and supporting containers survived, but the original Qwen process had been launched interactively and was not automatically recreated.

The environment was then converted toward appliance-style operation:

- persistent named TensorRT containers;
- deterministic runtime configuration stored outside the containers;
- distributed SSH/MPI preflight before model launch;
- protected local credentials rather than interactive token entry;
- systemd supervision for the long-running distributed Qwen service;
- conservative restart-on-failure behavior;
- simple status/start/stop/repair commands.

After the transition, the complete stack reported healthy:

```text
Model service            OK
Worker node              OK
Qwen3-235B               OK
LiteLLM                  OK
Open WebUI               OK
Prometheus               OK
Grafana                  OK
```

The remaining durability proof is a full controlled two-node reboot/power-cycle with no manual recovery commands.

## Observability

```text
GPU exporter (node 1) --+
                         |
GPU exporter (node 2) --+--> Prometheus --> Grafana
                         |
LiteLLM metrics ---------+
```

Monitoring configuration is moving toward file-based provisioning so dashboards and data sources are reproducible rather than dependent on manual clicking.

## Portable AI memory

The experiment now also includes a lightweight Memoryfield-inspired memory layer.

The durable source of truth is simply Markdown. Initial memories capture architecture, operating principles, model strategy and lessons learned.

This has several advantages:

- no additional always-on database is required for the initial implementation;
- memories are human-readable and easy to version/back up;
- future models can reuse the same accumulated knowledge;
- semantic indexing can be added later only if the collection becomes large enough to need it.

The project intentionally avoids storing every chat or transient debugging event as memory. The focus is durable decisions, reasoning, procedures, discoveries and research conclusions.

## Operations goal

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

## What comes next

1. Perform a full two-node reboot/power-cycle acceptance test.
2. Keep the browser model picker intentionally limited to Fast + Deep.
3. Connect the Markdown memory layer directly into normal AI use.
4. Move away from infrastructure work and into useful research/status/automation workflows.
5. Add additional models or services only when they solve a concrete capability gap.

## Security

No passwords, SSH private keys, Hugging Face tokens, API keys, gateway keys, database credentials, or other secrets belong in this repository. Any organizational or regulated data should only be used where the environment and workflow have been explicitly approved for that data.

---

This public repository intentionally documents architecture, milestones, lessons learned and benchmark results without exposing private addressing, accounts, credentials or sensitive topology details.
