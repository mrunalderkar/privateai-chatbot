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
- ✅ **Ingress** (via Traefik, k3d's built-in controller) — chatbot reachable at a clean hostname (`chatbot.local:8080`) instead of raw IP/NodePort
- ✅ **HorizontalPodAutoscaler** on the Nginx Deployment (1-2 replicas, scales on CPU >50%) — demonstrates working autoscaling; not applied to Ollama, since the dev VM's memory (2GB total) has no headroom to run a second LLM replica
- ✅ **NetworkPolicy** restricting traffic so only Nginx can reach Ollama — verified by testing from a non-Nginx pod to confirm it's actually enforced, not just applied
- **CI/CD (GitHub Actions)** was intentionally skipped — this project runs locally for demonstration purposes rather than as a long-running deployment, so an automated pipeline wasn't a priority

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
   kubectl apply -f nginx-ingress.yaml
   kubectl apply -f nginx-hpa.yaml
   ```
4. Add the VM's IP to your local hosts file mapped to `chatbot.local`, then access the chatbot at `http://chatbot.local:8080`. The monitoring dashboard is available at `http://<node-ip>:30428/vmui`

## Challenges & Solutions

- **NodePort not reachable from the VM**: k3d runs cluster nodes as Docker containers, so a Kubernetes NodePort is only reachable if that port was explicitly published to Docker. Fixed with `k3d cluster edit privateai --port-add 30428:30428@loadbalancer` rather than recreating the whole cluster.
- **Resource constraints**: With only 2GB available, a full Prometheus + Grafana stack was too heavy. Switched to VictoriaMetrics (lighter on memory) and dropped Grafana entirely in favor of its built-in `vmui`, saving roughly 150-250Mi of RAM.
- **YAML indentation errors** when adding probes: caught early using `kubectl apply --dry-run=client` before applying changes to live pods, avoiding downtime from misconfigured manifests.
- **Ingress not reachable on port 80**: k3d's load balancer only had port `8080` published to the VM (mapped to Traefik's internal port 80), so `http://chatbot.local` timed out until the port was included explicitly (`http://chatbot.local:8080`). Diagnosed by testing with `curl -H "Host: chatbot.local" http://localhost:8080/` directly on the VM before touching the browser.

## Screenshots

- Chatbot running at `chatbot.local:8080`
- <img width="1910" height="940" alt="Screenshot 2026-08-19 210043" src="https://github.com/user-attachments/assets/9cdebc33-3995-4e40-bacf-2768faf2026c" />

- VictoriaMetrics dashboard showing Ollama's memory spike during inference
- <img width="1899" height="594" alt="Screenshot 2026-08-19 210755" src="https://github.com/user-attachments/assets/b0603f37-fd39-4293-a170-765482c47b04" />
<img width="1917" height="826" alt="Screenshot 2026-08-19 210807" src="https://github.com/user-attachments/assets/36929095-8f4a-4733-b0b7-25d76f6919cc" />
<img width="1898" height="826" alt="Screenshot 2026-08-19 212028" src="https://github.com/user-attachments/assets/d82f4461-82a1-43bd-9ecb-7192c9e516b9" />

- `kubectl get hpa` output showing live autoscaling status
- <img width="1270" height="367" alt="Screenshot 2026-08-19 212129" src="https://github.com/user-attachments/assets/01e7ef6b-7131-4c10-aeea-733469e0f400" />


*(Add images here — drag and drop directly into GitHub's file editor to auto-generate the markdown links.)*

## With More Resources

This project was intentionally built under tight constraints (a 2GB VM) to demonstrate resource-conscious engineering. Given more headroom, natural next steps would be: enabling HPA on Ollama itself, multi-node HA, a GitHub Actions CI pipeline for manifest validation, and TLS via cert-manager on the Ingress.

## Tech Stack

Kubernetes (k3d/k3s) · Docker · Ollama · Nginx · VictoriaMetrics · vmagent · Traefik · Ubuntu · VirtualBox

---

Built by Mrunal Derkar
