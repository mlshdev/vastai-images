ARG PYTORCH_BASE=vastai/pytorch:2.5.1-cuda-12.1.1

FROM ${PYTORCH_BASE}

# Maintainer details
LABEL org.opencontainers.image.source="https://github.com/vastai/"
LABEL org.opencontainers.image.description="ComfyUI image suitable for Vast.ai."
LABEL maintainer="Vast.ai Inc <contact@vast.ai>"

# Copy Supervisor configuration and startup scripts
COPY ./ROOT /

# Required or we will not build
ARG COMFYUI_REF

RUN /bin/bash -c 'set -euo pipefail && \
    [[ -n "${COMFYUI_REF}" ]] || { echo "Must specify COMFYUI_REF" && exit 1; } && \
    source /venv/main/bin/activate && \
    torch_version_pre="$(python -c "import torch; print (torch.__version__)")" && \
    uv pip install torch==$PYTORCH_VERSION --index-url "${PYTORCH_INDEX_URL}" && \
    cd /opt/workspace-internal/ && \
    git clone https://github.com/comfyanonymous/ComfyUI && \
    cd /opt/workspace-internal/ComfyUI && \
    git checkout "${COMFYUI_REF}" && \
    uv pip install --no-cache-dir \
        -r requirements.txt && \
    ln -s /opt/workspace-internal/ComfyUI/models/checkpoints /opt/workspace-internal/ComfyUI/models/ckpt && \
    cd /opt/workspace-internal/ComfyUI/custom_nodes && \
    git clone https://github.com/ltdrdata/ComfyUI-Manager && \
    uv pip install -r ComfyUI-Manager/requirements.txt && \
    git clone https://github.com/cubiq/ComfyUI_essentials && \
    uv pip install -r ComfyUI_essentials/requirements.txt && \
    cd /opt/workspace-internal/ComfyUI && \
    LD_PRELOAD=libtcmalloc_minimal.so.4 \
         python main.py \
            --cpu \
            --listen 127.0.0.1 \
            --port 11404 \
            --disable-auto-launch \
            --quick-test-for-ci && \
    mkdir -p /opt/workspace-internal/ComfyUI/user/default/ComfyUI-Manager/components && \
    torch_version_post="$(python -c "import torch; print (torch.__version__)")" && \
    [[ $torch_version_pre = $torch_version_post ]] || { echo "PyTorch version mismatch (wanted ${torch_version_pre} but got ${torch_version_post})"; exit 1; }'

    ENV COMFYUI_ARGS="--disable-auto-launch --port 18188"