ARG PYTORCH_BASE=vastai/pytorch:2.5.1-cuda-12.1.1

FROM ${PYTORCH_BASE}

# Maintainer details
LABEL org.opencontainers.image.source="https://github.com/vastai/"
LABEL org.opencontainers.image.description="Stable Diffusion A1111 image suitable for Vast.ai."
LABEL maintainer="Vast.ai Inc <contact@vast.ai>"

# Copy Supervisor configuration and startup scripts
COPY ./ROOT /

# Required or we will not build
ARG KOHYA_REF

# Kohya's GUI provides some great install scripts for local use, but we will manually configure everything we need
RUN /bin/bash -c '[[ -n "${KOHYA_REF}" ]] || { echo "Must specify KOHYA_REF" && exit 1; } && \
    source /venv/main/bin/activate && \
    torch_version_pre="$(python -c "import torch; print (torch.__version__)")" && \
    pip install xformers torch==$PYTORCH_VERSION --index-url "${PYTORCH_INDEX_URL}" && \
    cd /opt/workspace-internal/ && \
    git clone --recursive https://github.com/bmaltais/kohya_ss.git && \
    cd /opt/workspace-internal/kohya_ss && \
    git checkout "${KOHYA_REF}" && \
    git submodule update --init --recursive && \
    grep -v "^--extra-index-url\|^torch==\|^torchvision==\|^xformers==\|^tensorboard\|^tensorflow" requirements_linux.txt > requirements_linux_modified.txt && \
    pip install --no-cache-dir \
        torch==${PYTORCH_VERSION} \
        -r requirements_linux_modified.txt && \
    rm -f requirements_linux_modified.txt && \
    sed -i "s|link=f\"http://localhost:{self.tensorboard_port}\"|link=f\"http://{os.environ.get('\''PUBLIC_IPADDR'\'')}:{os.environ.get('\''VAST_TCP_PORT_6006'\'')}\"|" \
        /opt/workspace-internal/kohya_ss/kohya_gui/class_tensorboard.py && \
    mkdir -p "/root/.cache/huggingface/accelerate" && \
    cp config_files/accelerate/default_config.yaml /root/.cache/huggingface/accelerate/default_config.yaml && \
    torch_version_post="$(python -c "import torch; print (torch.__version__)")" && \
    [[ $torch_version_pre = $torch_version_post ]] || { echo "PyTorch version mismatch (wanted ${torch_version_pre} but got ${torch_version_post})"; exit 1; }'
