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
    # FIX: CVE-2026-53224, CVE-2026-64531 — upgrade linux-libc-dev to patched version
    && apt-get upgrade -y linux-libc-dev \
    && rm -rf /var/lib/apt/lists/*

# ── FIX: Remove cfssl binaries from base image (2 CRITICAL + 43 HIGH CVEs) ───
# cfssl was built with Go 1.22.0 and is the worst offender.
# Remove all cfssl binaries shipped in the base image.
RUN rm -f \
    /usr/bin/cfssl \
    /usr/bin/cfssl-bundle \
    /usr/bin/cfssl-certinfo \
    /usr/bin/cfssl-newkey \
    /usr/bin/cfssl-scan \
    /usr/bin/cfssljson \
    || true

# ── FIX: Remove dive (30 HIGH CVEs from old Go runtime) ─────────────────────
RUN rm -f /usr/bin/dive || true

# ── OpenTofu repository + install ─────────────────────────────────────────────
# Pinned to latest stable — tofu ships its own Go runtime; keep updated.
RUN install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://get.opentofu.org/opentofu.gpg \
        | tee /etc/apt/keyrings/opentofu.gpg >/dev/null \
    && curl -fsSL https://packages.opentofu.org/opentofu/tofu/gpgkey \
        | gpg --no-tty --batch --dearmor \
            -o /etc/apt/keyrings/opentofu-repo.gpg >/dev/null \
    && chmod a+r \
        /etc/apt/keyrings/opentofu.gpg \
        /etc/apt/keyrings/opentofu-repo.gpg \
    && printf \
        'deb [signed-by=/etc/apt/keyrings/opentofu.gpg,/etc/apt/keyrings/opentofu-repo.gpg] https://packages.opentofu.org/opentofu/tofu/any/ any main\ndeb-src [signed-by=/etc/apt/keyrings/opentofu.gpg,/etc/apt/keyrings/opentofu-repo.gpg] https://packages.opentofu.org/opentofu/tofu/any/ any main\n' \
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

# ── FIX: Python vulns — CVE-2026-69247, CVE-2026-69249, GHSA-537c-gmf6-5ccf ─
# The gcloud SDK bundles its own Python env. Upgrade cryptography in BOTH
# the system Python and the gcloud bundled virtualenv.
RUN python3 -m pip install --break-system-packages --upgrade \
        "cryptography>=50.0.0"

# Upgrade cryptography inside gcloud's bundled Python virtualenv if present
RUN GCLOUD_PY="$(find /usr/lib/google-cloud-sdk -name 'python3' -type f 2>/dev/null | head -1)" \
    && if [ -n "$GCLOUD_PY" ]; then \
         "$GCLOUD_PY" -m pip install --upgrade "cryptography>=50.0.0" 2>/dev/null || true; \
       fi

# ── FIX: Node.js vulns in code-server bundled deps ───────────────────────────
# CVE-2026-69192 (ip-address), CVE-2026-59869 / GHSA-5p4m-2wfm-xmqj (js-yaml),
# CVE-2026-73566 (tar), CVE-2026-13697 (undici)
# Patch each package inside code-server's node_modules using npm overrides.
RUN CODE_SERVER_MODULES="$(find /usr/lib/code-server /opt/code-server \
        -maxdepth 4 -name node_modules -type d 2>/dev/null | head -1)" \
    && if [ -n "$CODE_SERVER_MODULES" ]; then \
        # Navigate to the code-server root (parent of node_modules) and patch \
        CS_ROOT="$(dirname "$CODE_SERVER_MODULES")"; \
        cd "$CS_ROOT" && \
        npm install --save \
            "ip-address@>=10.3.1" \
            "js-yaml@>=4.3.1" \
            "tar@>=7.5.21" \
            "undici@>=7.29.0" \
        2>/dev/null || true; \
       fi

# ── FIX: fzf — rebuild/replace with a version using Go 1.26.5+ ───────────────
# fzf from the base image was built with a vulnerable Go runtime (8 HIGH CVEs).
# Install the latest upstream fzf release, which ships pre-built binaries
# compiled with a current Go toolchain.
RUN FZF_VER="$(curl -fsSL https://api.github.com/repos/junegunn/fzf/releases/latest \
        | grep '"tag_name"' | sed 's/.*"v\([^"]*\)".*/\1/')" \
    && ARCH="$(dpkg --print-architecture)" \
    && curl -fsSL \
        "https://github.com/junegunn/fzf/releases/download/v${FZF_VER}/fzf-${FZF_VER}-linux_${ARCH}.tar.gz" \
        -o /tmp/fzf.tar.gz \
    && tar -xzf /tmp/fzf.tar.gz -C /usr/local/bin fzf \
    && chmod 0755 /usr/local/bin/fzf \
    # Replace the old copies with the patched binary
    && for OLD in /usr/bin/fzf $(which fzf 2>/dev/null); do \
           [ "$OLD" != "/usr/local/bin/fzf" ] && cp /usr/local/bin/fzf "$OLD" 2>/dev/null || true; \
       done \
    && rm -f /tmp/fzf.tar.gz

# ── Smoke test ────────────────────────────────────────────────────────────────
RUN tofu version \
    && gcloud version \
    && gke-gcloud-auth-plugin --version \
    && python3 -c "import cryptography; print('cryptography', cryptography.__version__)"

# ── Login shell env ───────────────────────────────────────────────────────────
RUN printf 'export USE_GKE_GCLOUD_AUTH_PLUGIN=True\n' > /etc/profile.d/gcloud.sh \
    && chmod +x /etc/profile.d/gcloud.sh
RUN printf '\nsource /etc/profile.d/gcloud.sh\n' >> /root/.bashrc

# ── Lab user shell env ────────────────────────────────────────────────────────
USER $LAB_USER
ENV HOME=/home/$LAB_USER
RUN printf '\nsource /etc/profile.d/gcloud.sh\n' >> "$HOME/.bashrc"

# ── VS Code extensions ────────────────────────────────────────────────────────
# FIX: terraform-ls ships with Go 1.25.12 (9 HIGH CVEs).
# Installing the latest extension version pulls the newest terraform-ls binary.
RUN code-server --install-extension hashicorp.terraform@latest \
    && code-server --install-extension googlecloudtools.cloudcode@latest

# ── Welcome ───────────────────────────────────────────────────────────────────
COPY welcome $HOME/.welcome
