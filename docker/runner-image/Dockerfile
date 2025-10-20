FROM summerwind/actions-runner:latest

USER root

# Install system packages
RUN apt-get update && apt-get install -y \
    python3-pip \
    python3-dev \
    build-essential \
    jq \
    bc \
    wget \
    curl \
    ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Python tools
RUN pip3 install --no-cache-dir \
    semgrep \
    yamllint

# Install kubectl (architecture-aware)
RUN ARCH=$(dpkg --print-architecture) \
    && curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/${ARCH}/kubectl" \
    && chmod +x kubectl \
    && mv kubectl /usr/local/bin/

# Install kubeconform (architecture-aware)
RUN ARCH=$(dpkg --print-architecture) \
    && wget -q https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-${ARCH}.tar.gz \
    && tar xf kubeconform-linux-${ARCH}.tar.gz \
    && mv kubeconform /usr/local/bin/ \
    && rm kubeconform-linux-${ARCH}.tar.gz

# Install kube-score (architecture-aware)
RUN ARCH=$(dpkg --print-architecture) \
    && wget -q https://github.com/zegl/kube-score/releases/download/v1.18.0/kube-score_1.18.0_linux_${ARCH} \
    && chmod +x kube-score_1.18.0_linux_${ARCH} \
    && mv kube-score_1.18.0_linux_${ARCH} /usr/local/bin/kube-score

# Install yq (architecture-aware)
RUN ARCH=$(dpkg --print-architecture) \
    && wget -qO /usr/local/bin/yq \
    "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_${ARCH}" \
    && chmod +x /usr/local/bin/yq

# Note: oasdiff installation skipped (requires Go toolchain or specific binary format)
# It will be installed at workflow runtime in contract-validation.yml

# Verify installations
RUN semgrep --version \
    && yamllint --version \
    && jq --version \
    && kubectl version --client \
    && kubeconform -v \
    && kube-score version \
    && yq --version

USER runner
