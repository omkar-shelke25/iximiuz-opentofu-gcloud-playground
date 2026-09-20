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
# The base image already ships fzf (~/.fzf), btop, jq and yq.
# ghostty-terminfo keeps full-screen apps working over `labctl ssh` from Ghostty.
RUN <<EOF
set -eu
pacman -Sy --noconfirm --needed archlinux-keyring
pacman -Su --noconfirm
pacman -S --noconfirm --needed \
  bash-completion \
  zsh \
  zsh-autosuggestions \
  zsh-syntax-highlighting \
  starship \
  eza \
  bat \
  zoxide \
  ripgrep \
  fd \
  git \
  git-delta \
  lazygit \
  atuin \
  fastfetch \
  ghostty-terminfo \
  tmux \
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

# ─── Prompt themes: Catppuccin Mocha ──────────────────────────────────────────
# starship.toml uses Nerd Font glyphs and powerline arrows (`icons on`).
# starship-plain.toml uses colored blocks only, the default for browser tabs.
RUN <<EOF
set -eu
cat > /etc/starship.toml <<'TOML'
"$schema" = 'https://starship.rs/config-schema.json'

format = """
[\ue0b6](red)\
$os\
$hostname\
[\ue0b0](bg:peach fg:red)\
$directory\
[\ue0b0](bg:yellow fg:peach)\
$git_branch\
$git_status\
[\ue0b0](fg:yellow bg:green)\
$terraform\
[\ue0b0](fg:green bg:sapphire)\
$gcloud\
$kubernetes\
[\ue0b0](fg:sapphire bg:lavender)\
$time\
[\ue0b4 ](fg:lavender)\
$cmd_duration\
$line_break\
$status\
$character"""

palette = 'catppuccin_mocha'

[palettes.catppuccin_mocha]
rosewater = "#f5e0dc"
flamingo = "#f2cdcd"
pink = "#f5c2e7"
mauve = "#cba6f7"
red = "#f38ba8"
maroon = "#eba0ac"
peach = "#fab387"
yellow = "#f9e2af"
green = "#a6e3a1"
teal = "#94e2d5"
sky = "#89dceb"
sapphire = "#74c7ec"
blue = "#89b4fa"
lavender = "#b4befe"
text = "#cdd6f4"
subtext1 = "#bac2de"
subtext0 = "#a6adc8"
overlay2 = "#9399b2"
overlay1 = "#7f849c"
overlay0 = "#6c7086"
surface2 = "#585b70"
surface1 = "#45475a"
surface0 = "#313244"
base = "#1e1e2e"
mantle = "#181825"
crust = "#11111b"

[os]
disabled = false
style = "bg:red fg:crust"
format = "[$symbol ]($style)"

[os.symbols]
Arch = "\U000f08c7"
Linux = "\U000f033d"

[hostname]
ssh_only = false
style = "bg:red fg:crust"
format = "[$hostname ]($style)"

[directory]
style = "bg:peach fg:crust"
format = "[ \uf07c $path ]($style)[$read_only]($read_only_style)"
read_only = "\uf023 "
read_only_style = "bg:peach fg:red"
truncation_length = 3
truncation_symbol = "…/"

[git_branch]
symbol = "\uf418"
style = "bg:yellow"
format = "[[ $symbol $branch ](fg:crust bg:yellow)]($style)"

[git_status]
style = "bg:yellow"
format = "[[($all_status$ahead_behind )](fg:crust bg:yellow)]($style)"

[terraform]
symbol = "\U000f1062"
detect_extensions = ["tf", "tofu", "tfplan", "tfstate"]
style = "bg:green"
format = "[[ $symbol $workspace ](fg:crust bg:green)]($style)"

[gcloud]
symbol = "\U000f11f6"
style = "bg:sapphire"
format = "[[ $symbol $project ](fg:crust bg:sapphire)]($style)"

[kubernetes]
disabled = false
symbol = "\U000f10fe"
style = "bg:sapphire"
format = "[[ $symbol $context( \\($namespace\\)) ](fg:crust bg:sapphire)]($style)"

[time]
disabled = false
time_format = "%H:%M"
style = "bg:lavender"
format = "[[ \uf017 $time ](fg:crust bg:lavender)]($style)"

[cmd_duration]
min_time = 2000
style = "fg:yellow"
format = "[\uf252 $duration ]($style)"

[status]
disabled = false
style = "bold fg:red"
format = "[\u2718 $status ]($style)"

[character]
success_symbol = "[\u276f](bold fg:green)"
error_symbol = "[\u276f](bold fg:red)"
TOML

cat > /etc/starship-plain.toml <<'TOML'
add_newline = true
format = "$hostname$directory$git_branch$git_status$terraform$gcloud$kubernetes$time$cmd_duration$line_break$status$character"

[hostname]
ssh_only = false
style = "bg:#f38ba8 fg:#11111b"
format = "[ $hostname ]($style)"

[directory]
truncation_length = 3
style = "bg:#fab387 fg:#11111b"
format = "[ $path ]($style)[$read_only]($read_only_style)"
read_only = " ro"

[git_branch]
symbol = "git:"
style = "bg:#f9e2af fg:#11111b"
format = "[ $symbol$branch ]($style)"

[git_status]
style = "bg:#f9e2af fg:#11111b"
format = "([$all_status$ahead_behind ]($style))"

[terraform]
symbol = "tofu:"
detect_extensions = ["tf", "tofu", "tfplan", "tfstate"]
style = "bg:#a6e3a1 fg:#11111b"
format = "[ $symbol$workspace ]($style)"

[gcloud]
symbol = "gcp:"
style = "bg:#74c7ec fg:#11111b"
format = "[ $symbol$project ]($style)"

[kubernetes]
disabled = false
symbol = "k8s:"
style = "bg:#74c7ec fg:#11111b"
format = "[ $symbol$context( \\($namespace\\)) ]($style)"

[time]
disabled = false
time_format = "%H:%M"
style = "bg:#b4befe fg:#11111b"
format = "[ $time ]($style)"

[cmd_duration]
min_time = 2000
style = "fg:#f9e2af"
format = " [took $duration]($style)"

[status]
disabled = false
style = "bold fg:#f38ba8"
format = "[$status ]($style)"

[character]
success_symbol = "[>](bold fg:#a6e3a1)"
error_symbol = "[>](bold fg:#f38ba8)"
TOML

mkdir -p /etc/fastfetch
cat > /etc/fastfetch/config.jsonc <<'JSON'
{
  "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
  "logo": { "source": "arch_small", "padding": { "top": 1, "right": 3 } },
  "display": { "separator": "  " },
  "modules": [
    "title", "separator", "os", "kernel", "uptime", "shell",
    "cpu", "memory", "disk", "localip",
    "break",
    { "type": "command", "key": "OpenTofu", "text": "tofu version | head -n1 | cut -d' ' -f2" },
    { "type": "command", "key": "gcloud", "text": "cat /opt/google-cloud-sdk/VERSION" },
    { "type": "command", "key": "kubectl", "text": "kubectl version --client 2>/dev/null | head -n1 | cut -d' ' -f3" },
    "break", "colors"
  ]
}
JSON

cat > /etc/tmux.conf <<'TMUX'
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",xterm-256color:RGB"
set -g mouse on
set -g history-limit 50000
set -g base-index 1
setw -g pane-base-index 1
set -g status-style "bg=#313244,fg=#cdd6f4"
set -g status-left "#[bold,fg=#11111b,bg=#a6e3a1] #S "
set -g status-right "#[fg=#11111b,bg=#b4befe] %H:%M "
TMUX

git config --system core.pager delta
git config --system interactive.diffFilter "delta --color-only"
git config --system delta.navigate true
git config --system delta.line-numbers true
git config --system delta.syntax-theme "Catppuccin Mocha"
git config --system merge.conflictStyle zdiff3
EOF

# ─── Shared env, sourced by bash and zsh ──────────────────────────────────────
RUN <<EOF
set -eu
cat > /etc/profile.d/opentofu-gcloud.sh <<'PROFILE'
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
export BAT_THEME="Catppuccin Mocha"
export MANPAGER="sh -c 'col -bx | bat -l man -p'"

# Bash fallback: plain prompt and the basics. zsh is the default shell.
[ -n "${BASH_VERSION:-}" ] || return 0
case $- in *i*) ;; *) return 0 ;; esac
[ -z "${__OTG_BASH_INIT:-}" ] || return 0
__OTG_BASH_INIT=1

export STARSHIP_CONFIG=/etc/starship-plain.toml
if [ -f /opt/google-cloud-sdk/completion.bash.inc ]; then
  . /opt/google-cloud-sdk/completion.bash.inc
fi
complete -C /usr/local/bin/tofu tofu
alias tf=tofu
complete -C /usr/local/bin/tofu tf
. <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
alias ls='eza --group-directories-first'
alias ll='eza -la --git --group-directories-first'
eval "$(zoxide init bash)"
eval "$(starship init bash)"
PROFILE
chmod 0644 /etc/profile.d/opentofu-gcloud.sh
printf '\n. /etc/profile.d/opentofu-gcloud.sh\n' >> /root/.bashrc
printf '\n. /etc/profile.d/opentofu-gcloud.sh\n' >> "/home/${LAB_USER}/.bashrc"
EOF

# ─── zsh: system-wide config, per-user stubs, default shell ───────────────────
RUN <<EOF
set -eu
cat > /etc/zsh/playground.zsh <<'ZSH'
# Shared zsh config for the OpenTofu playground.
# Personal additions go in ~/.zshrc.local

emulate sh -c '. /etc/profile.d/opentofu-gcloud.sh'
typeset -U path PATH
export LANG=${LANG:-C.UTF-8}
export EDITOR=${EDITOR:-vim} VISUAL=${VISUAL:-vim}

# History
HISTFILE=~/.zsh_history
HISTSIZE=100000
SAVEHIST=100000
setopt SHARE_HISTORY HIST_IGNORE_ALL_DUPS HIST_IGNORE_SPACE HIST_REDUCE_BLANKS INTERACTIVE_COMMENTS

# Keys
bindkey -e
bindkey '^[[H' beginning-of-line
bindkey '^[[F' end-of-line
bindkey '^[[3~' delete-char
bindkey '^[[1;5C' forward-word
bindkey '^[[1;5D' backward-word

# Completion
autoload -Uz compinit && compinit -i
autoload -U +X bashcompinit && bashcompinit
zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list 'm:{a-z}={A-Z}'
complete -o nospace -C /usr/local/bin/tofu tofu
complete -o nospace -C /usr/local/bin/tofu tf
source <(kubectl completion zsh)
compdef k=kubectl
[[ -f /opt/google-cloud-sdk/completion.zsh.inc ]] && source /opt/google-cloud-sdk/completion.zsh.inc

# Aliases
alias cat='bat --paging=never --style=plain'
alias grep='grep --color=auto'
alias top='btop'
alias lg='lazygit'
alias tf='tofu'
alias k='kubectl'

# fzf: Ctrl+T files, Alt+C directories (Ctrl+R is atuin)
export FZF_DEFAULT_COMMAND='fd --type f --hidden --exclude .git --exclude .terraform'
export FZF_CTRL_T_COMMAND=$FZF_DEFAULT_COMMAND
export FZF_ALT_C_COMMAND='fd --type d --hidden --exclude .git --exclude .terraform'
export FZF_DEFAULT_OPTS='--height 50% --layout=reverse --border=rounded --info=inline --color=bg+:#313244,spinner:#f5e0dc,hl:#f38ba8,fg:#cdd6f4,header:#f38ba8,info:#cba6f7,pointer:#f5e0dc,marker:#b4befe,fg+:#cdd6f4,prompt:#cba6f7,hl+:#f38ba8,selected-bg:#45475a,border:#6c7086,label:#cdd6f4'
export FZF_CTRL_T_OPTS="--preview 'bat --color=always --style=numbers --line-range=:200 {}'"
if [[ -f ~/.fzf.zsh ]]; then
  source ~/.fzf.zsh
elif (( $+commands[fzf] )); then
  source <(fzf --zsh)
fi

# Icons need a Nerd Font on the machine that draws the terminal.
# Browser tabs cannot load one, so icons default to off. They turn on for
# Ghostty, Kitty and WezTerm over `labctl ssh`. `icons on|off` overrides
# and remembers the choice.
_icons_default() {
  if [[ -f $HOME/.config/term-icons ]]; then print -r -- "$(<$HOME/.config/term-icons)"; return; fi
  case "$TERM" in
    xterm-ghostty|xterm-kitty|wezterm) print on ;;
    *) print off ;;
  esac
}
_apply_icons() {
  local i=--icons=auto
  export STARSHIP_CONFIG=/etc/starship.toml
  if [[ $TERM_ICONS == off ]]; then
    i=--icons=never
    export STARSHIP_CONFIG=/etc/starship-plain.toml
  fi
  alias ls="eza $i --group-directories-first"
  alias ll="eza -la $i --git --group-directories-first"
  alias tree="eza --tree $i"
  export FZF_ALT_C_OPTS="--preview 'eza --tree --level=2 ${i/auto/always} --color=always {}'"
}
icons() {
  [[ $1 == on || $1 == off ]] || { print "usage: icons on|off"; return 1; }
  TERM_ICONS=$1
  mkdir -p $HOME/.config
  print -r -- $1 > $HOME/.config/term-icons
  _apply_icons
}
TERM_ICONS=$(_icons_default)
_apply_icons

# History search, smart cd, prompt
eval "$(atuin init zsh --disable-up-arrow)"
eval "$(zoxide init zsh)"
eval "$(starship init zsh)"

# System info once per session, then the playground welcome on first login
if [[ -t 1 && -z $FASTFETCH_SHOWN ]]; then
  export FASTFETCH_SHOWN=1
  fastfetch -c /etc/fastfetch/config.jsonc
fi
if [[ -t 0 && -f $HOME/.welcome ]]; then
  if [[ -x $HOME/.welcome ]]; then "$HOME/.welcome"; else command cat "$HOME/.welcome"; fi
  echo
  rm -f "$HOME/.welcome"
fi
ZSH

cat > /etc/zsh/playground-plugins.zsh <<'ZSH'
# Loaded last: syntax highlighting must come after every other widget.
source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=#6c7086'
ZSH_AUTOSUGGEST_STRATEGY=(history completion)
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
ZSH

for home in /root "/home/${LAB_USER}"; do
  cat > "$home/.zshrc" <<'ZSH'
# Playground defaults live in /etc/zsh. Personal additions go in ~/.zshrc.local
source /etc/zsh/playground.zsh
[[ -f ~/.zshrc.local ]] && source ~/.zshrc.local
source /etc/zsh/playground-plugins.zsh
ZSH
  mkdir -p "$home/.config/atuin"
  cat > "$home/.config/atuin/config.toml" <<'TOML'
update_check = false
style = "compact"
inline_height = 20
show_preview = true
TOML
done
chown -R "${LAB_USER}:" "/home/${LAB_USER}/.zshrc" "/home/${LAB_USER}/.config"

usermod -s /usr/bin/zsh root
usermod -s /usr/bin/zsh "${LAB_USER}"
EOF

# ─── Smoke test ───────────────────────────────────────────────────────────────
RUN <<EOF
set -eu
tofu version
gcloud version
kubectl version --client
gke-gcloud-auth-plugin --version
zsh --version
starship --version
atuin --version
lazygit --version
fastfetch --version
delta --version
zsh -ic 'exit'
rm -f /root/.zcompdump*
EOF

# ─── Lab user ─────────────────────────────────────────────────────────────────
USER $LAB_USER
ENV HOME=/home/$LAB_USER

RUN code-server --install-extension hashicorp.terraform \
    && code-server --install-extension googlecloudtools.cloudcode \
    && code-server --install-extension catppuccin.catppuccin-vsc

# code-server: zsh login terminal, Catppuccin Mocha theme. The font list lets
# a locally installed Nerd Font draw icons after `icons on`.
# Merges into the settings the base image already ships.
RUN <<EOF
set -eu
dir="$HOME/.local/share/code-server/User"
mkdir -p "$dir"
cat > /tmp/otg-settings.json <<'JSON'
{
  "workbench.colorTheme": "Catppuccin Mocha",
  "terminal.integrated.defaultProfile.linux": "zsh",
  "terminal.integrated.profiles.linux": {
    "zsh": { "path": "zsh", "args": ["-l"] },
    "bash": { "path": "bash", "args": ["-l"] }
  },
  "terminal.integrated.fontFamily": "'JetBrainsMono Nerd Font', 'JetBrains Mono', Menlo, Consolas, monospace",
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

RUN zsh -ic 'exit' && rm -f "$HOME"/.zcompdump*

COPY --chown=$LAB_USER:$LAB_USER welcome $HOME/.welcome
