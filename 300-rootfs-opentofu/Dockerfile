# syntax=docker/dockerfile:1

ARG ROOTFS_IMAGE=ghcr.io/iximiuz/labs/rootfs:archlinux-0f076c61

# ─── Stage 1: toolchain ───────────────────────────────────────────────────────
# Downloads, verifies and installs OpenTofu + the gcloud CLI. Archives,
# checksums and the SDK component installer never reach the final image.
FROM ${ROOTFS_IMAGE} AS toolchain

ARG OPENTOFU_VERSION=1.12.6
ARG GCLOUD_VERSION=583.0.0
ARG GCLOUD_COMPONENTS="gke-gcloud-auth-plugin kubectl"
# Optional pin. Take it from the versioned-archives page for GCLOUD_VERSION.
ARG GCLOUD_SHA256=""

USER root
ENV HOME=/root
WORKDIR /tmp/build

# OpenTofu, verified against the release SHA256SUMS.
RUN <<EOF
set -eu
base="https://github.com/opentofu/opentofu/releases/download/v${OPENTOFU_VERSION}"
curl -fsSL -O "${base}/tofu_${OPENTOFU_VERSION}_linux_amd64.tar.gz"
curl -fsSL -O "${base}/tofu_${OPENTOFU_VERSION}_SHA256SUMS"
sha256sum --ignore-missing -c "tofu_${OPENTOFU_VERSION}_SHA256SUMS"
tar -xzf "tofu_${OPENTOFU_VERSION}_linux_amd64.tar.gz" -C /usr/local/bin tofu
chmod 0755 /usr/local/bin/tofu
EOF

# gcloud CLI from the versioned archive. The linux/amd64 archive ships its own
# Python interpreter, so the final image needs no Python of its own.
RUN <<EOF
set -eu
curl -fsSL -o gcloud.tar.gz \
  "https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-${GCLOUD_VERSION}-linux-x86_64.tar.gz"
if [ -n "${GCLOUD_SHA256}" ]; then
  echo "${GCLOUD_SHA256}  gcloud.tar.gz" | sha256sum -c -
fi
tar -xzf gcloud.tar.gz -C /opt
/opt/google-cloud-sdk/install.sh \
  --quiet \
  --usage-reporting=false \
  --path-update=false \
  --command-completion=false \
  --additional-components ${GCLOUD_COMPONENTS}
rm -rf /opt/google-cloud-sdk/.install/.backup
find /opt/google-cloud-sdk -type d -name __pycache__ -prune -exec rm -rf {} +
EOF

# ─── Stage 2: playground rootfs ───────────────────────────────────────────────
FROM ${ROOTFS_IMAGE}

ARG LAB_USER=laborant

USER root
ENV HOME=/root

COPY --from=toolchain /usr/local/bin/tofu /usr/local/bin/tofu
COPY --from=toolchain /opt/google-cloud-sdk /opt/google-cloud-sdk

RUN <<EOF
set -eu
for bin in gcloud gsutil bq kubectl gke-gcloud-auth-plugin; do
  if [ -x "/opt/google-cloud-sdk/bin/${bin}" ]; then
    ln -sf "/opt/google-cloud-sdk/bin/${bin}" "/usr/local/bin/${bin}"
  fi
done
EOF

# ─── Login shell env, shared by root and $LAB_USER ────────────────────────────
RUN <<EOF
set -eu
cat > /etc/profile.d/opentofu-gcloud.sh <<'PROFILE'
export USE_GKE_GCLOUD_AUTH_PLUGIN=True

if [ -n "${BASH_VERSION:-}" ]; then
  if [ -f /opt/google-cloud-sdk/completion.bash.inc ]; then
    . /opt/google-cloud-sdk/completion.bash.inc
  fi
  complete -C /usr/local/bin/tofu tofu
fi
PROFILE
chmod 0755 /etc/profile.d/opentofu-gcloud.sh
printf '\n. /etc/profile.d/opentofu-gcloud.sh\n' >> /root/.bashrc
EOF

# ─── Smoke test ───────────────────────────────────────────────────────────────
RUN <<EOF
set -eu
tofu version
gcloud version
kubectl version --client
gke-gcloud-auth-plugin --version
EOF

# ─── Lab user ─────────────────────────────────────────────────────────────────
USER $LAB_USER
ENV HOME=/home/$LAB_USER

RUN printf '\n. /etc/profile.d/opentofu-gcloud.sh\n' >> "$HOME/.bashrc"

RUN code-server --install-extension hashicorp.terraform \
    && code-server --install-extension googlecloudtools.cloudcode

COPY --chown=$LAB_USER:$LAB_USER welcome $HOME/.welcome
