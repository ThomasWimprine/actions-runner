FROM summerwind/actions-runner:latest

USER root

# Install system packages
RUN apt-get update && apt-get install -y \
    python3-pip \
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

# Install kubectl
RUN curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl" \
    && chmod +x kubectl \
    && mv kubectl /usr/local/bin/

# Install kubeconform
RUN wget -q https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz \
    && tar xf kubeconform-linux-amd64.tar.gz \
    && mv kubeconform /usr/local/bin/ \
    && rm kubeconform-linux-amd64.tar.gz

# Install kube-score
RUN wget -q https://github.com/zegl/kube-score/releases/download/v1.18.0/kube-score_1.18.0_linux_amd64 \
    && chmod +x kube-score_1.18.0_linux_amd64 \
    && mv kube-score_1.18.0_linux_amd64 /usr/local/bin/kube-score

# Install yq
RUN wget -qO /usr/local/bin/yq \
    "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64" \
    && chmod +x /usr/local/bin/yq

# Install oasdiff
RUN curl -L https://github.com/Tufin/oasdiff/releases/latest/download/oasdiff_linux_amd64 -o /usr/local/bin/oasdiff \
    && chmod +x /usr/local/bin/oasdiff

# Verify installations
RUN semgrep --version \
    && yamllint --version \
    && jq --version \
    && kubectl version --client \
    && kubeconform -v \
    && kube-score version \
    && yq --version \
    && oasdiff version

USER runner
