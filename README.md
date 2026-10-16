# Gurukul

Self-hosted LLM inference platform on AWS EKS. Open-weight models served through an OpenAI-compatible API on spot GPUs that scale to zero when idle, with a live cost-per-million-tokens dashboard.

> Status: in active development (Sept – Nov 2026). Results below are filled in as each phase is measured.

## What it does

- Serves open-weight LLMs (Qwen2.5-7B-Instruct-AWQ, Qwen2.5-1.5B for dev) with **vLLM** behind an OpenAI-compatible API
- GPU nodes are **g5.xlarge spot** (NVIDIA A10G, 24 GB), launched on demand by **Karpenter**, running **Bottlerocket**
- The whole GPU fleet **scales to zero** with **KEDA** and wakes on the first request
- A **FastAPI + Redis gateway** handles API keys, per-key rate limits, token metering, and holds requests while a GPU node cold-starts
- **Grafana** shows live cost per 1M tokens from real spot prices, vLLM token counters and DCGM GPU metrics, with a break-even line against public API pricing
- Delivered with **Terraform + Helm + Argo CD**, Argo Rollouts canaries gated on p95 time-to-first-token, cosign-signed images enforced by **Kyverno**, and sha256-verified model weights

## Architecture

```
                         Internet
                            |
                 Route53: api.gurukul.<domain>
                            |
                    ALB (TLS via ACM)            <- AWS Load Balancer Controller
                            |
   +------------------------|------------------- EKS cluster (private subnets) ---+
   |                        v                                                     |
   |   [gateway]  FastAPI: auth, rate limit, metering, request holding            |
   |       |  \                                                                   |
   |       |   +--> [redis]  (rate-limit buckets, counters)                       |
   |       v                                                                      |
   |   [llm-qwen]  vLLM, OpenAI-compatible, /metrics   <- Argo Rollouts (canary)  |
   |       ^  initContainer: weights from S3 + sha256 verify  <- KEDA (0..N)      |
   |       |                                                                      |
   |   GPU node (g5.xlarge spot, Bottlerocket, NVMe cache)       <- Karpenter     |
   |                                                                              |
   |   system nodes: argocd, keda, karpenter, kube-prometheus-stack, kyverno,     |
   |   external-secrets, aws-lb-controller, spot-price-exporter, dcgm-exporter    |
   +------------------------------------------------------------------------------+
            |                         |                          |
     S3 (models, KMS, private)   Secrets Manager            ECR (scan on push)
     via S3 gateway endpoint                                GitHub Actions -> OIDC -> ECR
            |
     NAT instance for remaining egress (instead of NAT Gateway, ~90% cheaper)
```

## Tech stack

AWS (EKS, EC2 spot, S3, KMS, ECR, Secrets Manager, Route53, ACM, FIS) · Terraform · Helm · Karpenter · KEDA · vLLM · NVIDIA A10G · Bottlerocket · FastAPI · Redis · Prometheus · Grafana · DCGM exporter · Argo CD · Argo Rollouts · Kyverno · cosign · Trivy · GitHub Actions · k6

## Roadmap

| Phase | Scope | Status |
|---|---|---|
| 0 | Foundations: account, GPU quotas, tooling, Terraform bootstrap (state, ECR, GitHub OIDC) | |
| 1 | vLLM serving on a single spot GPU, first benchmark | |
| 2 | VPC, EKS, Karpenter, scale-to-zero proof | |
| 3 | Model weights in S3, hash-verified, served by vLLM on Karpenter GPU nodes | |
| 4 | API gateway: keys, rate limits, metering, TLS ingress | |
| 5 | KEDA scale-to-zero, spot interruption handling | |
| 6 | Prometheus, Grafana, DCGM, live cost per 1M tokens | |
| 7 | Argo CD, Argo Rollouts canaries, signed images, Kyverno | |
| 8 | Security pass: threat model, least privilege, NetworkPolicies, CIS benchmark | |
| 9 | Load and chaos testing (k6, AWS FIS), results | |

## Results

Measured numbers are added here (and in `docs/RESULTS.md`) as each phase completes: tokens/s per A10G, TTFT p50/p95, scale-from-zero time, cost per 1M tokens (GPU-only and fully loaded), and recovery from spot interruptions.

## Cost discipline

Nothing runs overnight. `make down` scales GPU workloads to zero and destroys the cluster after every session, `make verify-empty` checks nothing billable is left behind, and an AWS Budget plus a Karpenter GPU limit cap the spend.

## Author

Chirag Goyal · [LinkedIn](https://www.linkedin.com/in/chirag-goyal-cg640) · AWS Certified Solutions Architect – Associate
