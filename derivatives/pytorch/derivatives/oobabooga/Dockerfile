ARG PYTORCH_BASE

FROM ${PYTORCH_BASE}

# Maintainer details
LABEL org.opencontainers.image.source="https://github.com/vastai/"
LABEL org.opencontainers.image.description="Oobabooga Text Generation WebUI image suitable for Vast.ai."
LABEL maintainer="Vast.ai Inc <contact@vast.ai>"

# Copy Supervisor configuration and startup scripts
COPY ./ROOT /

# Required or we will not build
ARG OOBABOOGA_REF

RUN /bin/bash -c 'set -euo pipefail && \
    [[ -n "${OOBABOOGA_REF}" ]] || { echo "Must specify OOBABOOGA_REF" && exit 1; } && \
    source /venv/main/bin/activate && \
    torch_version_pre="$(python -c "import torch; print (torch.__version__)")" && \
    pip install xformers torch==$PYTORCH_VERSION --index-url "${PYTORCH_INDEX_URL}" && \
    cd /opt/workspace-internal && \
    git clone https://github.com/oobabooga/text-generation-webui && \
    cd /opt/workspace-internal/text-generation-webui && \
    git checkout "${OOBABOOGA_REF}" && \
    pip install --no-cache-dir \
        -r requirements.txt && \
    torch_version_post="$(python -c "import torch; print (torch.__version__)")" && \
    [[ $torch_version_pre = $torch_version_post ]] || { echo "PyTorch version mismatch (wanted ${torch_version_pre} but got ${torch_version_post})"; exit 1; }'
