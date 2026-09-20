# syntax=docker/dockerfile:1

ARG ROOTFS_IMAGE=ghcr.io/iximiuz/labs/rootfs:archlinux-0f076c61

# ─── Stage 1: toolchain ───────────────────────────────────────────────────────
# Downloads, verifies and installs OpenTofu + the gcloud CLI. Archives,
# checksums and the SDK component installer never reach the final image.
FROM ${ROOTFS_IMAGE} AS toolchain

ARG OPENTOFU_VERSION=1.12.6
ARG GCLOUD_VERSION=585.0.0
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

# ─── System update + terminal tooling ─────────────────────────────────────────
# Keyring first so the full upgrade does not fail on stale signing keys.
RUN <<EOF
set -eu
pacman -Sy --noconfirm --needed archlinux-keyring
pacman -Su --noconfirm
pacman -S --noconfirm --needed \
  bash-completion \
  starship \
  eza \
  bat \
  fzf \
  zoxide \
  ripgrep \
  fd \
  git \
  git-delta \
  jq \
  go-yq \
  tmux \
  btop \
  tree \
  less \
  k9s \
  kubectx
rm -rf /var/cache/pacman/pkg/*
EOF

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

# ─── Prompt theme (no Nerd Font needed, renders in the browser terminal) ──────
RUN <<EOF
set -eu
cat > /etc/starship.toml <<'TOML'
add_newline = true
command_timeout = 1000

format = """
$directory\
$git_branch\
$git_status\
${custom.tofu}\
$kubernetes\
$gcloud\
$cmd_duration\
$line_break\
$character"""

[character]
success_symbol = "[❯](bold green)"
error_symbol = "[❯](bold red)"

[directory]
style = "bold cyan"
truncation_length = 3
truncate_to_repo = true

[git_branch]
symbol = "git:"
style = "bold purple"
format = "[$symbol$branch]($style) "

[git_status]
style = "red"
format = "([\\[$all_status$ahead_behind\\]]($style) )"

[custom.tofu]
command = "tofu workspace show 2>/dev/null"
detect_extensions = ["tf", "tofu"]
shell = ["bash", "--noprofile", "--norc"]
style = "bold yellow"
format = "[tofu:$output]($style) "

[kubernetes]
disabled = false
style = "bold blue"
format = "[k8s:$context( \\($namespace\\))]($style) "

[gcloud]
style = "bold green"
format = "[gcp:$project]($style) "

[cmd_duration]
min_time = 2000
style = "dimmed yellow"
format = "[took $duration]($style) "
TOML

cat > /etc/tmux.conf <<'TMUX'
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",xterm-256color:RGB"
set -g mouse on
set -g history-limit 50000
set -g base-index 1
setw -g pane-base-index 1
set -g status-style "bg=colour236,fg=colour250"
set -g status-left "#[bold,fg=colour39] #S "
set -g status-right "#[fg=colour245]%H:%M "
TMUX

git config --system core.pager delta
git config --system interactive.diffFilter "delta --color-only"
git config --system delta.navigate true
git config --system delta.line-numbers true
git config --system merge.conflictStyle zdiff3
EOF

# ─── Login shell env, shared by root and $LAB_USER ────────────────────────────
RUN <<EOF
set -eu
cat > /etc/profile.d/opentofu-gcloud.sh <<'PROFILE'
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
export STARSHIP_CONFIG=/etc/starship.toml
export BAT_THEME=TwoDark
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
export FZF_DEFAULT_COMMAND='fd --type f --hidden --exclude .git'
export FZF_DEFAULT_OPTS='--height 40% --layout=reverse --border'

# Everything below is interactive bash only, and runs once per shell.
[ -n "${BASH_VERSION:-}" ] || return 0
case $- in *i*) ;; *) return 0 ;; esac
[ -z "${__OTG_BASH_INIT:-}" ] || return 0
__OTG_BASH_INIT=1

HISTSIZE=10000
HISTFILESIZE=20000
HISTCONTROL=ignoreboth:erasedups
shopt -s histappend

if [ -f /opt/google-cloud-sdk/completion.bash.inc ]; then
  . /opt/google-cloud-sdk/completion.bash.inc
fi

complete -C /usr/local/bin/tofu tofu
alias tf=tofu
complete -C /usr/local/bin/tofu tf

if command -v kubectl >/dev/null 2>&1; then
  . <(kubectl completion bash)
  alias k=kubectl
  complete -o default -F __start_kubectl k
fi

alias ls='eza --group-directories-first'
alias ll='eza -l --git --group-directories-first'
alias la='eza -la --git --group-directories-first'
alias lt='eza --tree --level=2 --group-directories-first'

[ -f /usr/share/fzf/key-bindings.bash ] && . /usr/share/fzf/key-bindings.bash
[ -f /usr/share/fzf/completion.bash ] && . /usr/share/fzf/completion.bash

eval "$(zoxide init bash)"
eval "$(starship init bash)"
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
starship --version
eza --version
bat --version
fzf --version
zoxide --version
delta --version
k9s version --short
EOF

# ─── Lab user ─────────────────────────────────────────────────────────────────
USER $LAB_USER
ENV HOME=/home/$LAB_USER

RUN printf '\n. /etc/profile.d/opentofu-gcloud.sh\n' >> "$HOME/.bashrc"

RUN code-server --install-extension hashicorp.terraform \
    && code-server --install-extension googlecloudtools.cloudcode

# code-server terminal: login bash (loads the prompt), modern dark theme.
# Merges into any settings the base image already ships.
RUN <<EOF
set -eu
dir="$HOME/.local/share/code-server/User"
mkdir -p "$dir"
cat > /tmp/otg-settings.json <<'JSON'
{
  "workbench.colorTheme": "Default Dark Modern",
  "terminal.integrated.defaultProfile.linux": "bash",
  "terminal.integrated.profiles.linux": {
    "bash": { "path": "bash", "args": ["-l"] }
  },
  "terminal.integrated.fontSize": 14,
  "terminal.integrated.lineHeight": 1.2,
  "terminal.integrated.cursorStyle": "line",
  "terminal.integrated.cursorBlinking": true,
  "terminal.integrated.smoothScrolling": true,
  "terminal.integrated.scrollback": 10000,
  "editor.fontSize": 14,
  "editor.smoothScrolling": true,
  "editor.cursorSmoothCaretAnimation": "on",
  "editor.bracketPairColorization.enabled": true,
  "editor.guides.bracketPairs": "active",
  "files.associations": { "*.tofu": "terraform" },
  "[terraform]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true
  }
}
JSON
if [ -s "$dir/settings.json" ]; then
  jq -s '.[0] * .[1]' "$dir/settings.json" /tmp/otg-settings.json > /tmp/otg-merged.json
  mv /tmp/otg-merged.json "$dir/settings.json"
else
  mv /tmp/otg-settings.json "$dir/settings.json"
fi
rm -f /tmp/otg-settings.json
EOF

COPY --chown=$LAB_USER:$LAB_USER welcome $HOME/.welcome
