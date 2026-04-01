
# LLM Inference & Evaluation Platform

> A production-grade MLOps platform for serving, evaluating, and optimizing Large Language Models at scale — featuring intelligent multi-model routing, automated quality evaluation, semantic caching, and real-time observability.

[![CI/CD](https://github.com/suchethg/LLM-Inference-Evaluation-Platform/actions/workflows/deploy.yml/badge.svg)](https://github.com/suchethg/LLM-Inference-Evaluation-Platform/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![Kubernetes](https://img.shields.io/badge/kubernetes-1.29+-326CE5.svg)](https://kubernetes.io/)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Key Features](#key-features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Benchmarks](#benchmarks)
- [Roadmap](#roadmap)

---

## Overview

This platform solves the core challenges of running LLMs in production:

- **Cost**: Smart routing and semantic caching reduce inference costs by ~55%
- **Quality**: Automated evaluation pipelines catch regressions before users do
- **Reliability**: Multi-model gateway with fallback chains ensures 99.9% uptime
- **Observability**: Full token-level tracing and output scoring on every request

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    GitHub Actions CI/CD                       │
│         Lint → Eval Gate → Canary Deploy → Promote           │
└───────────────────────────┬──────────────────────────────────┘
                            │
              ┌─────────────▼──────────────┐
              │       Model Registry        │
              │   MLflow + S3 (versioned    │
              │   models, prompts, evals)   │
              └─────────────┬──────────────┘
                            │
         ┌──────────────────▼───────────────────┐
         │          Inference Gateway             │
         │        Ray Serve on EKS               │
         │  • Latency-based model selection      │
         │  • A/B traffic splitting              │
         │  • OSS → API fallback chain           │
         │  • Semantic cache lookup              │
         └───────┬──────────────────┬────────────┘
                 │                  │
    ┌────────────▼──┐    ┌──────────▼──────────┐
    │  vLLM Cluster │    │  OpenAI / Anthropic  │
    │  Llama 3,     │    │  API Fallback        │
    │  Mistral      │    └──────────────────────┘
    └────────────┬──┘
                 │
    ┌────────────▼──────────────────────────────┐
    │           Async Eval Pipeline              │
    │  • LLM-as-judge (Ragas)                   │
    │  • Deterministic checks (schema, format)   │
    │  • Output drift detection                  │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │            Observability Stack             │
    │  Prometheus + Grafana + OpenTelemetry      │
    │  • P99 latency per model                  │
    │  • Token cost per team / feature           │
    │  • Quality score trends & drift alerts     │
    └───────────────────────────────────────────┘
```

---

## Key Features

### 🔀 Intelligent Multi-Model Routing
- Routes requests dynamically based on latency SLOs, task complexity, and cost budget
- A/B traffic splitting for prompt and model experiments
- Automatic fallback: OSS failure → API model with zero user impact

### 📦 Prompt & Adapter Version Control
- Prompts versioned in MLflow alongside LoRA adapters
- Canary deployments for prompt changes with automated rollback on quality regression
- Every response traceable to its exact prompt version and model checkpoint

### 🧪 Async LLM Evaluation Pipeline
- Every response sampled asynchronously into an eval pipeline
- LLM-as-judge scoring: faithfulness, relevance, toxicity, groundedness (Ragas)
- Deterministic checks: JSON schema validation, format compliance, regex assertions
- Drift alerts fire when rolling quality scores drop below configurable thresholds

### 💰 Cost & Token Optimization
- Semantic caching with Redis (~42% average cache hit rate)
- Token budget enforcement per use-case and per team
- Real-time cost attribution dashboard

### 🚀 Production CI/CD
- Pipeline: `lint prompt → run eval suite → canary deploy → promote`
- Deployment blocked automatically if eval score drops >5% vs baseline
- ArgoCD for GitOps-style rollouts on EKS

---

## Tech Stack

| Layer | Technology |
|---|---|
| Inference Engine | vLLM 0.4+, Ray Serve |
| Orchestration | EKS + Karpenter (GPU autoscaling) |
| Model Registry | MLflow on S3 |
| Feature Store | Redis + DynamoDB |
| Evaluation | Ragas, custom eval harness |
| Semantic Cache | Redis + LangChain SemanticCache |
| Observability | Prometheus, Grafana, OpenTelemetry |
| CI/CD | GitHub Actions + ArgoCD |
| Infrastructure | Terraform |
| Models | Llama 3 8B/70B, Mistral 7B, GPT-4o (fallback) |

---

## Project Structure

```
LLM-Inference-Evaluation-Platform/
├── infra/                    # Terraform (EKS, VPC, Redis, S3)
├── inference/
│   ├── gateway/              # Ray Serve router, fallback, cache
│   ├── vllm/                 # vLLM serving configs
│   └── k8s/                  # Kubernetes manifests
├── eval/
│   ├── pipeline.py           # Async eval orchestration
│   ├── judges/               # LLM-as-judge + deterministic checks
│   └── drift_detector.py
├── registry/                 # Prompt + adapter versioning, canary logic
├── observability/            # OTel config, Prometheus rules, Grafana dashboards
├── ci/                       # GitHub Actions workflows
├── tests/                    # Unit, integration, load tests
├── docker/                   # Dockerfiles + docker-compose for local dev
└── docs/                     # Architecture, runbooks, ADRs
```

---

## Getting Started

### Local Development

```bash
git clone https://github.com/suchethg/LLM-Inference-Evaluation-Platform.git
cd LLM-Inference-Evaluation-Platform

docker compose -f docker/docker-compose.yml up -d

curl http://localhost:8000/health

curl -X POST http://localhost:8000/v1/chat \
  -H "Content-Type: application/json" \
  -d '{"model": "auto", "messages": [{"role": "user", "content": "Hello"}]}'
```

### Production (EKS)

```bash
cd infra/
terraform init && terraform apply

aws eks update-kubeconfig --name llm-platform --region us-east-1

kubectl apply -f inference/k8s/
kubectl apply -f eval/k8s/
```

### Environment Variables

```bash
AWS_REGION=us-east-1
OPENAI_API_KEY=<your-key>
ANTHROPIC_API_KEY=<your-key>
REDIS_URL=redis://<elasticache-endpoint>:6379
MLFLOW_TRACKING_URI=http://<mlflow-server>:5000
EVAL_SAMPLE_RATE=0.1
DRIFT_ALERT_THRESHOLD=0.05
```

---

## Benchmarks

Measured on `g5.2xlarge` (Llama 3 8B, 4-bit quantized), 50 concurrent users:

| Metric | Value |
|---|---|
| P50 Latency (TTFT) | 82ms |
| P99 Latency (TTFT) | 198ms |
| Throughput | ~1,200 tokens/sec |
| Cache Hit Rate | ~42% |
| Cost Reduction vs GPT-4o only | ~55% |
| Eval Pipeline Lag (async) | < 3 seconds |

---

## Roadmap

- [x] Multi-model gateway with fallback chains
- [x] Semantic caching
- [x] Async LLM-as-judge eval pipeline
- [x] Prompt versioning with MLflow
- [x] Canary deployments via ArgoCD
- [x] Grafana observability dashboards
- [ ] LoRA fine-tuning pipeline with eval gate
- [ ] Online learning: eval scores → training data feedback loop
- [ ] Multi-region active-active deployment
- [ ] Guardrails integration at gateway layer

---

## License

MIT — see [LICENSE](LICENSE) for details.
```