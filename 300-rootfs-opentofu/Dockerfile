
# syntax=docker/dockerfile:1

FROM ghcr.io/iximiuz/labs/rootfs:ubuntu-docker

ARG LAB_USER=laborant

# ── System dependencies ───────────────────────────────────────────────────────
USER root
ENV HOME=/root

RUN apt-get update && apt-get install -y \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg \
        git \
        jq \
        unzip \
        python3 \
        python3-pip \
    && rm -rf /var/lib/apt/lists/*

# ── OpenTofu repository + install ─────────────────────────────────────────────
RUN install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://get.opentofu.org/opentofu.gpg \
        | tee /etc/apt/keyrings/opentofu.gpg >/dev/null \
    && curl -fsSL https://packages.opentofu.org/opentofu/tofu/gpgkey \
        | gpg --no-tty --batch --dearmor -o /etc/apt/keyrings/opentofu-repo.gpg >/dev/null \
    && chmod a+r /etc/apt/keyrings/opentofu.gpg /etc/apt/keyrings/opentofu-repo.gpg \
    && echo \
        "deb [signed-by=/etc/apt/keyrings/opentofu.gpg,/etc/apt/keyrings/opentofu-repo.gpg] https://packages.opentofu.org/opentofu/tofu/any/ any main\ndeb-src [signed-by=/etc/apt/keyrings/opentofu.gpg,/etc/apt/keyrings/opentofu-repo.gpg] https://packages.opentofu.org/opentofu/tofu/any/ any main" \
        | tee /etc/apt/sources.list.d/opentofu.list >/dev/null \
    && chmod a+r /etc/apt/sources.list.d/opentofu.list \
    && apt-get update \
    && apt-get install -y tofu \
    && rm -rf /var/lib/apt/lists/*

# ── Google Cloud CLI repository + install ────────────────────────────────────
RUN echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
        | tee /etc/apt/sources.list.d/google-cloud-sdk.list >/dev/null \
    && curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
        | gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg \
    && apt-get update \
    && apt-get install -y \
        google-cloud-cli \
        google-cloud-cli-gke-gcloud-auth-plugin \
    && rm -rf /var/lib/apt/lists/*

# ── Smoke test ────────────────────────────────────────────────────────────────
RUN tofu version \
    && gcloud version \
    && gke-gcloud-auth-plugin --version

# ── Login shell env ───────────────────────────────────────────────────────────
RUN printf 'export USE_GKE_GCLOUD_AUTH_PLUGIN=True\n' > /etc/profile.d/gcloud.sh \
    && chmod +x /etc/profile.d/gcloud.sh

RUN printf '\nsource /etc/profile.d/gcloud.sh\n' >> /root/.bashrc

# ── Lab user shell env ────────────────────────────────────────────────────────
USER $LAB_USER
ENV HOME=/home/$LAB_USER

RUN printf '\nsource /etc/profile.d/gcloud.sh\n' >> "$HOME/.bashrc"

# ── VS Code extensions ────────────────────────────────────────────────────────
RUN code-server --install-extension hashicorp.terraform \
    && code-server --install-extension googlecloudtools.cloudcode

# ── Welcome ───────────────────────────────────────────────────────────────────
COPY welcome $HOME/.welcome
