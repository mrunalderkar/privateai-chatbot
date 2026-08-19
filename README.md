# PrivateAI Chatbot on Kubernetes

A self-hosted, private AI chatbot running entirely on a local Kubernetes cluster — no data ever leaves the infrastructure. Built to demonstrate hands-on Kubernetes, containerization, networking, and observability skills.

## Why "Private" AI?

Most AI chatbots send your data to a third-party cloud API. This project runs a small open-source LLM (via [Ollama](https://ollama.com)) entirely inside a self-managed Kubernetes cluster, so every prompt and response stays on infrastructure you control — a pattern relevant to any organization with data-privacy or compliance requirements.

## Architecture

```
 Browser
    |
    v
 Nginx Pod (reverse proxy + static frontend)
    |
    v
 Ollama Service (ClusterIP)
    |
    v
 Ollama Pod (LLM inference)
    |
    v
 PersistentVolumeClaim (model storage)
```

- **Frontend**: A lightweight HTML/JS chat page, served by Nginx and mounted in via a ConfigMap
- **Backend**: [Ollama](https://ollama.com) running `qwen2.5:0.5b` (small enough to run on CPU with limited resources)
- **Cluster**: [k3d](https://k3d.io) (k3s inside Docker) running inside an Ubuntu VM on VirtualBox — chosen over Minikube for its lower resource footprint
- **Storage**: A PVC ensures the downloaded model persists across pod restarts, avoiding re-downloads

## Features

- ✅ **Deployments** with defined resource requests/limits for both Ollama and Nginx
- ✅ **PersistentVolumeClaim** for model data persistence
- ✅ **Liveness & readiness probes** on both pods — Kubernetes actively verifies each app is responding, not just that the process exists
- ✅ **Continuous monitoring** via a lightweight [VictoriaMetrics](https://victoriametrics.com) + vmagent stack (a resource-efficient alternative to Prometheus), visualized through VictoriaMetrics' built-in `vmui` dashboard — no Grafana required, keeping the footprint small enough for a 2GB-constrained VM
- 🚧 NetworkPolicy (restricting Nginx→Ollama traffic) — in progress
- 🚧 HorizontalPodAutoscaler — in progress
- 🚧 Ingress (clean URL routing) — in progress
- 🚧 GitHub Actions CI (manifest validation) — planned

## Monitoring

Metrics (CPU, memory) for both pods are scraped every 60 seconds and stored in VictoriaMetrics, viewable live at `http://<node-ip>:30428/vmui`. This surfaces real behavior — for example, Ollama's memory usage sitting near idle (~36MB) until a chat request triggers model loading, at which point it spikes to several hundred MB during inference.

## Why k3d instead of Minikube

The development VM has limited RAM (2GB available for the whole stack). k3d runs Kubernetes nodes as lightweight Docker containers rather than a full VM-in-VM setup, which meaningfully reduces the baseline resource overhead — a real constraint-driven decision rather than a default choice.

## Setup

1. Ensure Docker and k3d are installed on the host/VM
2. Create the cluster with the required NodePort published:
   ```bash
   k3d cluster create privateai --agents 1 -p "30428:30428@loadbalancer" -p "8080:80@loadbalancer"
   ```
3. Apply the manifests:
   ```bash
   kubectl apply -f ollama-pvc.yaml
   kubectl apply -f ollama-deployement.yaml
   kubectl apply -f ollama-service.yaml
   kubectl apply -f nginx-configMap.yaml
   kubectl apply -f nginx-deployment.yaml
   kubectl apply -f nginx-service.yaml
   kubectl apply -f monitoring.yaml
   ```
4. Access the chatbot via the Nginx service, and the monitoring dashboard at `http://<node-ip>:30428/vmui`

## Challenges & Solutions

- **NodePort not reachable from the VM**: k3d runs cluster nodes as Docker containers, so a Kubernetes NodePort is only reachable if that port was explicitly published to Docker. Fixed with `k3d cluster edit privateai --port-add 30428:30428@loadbalancer` rather than recreating the whole cluster.
- **Resource constraints**: With only 2GB available, a full Prometheus + Grafana stack was too heavy. Switched to VictoriaMetrics (lighter on memory) and dropped Grafana entirely in favor of its built-in `vmui`, saving roughly 150-250Mi of RAM.
- **YAML indentation errors** when adding probes: caught early using `kubectl apply --dry-run=client` before applying changes to live pods, avoiding downtime from misconfigured manifests.

## Tech Stack

Kubernetes (k3d/k3s) · Docker · Ollama · Nginx · VictoriaMetrics · vmagent · Ubuntu · VirtualBox
