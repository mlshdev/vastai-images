# Custom Docker Image: JupyterLab + ComfyUI + SSH for Vast.ai

This custom Docker image provides a complete AI development environment for [Vast.ai](https://vast.ai) GPU cloud with:

- **JupyterLab**: Interactive notebook development environment
- **ComfyUI**: Node-based Stable Diffusion interface with Manager and essential nodes
- **SSH Access**: Secure key-based SSH access for remote development
- **Full Vast.ai Compatibility**: Platform-specific tools for HTTPS, tunnels, and web UI

## Base Image

Built from scratch using `nvidia/cuda:13.0.2-cudnn-runtime-ubuntu24.04` with:
- CUDA 13.0.2 with cuDNN runtime
- Ubuntu 24.04 LTS
- Python 3.12 virtual environment

## Key Features

### Package Management
- **uv**: Fast Python package manager (installed via multi-stage build from distroless image)
- Pre-configured virtual environment at `/venv/main`

### Deep Learning Stack
- PyTorch with CUDA support
- TensorRT for inference optimization
- Transformers, Accelerate, and other AI libraries

### Web Services
- **Caddy**: Reverse proxy with automatic TLS (built with TLS redirect support)
- **Cloudflared**: Cloudflare tunnel support for secure external access

### Development Tools
- JupyterLab with ipykernel support
- TensorBoard for training visualization
- Hugging Face CLI for model management
- Git LFS for large file handling

## Building the Image

```bash
# Build for amd64 (standard GPU instances)
docker buildx build --platform linux/amd64 -t your-registry/comfyui-jupyter-ssh:latest .

# Build with specific ComfyUI version
docker buildx build --platform linux/amd64 \
    --build-arg COMFYUI_REF=v0.3.0 \
    -t your-registry/comfyui-jupyter-ssh:v0.3.0 .

# Build with custom Python version
docker buildx build --platform linux/amd64 \
    --build-arg PYTHON_VERSION=3.11 \
    -t your-registry/comfyui-jupyter-ssh:py311 .
```

## Build Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `PYTHON_VERSION` | `3.12` | Python version for the main virtual environment |
| `COMFYUI_REF` | `master` | ComfyUI git ref (branch, tag, or commit) |
| `TARGETARCH` | Auto-detected | Target architecture (amd64, arm64) |

## Using on Vast.ai

### Creating a Template

1. Push the built image to a container registry (Docker Hub, GHCR, etc.)
2. In Vast.ai, create a new template with your image
3. Configure port mappings:
   - Port 8080: JupyterLab (via Caddy proxy)
   - Port 8188: ComfyUI (internal port 18188)
   - Port 22: SSH access

### Environment Variables

| Variable | Description |
|----------|-------------|
| `WORKSPACE` | Working directory (default: `/workspace`) |
| `COMFYUI_ARGS` | Additional ComfyUI arguments |
| `PROVISIONING_SCRIPT` | URL to custom provisioning script |
| `ENABLE_HTTPS` | Enable HTTPS via Caddy (`true`/`false`) |
| `WEB_PASSWORD` | Web interface password |

### SSH Access

SSH access uses key-based authentication only. Configure your SSH key in the Vast.ai dashboard:

```bash
ssh root@INSTANCE_IP -p SSH_PORT
```

For SSH tunneling to services:

```bash
# JupyterLab via SSH tunnel
ssh root@INSTANCE_IP -p SSH_PORT -L 8888:localhost:18080

# ComfyUI via SSH tunnel
ssh root@INSTANCE_IP -p SSH_PORT -L 8188:localhost:18188
```

## Directory Structure

```
/workspace/                 # Main workspace (mounted volume)
├── ComfyUI/               # ComfyUI installation
│   ├── models/            # Model files
│   ├── custom_nodes/      # Custom nodes (Manager, Essentials)
│   └── output/            # Generated images
/venv/main/                # Main Python virtual environment
/opt/portal-aio/           # Vast.ai portal web app
/opt/instance-tools/bin/   # Instance management tools
```

## Pre-installed Custom Nodes

- **ComfyUI-Manager**: Node management and installation
- **ComfyUI_essentials**: Essential utility nodes

## Services (Supervisor)

The following services are managed by Supervisor:

| Service | Port | Description |
|---------|------|-------------|
| caddy | Various | HTTPS reverse proxy |
| jupyter | 18080 | JupyterLab server |
| comfyui | 18188 | ComfyUI server |
| tunnel_manager | 11112 | Cloudflare tunnel manager |
| instance_portal | 1111 | Vast.ai instance portal |

## Customization

### Adding Models

Models can be added to `/workspace/ComfyUI/models/` subdirectories:
- `checkpoints/` - Stable Diffusion models
- `loras/` - LoRA models
- `controlnet/` - ControlNet models
- `vae/` - VAE models

### Adding Custom Nodes

1. SSH into the instance
2. Navigate to `/workspace/ComfyUI/custom_nodes/`
3. Clone the custom node repository
4. Restart the ComfyUI service: `supervisorctl restart comfyui`

### Provisioning Script

Use the `PROVISIONING_SCRIPT` environment variable to run a custom setup script on first boot:

```bash
export PROVISIONING_SCRIPT=https://example.com/my-setup.sh
```

## License

This image is built for use with Vast.ai. See individual component licenses:
- NVIDIA CUDA: NVIDIA proprietary license
- ComfyUI: GPL-3.0
- JupyterLab: BSD-3-Clause
- PyTorch: BSD-3-Clause
