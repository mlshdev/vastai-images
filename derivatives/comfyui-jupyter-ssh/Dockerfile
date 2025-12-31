# Custom Dockerfile for vast.ai with JupyterLab, ComfyUI, and SSH access
# Built from scratch using nvidia/cuda:13.0.2-cudnn-runtime-ubuntu24.04
# Target platform: linux/amd64 only
#
# This image provides:
# - JupyterLab with ipykernel support
# - ComfyUI with Manager and essential nodes
# - SSH server for remote access
# - Vast.ai platform compatibility (Caddy for HTTPS, Cloudflared tunnels)
# - TensorRT and deep learning dependencies via uv
# - AWS CLI and Hetzner CLI for cloud management
#
# Build with: docker buildx build --platform linux/amd64 -t comfyui-jupyter-ssh .

### Stage 1: Copy uv from distroless image (as per requirements)
FROM --platform=linux/amd64 ghcr.io/astral-sh/uv:latest AS uv_builder

### Stage 2: Build Caddy with single port TLS redirect for vast.ai HTTPS support
FROM --platform=linux/amd64 docker.io/library/golang:trixie AS caddy_builder

# Install xcaddy for the current architecture
RUN go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest

# Build Caddy with TLS redirect support
ENV CGO_ENABLED=0
RUN GOOS=linux GOARCH=amd64 xcaddy build \
    --with github.com/caddyserver/caddy/v2=github.com/ai-dock/caddy/v2@httpredirect \
    --with github.com/caddyserver/replace-response

### Stage 3: Main Build
FROM --platform=linux/amd64 docker.io/nvidia/cuda:13.0.2-cudnn-runtime-ubuntu24.04 AS main_build

# Maintainer details
LABEL org.opencontainers.image.source="https://github.com/mlshdev/vastai-images"
LABEL org.opencontainers.image.description="Custom image with JupyterLab, ComfyUI, and SSH for Vast.ai"
LABEL maintainer="Custom Build"

# Copy uv binaries from distroless image (multi-stage build requirement)
COPY --from=uv_builder /uv /usr/local/bin/uv
COPY --from=uv_builder /uvx /usr/local/bin/uvx

# Support pipefail so we don't build broken images
SHELL ["/bin/bash", "-c", "umask 002 && /bin/bash -c \"$@\"", "-"]
# Use umask 002 so 'user' can easily share the root group
RUN sed -i '1i umask 002' /root/.bashrc

# Vast.ai environment variables used for Jupyter & Data sync
ENV DATA_DIRECTORY=/workspace
ENV WORKSPACE=/workspace

# Ubuntu 24.04 requires this for compatibility with vast.ai /.launch script
ENV PIP_BREAK_SYSTEM_PACKAGES=1

# Don't ask questions we cannot answer during the build
ENV DEBIAN_FRONTEND=noninteractive
# Allow immediate output
ENV PYTHONUNBUFFERED=1

# Configure uv environment variables
ENV UV_CACHE_DIR=/.uv/cache
ENV UV_NO_CACHE=1
ENV UV_LINK_MODE=copy
ENV UV_PYTHON_BIN_DIR=/.uv/python_bin
ENV UV_PYTHON_INSTALL_DIR=/.uv/python_install

# Create a useful base environment with commonly used tools
RUN \
    set -euo pipefail && \
    dpkg --add-architecture i386 && \
    apt-get update && \
    apt-get upgrade -y && \
    apt-get install --no-install-recommends -y \
    # Base system utilities
    acl \
    bc \
    ca-certificates \
    gpg-agent \
    software-properties-common \
    locales \
    lsb-release \
    curl \
    wget \
    sudo \
    moreutils \
    nano \
    vim \
    less \
    jq \
    git \
    git-lfs \
    man \
    tzdata \
    # Display
    fonts-dejavu \
    fonts-freefont-ttf \
    fonts-ubuntu \
    ffmpeg \
    libgl1 \
    libglx-mesa0 \
    # System monitoring & debugging
    htop \
    iotop \
    strace \
    libtcmalloc-minimal4 \
    lsof \
    procps \
    psmisc \
    nvtop \
    # Infiniband support (if devices mounted)
    rdma-core \
    libibverbs1 \
    ibverbs-providers \
    libibumad3 \
    librdmacm1 \
    infiniband-diags \
    # Development essentials
    build-essential \
    cmake \
    ninja-build \
    gdb \
    libssl-dev \
    # System Python
    python3-full \
    python3-dev \
    python3-pip \
    # Network utilities
    netcat-traditional \
    net-tools \
    dnsutils \
    iproute2 \
    iputils-ping \
    traceroute \
    # File management
    dos2unix \
    rsync \
    rclone \
    zip \
    unzip \
    xz-utils \
    zstd \
    # Performance analysis
    linux-tools-common \
    cron \
    # Required for cron logging
    rsyslog \
    # SSH server for remote access (required by problem statement)
    openssh-server \
    # OpenCL General
    clinfo \
    pocl-opencl-icd \
    opencl-headers \
    ocl-icd-dev \
    ocl-icd-opencl-dev \
    # Vulkan
    libvulkan1 \
    vulkan-tools && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Configure SSH server for vast.ai key-based authentication
RUN \
    set -euo pipefail && \
    mkdir -p /var/run/sshd && \
    mkdir -p /root/.ssh && \
    chmod 700 /root/.ssh && \
    # Configure sshd for key-based authentication only (vast.ai injects keys)
    sed -i 's/#PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config && \
    sed -i 's/#PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    echo "HostKeyAlgorithms +ssh-rsa" >> /etc/ssh/sshd_config && \
    echo "PubkeyAcceptedKeyTypes +ssh-rsa" >> /etc/ssh/sshd_config

# Add a normal user account - Some applications don't like to run as root
RUN \
    set -euo pipefail && \
    useradd -ms /bin/bash user -u 1001 -g 0 && \
    sed -i '1i umask 002' /home/user/.bashrc && \
    echo "PATH=${PATH}" >> /home/user/.bashrc && \
    echo "user ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/user && \
    chmod 0440 /etc/sudoers.d/user && \
    mkdir -m 700 -p /run/user/1001 && \
    chown 1001:0 /run/user/1001 && \
    mkdir -p /run/dbus && \
    mkdir -p /opt/workspace-internal/

# Setup uv directories
RUN \
    set -euo pipefail && \
    mkdir -p "${UV_CACHE_DIR}" "${UV_PYTHON_BIN_DIR}" "${UV_PYTHON_INSTALL_DIR}"

# Install NVM for node version management
RUN \
    set -euo pipefail && \
    git clone https://github.com/nvm-sh/nvm.git /opt/nvm && \
    (cd /opt/nvm/ && git checkout `git describe --abbrev=0 --tags --match "v[0-9]*" $(git rev-list --tags --max-count=1)`) && \
    . /opt/nvm/nvm.sh && \
    nvm install --lts

# Create portal and instance tools directories
RUN mkdir -p /opt/portal-aio/caddy_manager /opt/portal-aio/tunnel_manager /var/log/portal /opt/instance-tools/bin

# Copy portal-aio web app for vast.ai Instance Portal
COPY ./portal-aio /opt/portal-aio

# Copy Caddy binary from builder (for vast.ai HTTPS support)
COPY --from=caddy_builder /go/caddy /opt/portal-aio/caddy_manager/caddy

# Install cloudflared and setup portal-aio for vast.ai HTTPS and tunnels
RUN \
    set -euo pipefail && \
    chown -R 0:0 /opt/portal-aio && \
    # Create portal venv and install dependencies
    uv venv --seed /opt/portal-aio/venv -p 3.11 && \
    mkdir -m 770 -p /var/log/portal && \
    chown 0:0 /var/log/portal/ && \
    . /opt/portal-aio/venv/bin/activate && \
    uv pip install -r /opt/portal-aio/requirements.txt && \
    deactivate && \
    # Install cloudflared
    mkdir -p /opt/portal-aio/tunnel_manager && \
    wget -O /opt/portal-aio/tunnel_manager/cloudflared https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 && \
    chmod +x /opt/portal-aio/tunnel_manager/cloudflared && \
    # Create symlinks for easy access
    ln -s /opt/portal-aio/caddy_manager/caddy /opt/instance-tools/bin/caddy && \
    ln -s /opt/portal-aio/tunnel_manager/cloudflared /opt/instance-tools/bin/cloudflared

# Create Python virtual environment with uv and install base tools
ARG PYTHON_VERSION=3.12
ENV PYTHON_VERSION=${PYTHON_VERSION}

RUN \
    set -euo pipefail && \
    # Create virtual environment with uv
    uv venv --seed /venv/main -p ${PYTHON_VERSION} && \
    mkdir -p /venv/main/etc/conda/activate.d && \
    echo 'echo -e "\033[32mActivated virtual environment at \033[36m$(realpath $VIRTUAL_ENV)\033[0m"' \
    > /venv/main/etc/conda/activate.d/environment.sh

# Add venv activation script compatible with vast.ai patterns
RUN cat <<'VENV_ACTIVATION_SCRIPT' > /venv/main/bin/activate
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    echo "This script must be sourced: source bin/activate"
    exit 1
fi

# Define deactivate function
deactivate() {
    if [ -n "${_OLD_VIRTUAL_PATH:-}" ]; then
        PATH="${_OLD_VIRTUAL_PATH}"
        export PATH
        unset _OLD_VIRTUAL_PATH
    fi
    if [ -n "${_OLD_VIRTUAL_PYTHONHOME:-}" ]; then
        PYTHONHOME="${_OLD_VIRTUAL_PYTHONHOME}"
        export PYTHONHOME
        unset _OLD_VIRTUAL_PYTHONHOME
    fi
    unset VIRTUAL_ENV
    unset -f deactivate
    return 0
}

# Activate the virtual environment
export VIRTUAL_ENV="/venv/main"
_OLD_VIRTUAL_PATH="$PATH"
export PATH="$VIRTUAL_ENV/bin:$PATH"

echo -e "\033[32mActivated virtual environment at \033[36m$VIRTUAL_ENV\033[0m"
VENV_ACTIVATION_SCRIPT

# Install TensorRT and deep learning dependencies via uv (as per requirements)
RUN \
    set -euo pipefail && \
    . /venv/main/bin/activate && \
    uv pip install \
    wheel \
    # PyTorch ecosystem
    torch \
    torchvision \
    torchaudio \
    # TensorRT for inference optimization (as per requirements)
    tensorrt \
    # ComfyUI and AI dependencies
    numpy \
    scipy \
    pillow \
    transformers \
    accelerate \
    safetensors \
    einops \
    aiohttp \
    pyyaml \
    # JupyterLab and notebook support (as per requirements)
    jupyterlab \
    jupyter \
    ipykernel \
    ipywidgets \
    # Hugging Face CLI
    "huggingface_hub[cli]" \
    # Supervisor for process management
    supervisor \
    # Tensorboard for monitoring
    tensorboard \
    # File transfer
    magic-wormhole && \
    # Install Jupyter kernel
    python -m ipykernel install \
    --name="main" \
    --display-name="Python3 (main venv)" && \
    python -m ipykernel install \
    --name="python3" \
    --display-name="Python3 (ipykernel)" && \
    deactivate

# Install Jupyter kernel for system Python
RUN \
    set -euo pipefail && \
    pip install --no-cache-dir ipykernel && \
    python3 -m ipykernel install \
    --name="system-python" \
    --display-name="Python3 (System)"

# Create supervisor log directory
RUN mkdir -p /var/log/supervisor

# Install AWS CLI (https://github.com/aws/aws-cli)
RUN \
    set -euo pipefail && \
    curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip" && \
    cd /tmp && \
    unzip awscliv2.zip && \
    ./aws/install && \
    rm -rf /tmp/awscliv2.zip /tmp/aws

# Install Hetzner CLI (https://github.com/hetznercloud/cli)
RUN \
    set -euo pipefail && \
    HCLOUD_VERSION="$(curl -fsSL "https://api.github.com/repos/hetznercloud/cli/releases/latest" | jq -r '.tag_name' | sed 's/v//')" && \
    curl -fsSL "https://github.com/hetznercloud/cli/releases/download/v${HCLOUD_VERSION}/hcloud-linux-amd64.tar.gz" -o /tmp/hcloud.tar.gz && \
    tar -xzf /tmp/hcloud.tar.gz -C /usr/local/bin hcloud && \
    chmod +x /usr/local/bin/hcloud && \
    rm -f /tmp/hcloud.tar.gz

# Install ComfyUI (as per requirements)
# ComfyUI is installed at /workspace/ComfyUI for easy access (synced from /opt/workspace-internal)
ARG COMFYUI_REF=master
RUN \
    set -euo pipefail && \
    . /venv/main/bin/activate && \
    # Get ComfyUI
    cd /opt/workspace-internal/ && \
    git clone https://github.com/comfyanonymous/ComfyUI && \
    cd /opt/workspace-internal/ComfyUI && \
    git checkout "${COMFYUI_REF}" && \
    # Install ComfyUI dependencies
    uv pip install --no-cache-dir -r requirements.txt && \
    # Avoid Jupyter directory display bug
    ln -s /opt/workspace-internal/ComfyUI/models/checkpoints /opt/workspace-internal/ComfyUI/models/ckpt && \
    # Install ComfyUI-Manager from Comfy-Org (official)
    cd /opt/workspace-internal/ComfyUI/custom_nodes && \
    git clone https://github.com/Comfy-Org/ComfyUI-Manager && \
    uv pip install -r ComfyUI-Manager/requirements.txt && \
    # Install ComfyUI_essentials
    git clone https://github.com/cubiq/ComfyUI_essentials && \
    uv pip install -r ComfyUI_essentials/requirements.txt && \
    mkdir -p /opt/workspace-internal/ComfyUI/user/default/ComfyUI-Manager/components && \
    deactivate

# Install vast-cli
RUN \
    set -euo pipefail && \
    cd /opt && \
    git clone https://github.com/vast-ai/vast-cli && \
    wget -O /usr/local/share/ca-certificates/jvastai.crt https://console.vast.ai/static/jvastai_root.cer && \
    update-ca-certificates

# Copy configuration files from ROOT directory (supervisor configs, scripts, etc.)
COPY ./ROOT/ /

# Create symlink for easy ComfyUI access (will be synced to /workspace on first boot)
RUN ln -sf /opt/workspace-internal/ComfyUI /ComfyUI

# Set environment variables
ENV PATH=/opt/instance-tools/bin:${PATH}
ENV COMFYUI_ARGS="--disable-auto-launch --port 18188"

ENTRYPOINT ["/opt/instance-tools/bin/entrypoint.sh"]
CMD []
