ARG PYTORCH_BASE=vastai/pytorch:2.5.1-cuda-12.1.1

FROM ${PYTORCH_BASE}

# Maintainer details
LABEL org.opencontainers.image.source="https://github.com/vastai/"
LABEL org.opencontainers.image.description="Stable Diffusion A1111 image suitable for Vast.ai."
LABEL maintainer="Vast.ai Inc <contact@vast.ai>"

# Copy Supervisor configuration and startup scripts
COPY ./ROOT /

# Required or we will not build
ARG A1111_REF

RUN /bin/bash -c '[[ -n "${A1111_REF}" ]] || { echo "Must specify A1111_REF" && exit 1; } && \
    source /venv/main/bin/activate && \
    torch_version_pre="$(python -c "import torch; print (torch.__version__)")" && \
    pip install xformers torch==$PYTORCH_VERSION --index-url "${PYTORCH_INDEX_URL}" && \
    pip install onnxruntime-gpu && \
    cd /opt/workspace-internal/ && \
    git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui && \
    cd /opt/workspace-internal/stable-diffusion-webui && \
    git checkout "${A1111_REF}" && \
    pip install --no-cache-dir \
        -r requirements_versions.txt && \
    cd /opt/workspace-internal/stable-diffusion-webui && \
    LD_PRELOAD=libtcmalloc_minimal.so.4 \
        python launch.py \
            --use-cpu all \
            --skip-torch-cuda-test \
            --skip-python-version-check \
            --no-download-sd-model \
            --do-not-download-clip \
            --no-half \
            --port 11404 \
            --exit && \
    torch_version_post="$(python -c "import torch; print (torch.__version__)")" && \
    [[ $torch_version_pre = $torch_version_post ]] || { echo "PyTorch version mismatch (wanted ${torch_version_pre} but got ${torch_version_post})"; exit 1; }'
