# Self-Hosted AI Service

This directory documents the sanitised deployment pattern for a GPU-backed local AI service.

## Architecture

~~~text
Client
  |
  | HTTPS
  v
Nginx reverse proxy
  |
  | HTTP
  v
Open WebUI
  |
  | Ollama API
  v
Ollama
  |
  | CUDA / NVIDIA runtime
  v
GPU
~~~

The service is deployed as a dedicated virtual machine on Proxmox. The GPU is passed through to the VM, while Open WebUI runs as a Docker container and communicates with the Ollama runtime over the private service network.

## Responsibilities

| Component | Responsibility |
|---|---|
| Proxmox VE | VM lifecycle and PCIe device passthrough |
| Linux VM | AI workload isolation |
| NVIDIA runtime | GPU access and acceleration |
| Ollama | Local model serving and inference |
| Open WebUI | Browser-facing AI application |
| Nginx | HTTPS ingress and reverse proxy |

## Design Goals

- Keep inference workloads isolated from the hypervisor.
- Use dedicated GPU acceleration rather than CPU-only inference.
- Keep the inference API on the private service network.
- Expose only the browser-facing application through the reverse proxy.
- Persist Open WebUI application data independently of the container lifecycle.
- Keep runtime secrets and live infrastructure mappings outside Git.

## Sanitised Configuration

Use the example Compose file as a starting point. Replace placeholders locally and keep production configuration outside the repository.

The repository intentionally does not contain:

- real IP addresses
- real service hostnames
- VM identifiers
- GPU PCI identifiers
- model inventories
- authentication data
- API keys or tokens
- production Docker volumes
- live reverse-proxy configuration

## Operational Validation

Validate each layer independently:

1. Confirm the VM can access the passed-through GPU.
2. Confirm the NVIDIA runtime/driver is functional.
3. Confirm Ollama is listening locally.
4. Confirm the Ollama API reports the expected models.
5. Confirm Open WebUI can reach the Ollama API.
6. Confirm the reverse proxy can reach Open WebUI.
7. Validate HTTPS from an approved client.

This layered approach isolates GPU, inference, application, proxy, and TLS failures.
