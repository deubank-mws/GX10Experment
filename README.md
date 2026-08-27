# GX10 Experiment

A hands-on experiment building a two-node local AI system from two ASUS GX10 / NVIDIA DGX Spark systems.

The goal is to learn what is practical with compact Blackwell systems: high-speed GPU networking, distributed inference, very large open-weight models, local APIs, agents, and observability.

## Current milestone — August 26, 2026

The two-node cluster is operational at the infrastructure layer:

- Two ASUS GX10 / NVIDIA DGX Spark systems, each with an NVIDIA GB10 and 128 GB unified memory
- DGX Spark 7.5.0, NVIDIA driver 580.173.02, CUDA 13.0
- Docker 29.2.1 with working GPU containers
- Two direct ConnectX-7 RoCE links between the systems, each negotiating at 200 Gb/s
- Persistent dedicated cluster networking
- Passwordless inter-node SSH for distributed jobs
- NCCL 2.30.7-1 and OpenMPI 4.1.6
- Successful two-node NCCL `all_gather` validation with both GB10 GPUs and zero validation errors
- TensorRT-LLM 1.3.0rc13 running in multi-node containers
- Container-to-container MPI/SSH validated across both systems
- `nvidia/Qwen3-235B-A22B-FP4` selected for the first large-model experiment
- Complete 27-shard Qwen checkpoint downloaded and verified on both nodes
- First TP=2 TensorRT-LLM model load underway

## Why Qwen3-235B-A22B-FP4?

The experiment is intentionally aimed at a model large enough to make meaningful use of both systems rather than simply running separate small models on each node. The FP4 checkpoint is a practical fit for Blackwell hardware and provides a useful test of distributed inference across the two-node system.

## Networking validation

Both active ConnectX interfaces report RDMA `ACTIVE`, physical link `LINK_UP`, and 200000 Mb/s link speed.

A 16 GiB NCCL `all_gather` test completed successfully across the two GB10 GPUs with zero wrong or out-of-bounds values. This established that the cluster can execute a real distributed GPU collective before attempting large-model inference.

## What comes next

The next milestones are:

1. Complete the first distributed Qwen load with TensorRT-LLM.
2. Validate generation and benchmark tokens/sec.
3. Expose an OpenAI-compatible API endpoint.
4. Add a browser-based chat interface.
5. Add observability for model, GPU, API, and agent usage.
6. Experiment with research and status-oriented agents and tool integrations.
7. Compare additional open-weight models and quantizations on the same hardware.

## Security

No passwords, SSH private keys, Hugging Face tokens, API keys, or other credentials belong in this repository. Any organizational or regulated data should only be used where the environment and workflow have been explicitly approved for that data.

---

This public repository intentionally documents the architecture, milestones, lessons learned, and benchmark results without exposing private configuration or credentials.