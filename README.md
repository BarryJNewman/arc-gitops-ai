# Arc GitOps

GitOps manifests for Azure Arc-enabled Kubernetes clusters, managed by [Flux v2](https://fluxcd.io/) via `az k8s-configuration flux create`.

## Structure

```
├── infrastructure/          # Cluster-level infra (namespaces, RBAC, etc.)
│   └── kustomization.yaml
├── apps/
│   ├── kustomization.yaml   # Root: includes podinfo, ollama, agent-zero
│   ├── podinfo/             # Smoke-test / health-check app
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── kustomization.yaml
│   ├── ollama/              # GPU-accelerated LLM serving (NVIDIA)
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── kustomization.yaml
│   ├── ollama-rocm/         # GPU-accelerated LLM serving (AMD ROCm)
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── kustomization.yaml
│   └── agent-zero/          # AI agent framework (Agent Zero)
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       └── kustomization.yaml
```

---

## Applications

### Podinfo

Lightweight Go microservice used as a smoke test to verify cluster health, ingress, and GitOps reconciliation.

| Property | Value |
|----------|-------|
| Image | `ghcr.io/stefanprodan/podinfo:6.7.1` |
| Port | 9898 (HTTP), 9999 (gRPC) |
| Ingress | `http://podinfo.local` |
| Resources | 50m–200m CPU, 64–128Mi RAM |

### Ollama

GPU-accelerated LLM inference server. Serves models via a REST API with an OpenAI-compatible endpoint at `/v1`. Two variants are provided — use the one matching your GPU hardware.

#### NVIDIA variant (`apps/ollama/`)

| Property | Value |
|----------|-------|
| Image | `ollama/ollama:latest` |
| Port | 11434 |
| Ingress | `http://ollama.local` |
| GPU | 1x NVIDIA GPU via `runtimeClassName: nvidia` + `nvidia.com/gpu` |
| Storage | hostPath `/var/lib/ollama` → `/root/.ollama` in container |
| Resources | 1–1 CPU, 4–8Gi RAM, 1x `nvidia.com/gpu` |

#### AMD ROCm variant (`apps/ollama-rocm/`)

| Property | Value |
|----------|-------|
| Image | `ollama/ollama:rocm` |
| Port | 11434 |
| Ingress | `http://ollama.local` |
| GPU | 1x AMD GPU via `amd.com/gpu` + `/dev/kfd` + `/dev/dri` device mounts |
| Storage | hostPath `/var/lib/ollama` → `/root/.ollama` in container |
| Resources | 1–1 CPU, 4–8Gi RAM, 1x `amd.com/gpu` |

Key differences: ROCm uses `ollama/ollama:rocm` image, `amd.com/gpu` resource (no `runtimeClassName`), and mounts `/dev/kfd` + `/dev/dri` for GPU access.

**Switching variants:** Edit `apps/kustomization.yaml` to include either `ollama` (NVIDIA) or `ollama-rocm` (AMD):
```yaml
resources:
  - podinfo
  - ollama-rocm   # ← swap 'ollama' for 'ollama-rocm' on AMD GPU nodes
  - agent-zero
```

**First-deploy model pull** (required once per node):
```bash
kubectl exec -n ollama deployment/ollama -- ollama pull qwen2.5:1.5b
```

### Agent Zero

AI agent framework with a web UI. Configured to use Ollama as its LLM backend via the `ollama` provider.

| Property | Value |
|----------|-------|
| Image | `agent0ai/agent-zero:latest` |
| Port | 80 (Web UI) |
| Ingress | `http://agent-zero.local` |
| LLM Backend | Ollama at `http://ollama.ollama.svc.cluster.local:11434` |
| Model | `qwen2.5:1.5b` (chat + utility) |
| Resources | 500m–2 CPU, 512Mi–2Gi RAM |

Agent Zero settings are injected via `A0_SET_*` environment variables. The `OLLAMA_BASE_URL` env var tells Agent Zero where to reach Ollama's API.

---

## Architecture Nuances

### Why Ollama instead of vLLM

vLLM v0.15.1 cannot run on GPUs with less than ~10GB VRAM. Even with the smallest model (Qwen2.5-1.5B-Instruct at 2.89 GiB FP16), vLLM's sampler warmup phase allocates 256 dummy requests and OOMs on a 6GB GPU. The `--max-num-seqs` flag does not reduce the warmup allocation count. Additionally:

- **Flash Attention 2** requires compute capability >= 8.0 (GTX 1660 is 7.5). `--enforce-eager` disables it but doesn't help with OOM.
- **CUDA graph compilation** must also be disabled on CC 7.5 via `--enforce-eager`.

Ollama handles all of this transparently — it loads models on demand, uses efficient quantized formats, and works on any CUDA-capable GPU.

### Kubernetes Service Naming

Never name a Kubernetes Service the same as an application that uses environment variables with that prefix. Kubernetes auto-injects `<SERVICE_NAME>_PORT` env vars into all pods in the same namespace. A service named `vllm` injects `VLLM_PORT=tcp://...` which collides with vLLM's internal config expecting `VLLM_PORT` to be a port number. This was solved by renaming the service to `vllm-server`, and is not an issue with Ollama since each app is in its own namespace.

### RKE2 Ingress Controller

RKE2's built-in nginx ingress controller does **not** create an `IngressClass` resource. It runs with `--watch-ingress-without-class=true`, so:

- **Do NOT** set `spec.ingressClassName: nginx` — it causes "no object matching key 'nginx'" errors.
- **Do NOT** use the `nginx.ingress.kubernetes.io/rewrite-target` annotation unless you need path rewriting.
- Simply omit `ingressClassName` and the controller picks up all Ingress resources automatically.

### Flux Kustomization Health Checks

Flux health checks can get stuck if a deployment is deleted while a health check is in progress. The kustomization enters a `Reconciliation in progress` loop. To recover:

```bash
# Suspend and resume to break the health check loop
kubectl patch kustomization <name> -n cluster-config --type=merge \
  -p '{"spec":{"suspend":true}}'
sleep 5
kubectl patch kustomization <name> -n cluster-config --type=merge \
  -p '{"spec":{"suspend":false}}'
```

If that doesn't work, delete and recreate the Flux configuration via `az k8s-configuration flux delete` / `create`.

### GPU Scheduling

- Only one pod can claim the GPU at a time (`nvidia.com/gpu: "1"` is a non-shareable resource).
- If a GPU-holding pod is stuck terminating, new GPU pods will remain `Pending`. Force-delete the stuck pod: `kubectl delete pod <name> -n <ns> --force --grace-period=0`.
- The `nvidia` RuntimeClass must exist on the cluster (created by the NVIDIA device plugin setup in `rke2-arc-setup.sh`).

### Ollama Model Persistence

Ollama stores downloaded models in `/root/.ollama` inside the container, mapped to `/var/lib/ollama` on the host via `hostPath`. This means:

- Models survive pod restarts on the same node.
- Models are **lost** if the node is reimaged or the hostPath directory is deleted.
- On first deploy (or after node reimage), you must manually pull the model.

### Startup Probes

Ollama uses a `startupProbe` with `failureThreshold: 30` and `periodSeconds: 5` (150s total) to handle slow model downloads on first boot. Without a startup probe, the liveness probe would kill the container before it finishes initializing.

---

## Security Considerations

### Current State (Development)

This deployment is configured for **local development / lab use**. The following items should be addressed before any production or internet-facing deployment:

### No Authentication on Ingress

All three ingresses (`podinfo.local`, `ollama.local`, `agent-zero.local`) are exposed without authentication. Anyone on the network can:

- **Query the LLM** via `ollama.local/v1/chat/completions`
- **Use Agent Zero's web UI** at `agent-zero.local` (which has shell/code execution capabilities)
- **Pull/push models** to Ollama via its API

**Mitigations:**
- Add nginx basic auth annotations to ingresses
- Use a reverse proxy with OAuth2/OIDC (e.g., oauth2-proxy)
- Restrict ingress to specific source IPs via `nginx.ingress.kubernetes.io/whitelist-source-range`
- Use NetworkPolicies to limit pod-to-pod traffic

### No TLS

All ingress traffic is plain HTTP. Credentials and LLM responses are transmitted in cleartext.

**Mitigations:**
- Add TLS termination at the ingress with cert-manager + Let's Encrypt
- Or use a self-signed CA for `.local` domains

### Agent Zero Has Broad Capabilities

Agent Zero is an autonomous AI agent with the ability to execute shell commands, write files, and browse the web. Running it in a Kubernetes pod means it has access to:

- The container's filesystem
- Network access to all cluster services (including Ollama)
- Any mounted volumes or secrets

**Mitigations:**
- Run Agent Zero with a restrictive `securityContext` (read-only root filesystem, drop all capabilities, non-root user)
- Use Kubernetes NetworkPolicies to restrict Agent Zero's egress to only the Ollama service
- Do not mount service account tokens (`automountServiceAccountToken: false`)
- Consider running Agent Zero in a sandboxed runtime (gVisor, Kata Containers)

### Container Images Use `latest` Tag

Both `ollama/ollama:latest` and `agent0ai/agent-zero:latest` use mutable tags. This means:

- Builds are not reproducible
- A compromised upstream image could be pulled automatically
- Pod restarts may pull a different version

**Mitigations:**
- Pin images to specific digests or version tags (e.g., `ollama/ollama:0.6.2`)
- Use an image admission controller (e.g., Kyverno, OPA Gatekeeper) to enforce digest pinning
- Mirror images to a private registry

### Secrets in Plain Text

The `OLLAMA_BASE_URL` and `A0_SET_*` values are stored as plain-text env vars in the deployment YAML (and in the Git repo). While none are currently sensitive, if API keys are added later they would be exposed.

**Mitigations:**
- Use Kubernetes Secrets for sensitive values
- Use Azure Key Vault with the Secrets Store CSI Driver for production secrets
- Enable Git repository encryption (e.g., SOPS + age) for secrets in GitOps repos

### hostPath Volumes

Ollama uses a `hostPath` volume (`/var/lib/ollama`). This:

- Bypasses Kubernetes storage abstractions
- Could allow container escape if combined with other vulnerabilities
- Is not portable across nodes

**Mitigations:**
- Use PersistentVolumeClaims with a proper StorageClass for production
- If hostPath is required, use `readOnly: true` where possible and restrict with PodSecurityPolicies/Standards

### GitOps Repo Access

The GitOps repo uses an SSH deploy key stored in Azure Key Vault. The key is:

- Generated during `build.sh` (image build time)
- Stored in Key Vault with a custom role that grants `getSecret` + `setSecret`
- Retrieved at first boot by `rke2-arc-setup.sh` and passed to `az k8s-configuration flux create`

**Ensure:**
- The deploy key is read-only on the GitHub repo
- The Key Vault access policy follows least privilege
- The service principal used for Arc onboarding has minimal permissions

---

## Usage

This repo is automatically applied to Arc-enabled clusters during first-boot onboarding via `rke2-arc-setup.sh`. It can also be applied manually:

```bash
az k8s-configuration flux create \
  -g arc-onboard-rg \
  -c <cluster-name> \
  -n cluster-config \
  --namespace cluster-config \
  -t connectedClusters \
  --scope cluster \
  -u ssh://git@github.com/BarryJNewman/arc-gitops \
  --branch main \
  --ssh-private-key-file <path-to-deploy-key> \
  --kustomization name=infra path=./infrastructure prune=true \
  --kustomization name=apps path=./apps prune=true dependsOn=\["infra"\]
```

### Post-Deploy Steps

1. **Pull the Ollama model** (required once per node):
   ```bash
   kubectl exec -n ollama deployment/ollama -- ollama pull qwen2.5:1.5b
   ```

2. **Add local DNS entries** (on your workstation):
   ```bash
   echo "192.168.0.171 podinfo.local ollama.local agent-zero.local" | sudo tee -a /etc/hosts
   ```

3. **Verify services**:
   ```bash
   curl http://podinfo.local/           # Podinfo JSON response
   curl http://ollama.local/v1/models   # Lists qwen2.5:1.5b
   curl http://agent-zero.local/        # Agent Zero web UI
   ```

### Changing the LLM Model

To switch models, pull the new model and update Agent Zero's env vars:

```bash
# Pull a different model
kubectl exec -n ollama deployment/ollama -- ollama pull <model-name>

# Update agent-zero deployment.yaml:
#   A0_SET_chat_model_name: "<model-name>"
#   A0_SET_utility_model_name: "<model-name>"
# Then git commit + push, or patch directly:
kubectl set env deployment/agent-zero -n agent-zero \
  A0_SET_chat_model_name=<model-name> \
  A0_SET_utility_model_name=<model-name>
```

### GPU Hardware Compatibility

| GPU | VRAM | Ollama | vLLM | Notes |
|-----|------|--------|------|-------|
| GTX 1660 (CC 7.5) | 6 GB | Yes | No | vLLM OOMs during sampler warmup |
| RTX 3080 (CC 8.6) | 10 GB | Yes | Yes | Tested with Llama 3.1 8B via Ollama |
| Any CC >= 7.0 | >= 4 GB | Yes | Varies | Ollama handles quantization automatically |
