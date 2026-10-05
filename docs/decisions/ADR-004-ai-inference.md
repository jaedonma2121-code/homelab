# ADR-004: GPU-Backed Local AI Inference

- **Status:** Accepted
- **Decision:** Run the AI workload in a dedicated Proxmox VM with NVIDIA GPU passthrough, using Ollama as the inference runtime and Open WebUI as the browser-facing application.

## Context

The lab needed local AI inference without depending on an external hosted API. The workload also benefits substantially from GPU acceleration, while the inference service should remain isolated from the hypervisor and other applications.

## Decision

Use a layered deployment:

~~~text
Proxmox VE
    |
    | GPU passthrough
    v
Dedicated AI VM
    |
    +--> Open WebUI
    |       |
    |       v
    |     Ollama
    |       |
    |       v
    |      GPU
    |
    +--> persistent application data
~~~

Nginx remains the client-facing ingress layer and terminates TLS before forwarding requests to Open WebUI.

## Rationale

- **Isolation:** AI workloads are contained within a dedicated VM.
- **Performance:** GPU passthrough enables hardware-accelerated inference.
- **Separation of concerns:** Open WebUI handles the application experience while Ollama handles inference.
- **Security:** The inference API remains internal and is not the normal client-facing endpoint.
- **Operational simplicity:** The architecture uses a small number of clearly defined components.

## Consequences

### Positive

- Local inference without an external model API.
- Clear separation between UI, inference runtime, and hardware.
- GPU resources can be monitored independently.
- Existing Nginx and private-network architecture can be reused.
- The workload is straightforward to expand with additional local models.

### Negative

- GPU passthrough adds VM and host configuration dependencies.
- GPU driver/runtime compatibility becomes an operational concern.
- AI workloads can consume significant memory, GPU memory, power, and storage.
- The AI VM becomes a separate infrastructure lifecycle to maintain.

## Troubleshooting Boundary

Failures should be isolated in this order:

~~~text
GPU
  ↓
NVIDIA runtime
  ↓
Ollama
  ↓
Open WebUI
  ↓
Nginx
  ↓
TLS / client
~~~

Each layer should be validated independently before changing another layer.

## Public Documentation Boundary

The repository intentionally excludes:

- real IP addresses
- live hostnames
- VM identifiers
- GPU PCI identifiers
- authentication secrets
- API keys
- production Docker configuration
- model inventory tied to the live environment
