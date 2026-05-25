# ADR-000: tesseract — Architecture Decisions

**Status:** Active  
**Repo:** [github.com/doctorspazz/tesseract](https://github.com/doctorspazz/tesseract)  
**Last updated:** 2026-05

---

## What this is

`tesseract` is a home lab AI agent infrastructure built as a single-node Kubernetes cluster. It exists to do three things in equal measure:

1. Run real workloads — specifically, AI agent pipelines, LLM gateway infrastructure, and workflow automation centered around a house listing scoring agent.
2. Demonstrate production-grade infrastructure patterns in a way that is honest about scale and constraint.
3. Serve as a reference and blog companion for people who want to build similar things.

The entire stack is GitOps-managed via ArgoCD. All configuration lives in this repository. No manual cluster state. If the box dies, re-running the bootstrap sequence should reproduce the environment.

---

## Hardware reality

| | |
|---|---|
| **Host** | Dell Latitude (repurposed), hostname `tesseract` |
| **CPU** | Intel i7-4600U @ 2.10GHz — 2 cores, 4 threads |
| **RAM** | 16 GB |
| **Disk** | 477 GB SSD (LVM-managed, root volume extended to full disk) |
| **OS** | Ubuntu 24.04.4 LTS |
| **Network** | Static LAN at `192.168.1.127`, `.home` DNS via Pi-hole |
| **GPU node** | Separate machine (`192.168.1.202`) running Ollama with local LLM inference — accessed as an external service, not a cluster node |

This is not a beefy machine. A 2013 dual-core mobile processor is a real constraint, and it shapes every decision here. The goal is not to pretend otherwise — it is to demonstrate that the *patterns* used by large engineering organizations are applicable at this scale, even if the numbers look different.


---

## Architecture decisions

### Cluster

**k3s, single control-plane node.**

Not kubeadm, not Talos, not k0s. k3s is what's actually used in edge and resource-constrained environments, it installs in minutes, and the time not spent debugging kubeadm bootstrap is time spent on the things this project is actually about. Talos is interesting; it is not interesting *here*.

Single node is the right call for the current hardware. The architecture supports adding worker nodes — k3s makes this a single command. That will happen when it becomes useful, not before.

### GitOps

**ArgoCD, app-of-apps pattern.**

ArgoCD is the more widely-known tool (vs. Flux) and has a UI that makes for better blog screenshots. The app-of-apps pattern means one root Application in `bootstrap/` points to everything else. Sync waves enforce deployment order: cert-manager before anything that needs certs, CloudNativePG before anything that needs Postgres.

All Helm values and Kustomize patches live in this repo. Secrets are the only exception — those come from the secrets backend at runtime.

### Ingress

**Traefik v3, carried forward from the pre-k8s stack.**

Switching to ingress-nginx would add no value here. Traefik in k8s mode via IngressRoute CRDs is capable, already understood, and the migration cost buys nothing. If a future post specifically examines ingress behavior under load, that's when it's worth swapping.

`.home` hostnames are resolved by Pi-hole. Wildcard TLS certificate for `*.home` issued by cert-manager against a local CA.

### Secrets

**sops-age + External Secrets Operator.**

Vault is excellent. Vault is also another service to run, backup, and unseal. sops-age encrypts secrets in-repo (age-encrypted, key stored offline), ESO syncs them to Kubernetes Secrets at deploy time. This covers the full secrets lifecycle with significantly less operational surface.

If a post specifically requires Vault for the demonstration, it goes in as an optional component at that point.

### Storage

**Longhorn for persistent volumes.**

Local-path-provisioner would work for a single node, but Longhorn gives volume snapshots, scheduled backups to S3 (MinIO), and a path to multi-node replication. It also adds real operational complexity — which is the point. Longhorn is a legitimate blog topic on its own.

### Data services

**CloudNativePG** for Postgres. Not a StatefulSet manifest, not Bitnami. CNPG is the operator serious shops are converging on — it models clusters as first-class objects, handles point-in-time recovery, and generates blog content naturally when things break.

**Altinity operator** for ClickHouse. Mature, well-documented, used in production by orgs running Langfuse at scale.

**MinIO operator** for object storage. Langfuse uses it for S3-compatible event storage. Longhorn uses it for volume backups. One service, two consumers.

**Redis** via Bitnami Helm chart, standalone. No cluster, no Sentinel. The workload doesn't justify it.

### Observability

**kube-prometheus-stack + Loki + Tempo + OpenTelemetry Collector.**

This is the modern open-source observability stack and the one closest to what large shops run internally. The OTel Collector is the central piece — everything ships to it, and it fans out to Prometheus (metrics), Loki (logs), and Tempo (traces). This means instrumenting a new service is a single endpoint change, not a reconfiguration of four separate pipelines.

**Langfuse** stays as the LLM-specific observability layer. It captures prompt/completion pairs, token counts, latency, and cost in a way that generic tracing tools don't. The intent is to link spans in Tempo back to Langfuse sessions so an n8n workflow run is traceable end-to-end: workflow trigger → LiteLLM gateway call → model response → Langfuse session.

### AI platform

**LiteLLM** as the model gateway. Routes to local Ollama (`192.168.1.202:11434`) and Anthropic API. All agent workflows call LiteLLM — nothing calls a model endpoint directly. This enforces cost tracking, gives routing flexibility, and means model swap-outs don't require touching application code.

LiteLLM's spend tracking feeds a Grafana dashboard (cost per model, per virtual key, per workflow) and a daily n8n digest. This is the FinOps loop.

**n8n** for workflow orchestration. Not Temporal, not Prefect, not Argo Workflows. n8n is the right tool for this workload: HTTP-triggered pipelines, IMAP polling, visual debugging, webhook endpoints. The complexity ceiling is knowable and appropriate.

### Policy

**Kyverno.** Not OPA Gatekeeper. Kyverno policies are Kubernetes-native YAML, easier to write incrementally, and sufficient for what's needed here: enforcing labels, blocking privileged containers, requiring resource limits.

---

## Explicit non-goals

These are things that belong in production environments at scale. They are not here, and that is a deliberate choice, not an oversight.

| Not doing | Why |
|---|---|
| Service mesh (Istio, Linkerd) | Zero east-west traffic benefit on a single node. NetworkPolicies cover the isolation story. |
| Multi-cluster | No second cluster exists. When one does, this doc will be updated. |
| Vault | sops-age + ESO covers the requirement with a fraction of the operational cost. |
| Talos Linux | The OS is not the interesting part of this project. |
| Tekton / Argo Workflows | CI via Forgejo Actions + a runner is sufficient. The pipeline complexity doesn't justify another operator. |
| Crossplane | Not managing cloud resources from this cluster. |
| HA control plane | Single node. If the node goes down, the node goes down. Stateful data is backed up. |
| Full multi-tenancy | Virtual keys in LiteLLM and Langfuse projects approximate the pattern for demonstration purposes. |

---

## Repository structure

```
tesseract/
├── bootstrap/              # k3s install notes + ArgoCD bootstrap + root app
├── platform/               # cluster-wide services
│   ├── traefik/
│   ├── cert-manager/
│   ├── external-secrets/
│   ├── kyverno/
│   └── longhorn/
├── observability/
│   ├── kube-prometheus-stack/
│   ├── loki/
│   ├── tempo/
│   ├── otel-collector/
│   └── grafana-dashboards/
├── data/
│   ├── cnpg/               # CloudNativePG cluster
│   ├── redis/
│   ├── clickhouse/
│   └── minio/
├── ai-platform/
│   ├── litellm/
│   ├── langfuse/
│   └── n8n/
├── apps/
│   └── house-scorer/
├── tests/                  # agent test harness
└── docs/                   # ADRs, runbooks, blog drafts
```

Each directory under `platform/`, `data/`, `observability/`, and `ai-platform/` is an ArgoCD Application. The root Application in `bootstrap/` is the entry point.

---

## What this was before

The stack running prior to this rebuild was Docker Compose — a working system with Traefik, n8n, LiteLLM, Langfuse, Postgres, Redis, ClickHouse, MinIO, Forgejo, and a full observability stack. That system is documented in the `docs/legacy-compose/` directory. The migration from Compose to Kubernetes is itself a thread in the blog series — documented as it happened, including what broke.

---

## Day 0 prerequisite

The default Ubuntu LVM install only allocates 100 GB to the root logical volume despite the full disk being available. Before building anything, extend it:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Verify with `df -h /`.

---

## Related posts

*Links added as posts publish.*

- [ ] Post 01 — Tesseract: the lay of the land (platform overview, what's running, how it fits together)
- [ ] Post 02 — OpenTelemetry across an AI agent pipeline
- [ ] Post 03 — LLM FinOps: cost tracking from LiteLLM to Grafana
- [ ] Post 04 — A test harness for agentic n8n workflows
- [ ] Post 05 — TBD based on what breaks interestingly
