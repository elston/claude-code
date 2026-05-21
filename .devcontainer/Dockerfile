FROM node:22-bookworm

ARG TZ
ENV TZ="$TZ"

ARG CLAUDE_CODE_VERSION=latest

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# --- Системные утилиты и dev-тулинг ---
RUN apt-get update && apt-get install -y --no-install-recommends \
  less \
  git \
  procps \
  sudo \
  fzf \
  zsh \
  man-db \
  unzip \
  gnupg2 \
  gh \
  iptables \
  ipset \
  iproute2 \
  dnsutils \
  aggregate \
  jq \
  nano \
  vim \
  curl \
  wget \
  ca-certificates \
  bash-completion \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# --- Docker CLI + compose plugin (без демона — через DOCKER_HOST на api-filter) ---
RUN apt-get update \
  && apt-get install -y --no-install-recommends ca-certificates gnupg \
  && install -m 0755 -d /etc/apt/keyrings \
  && curl -fsSL https://download.docker.com/linux/debian/gpg \
      | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
  && chmod a+r /etc/apt/keyrings/docker.gpg \
  && . /etc/os-release \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian ${VERSION_CODENAME} stable" \
      > /etc/apt/sources.list.d/docker.list \
  && apt-get update \
  && apt-get install -y --no-install-recommends docker-ce-cli docker-compose-plugin \
  && apt-get purge -y --auto-remove gnupg \
  && rm -rf /var/lib/apt/lists/*

# --- uv: единственный нужный python-инструмент ---
RUN curl -LsSf https://astral.sh/uv/install.sh \
      | UV_INSTALL_DIR=/usr/local/bin sh \
  && uv --version

# --- Конфиг uv для контейнерного сценария ---
# UV_PYTHON_INSTALL_DIR  — куда uv ставит интерпретаторы (общий для всех проектов)
# UV_CACHE_DIR           — общий кеш wheel'ов (ускоряет uv sync в любом проекте)
# UV_LINK_MODE=copy      — .venv обычно лежит на bind-mount workspace, hardlink
#                          между разными ФС не работает; copy надёжнее
# UV_COMPILE_BYTECODE=1  — компилит .pyc при установке, чуть быстрее старт
ENV UV_PYTHON_INSTALL_DIR=/opt/uv-python \
    UV_CACHE_DIR=/opt/uv-cache \
    UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1

# Предустановка Python 3.12 в образ — чтобы uv sync в проекте
# не качал интерпретатор каждый раз с нуля
RUN mkdir -p /opt/uv-python /opt/uv-cache /workspace /home/node/.claude /commandhistory \
  && touch /commandhistory/.bash_history \
  && chown -R node:node /opt/uv-python /opt/uv-cache /workspace /home/node/.claude /commandhistory \
  && sudo -u node uv python install 3.12

# --- git-delta для красивых git diff ---
ARG GIT_DELTA_VERSION=0.18.2
RUN ARCH=$(dpkg --print-architecture) \
  && wget -q "https://github.com/dandavison/delta/releases/download/${GIT_DELTA_VERSION}/git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb" \
  && dpkg -i "git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb" \
  && rm "git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb"

ENV DEVCONTAINER=true

# --- npm-global для глобальных npm пакетов от node ---
RUN mkdir -p /usr/local/share/npm-global \
  && chown -R node:node /usr/local/share/npm-global

WORKDIR /workspace
USER node

ENV NPM_CONFIG_PREFIX=/usr/local/share/npm-global \
    PATH=/usr/local/share/npm-global/bin:$PATH \
    SHELL=/bin/zsh \
    EDITOR=nano \
    VISUAL=nano

# --- zsh + oh-my-zsh ---
ARG ZSH_IN_DOCKER_VERSION=1.2.0
RUN sh -c "$(wget -O- https://github.com/deluan/zsh-in-docker/releases/download/v${ZSH_IN_DOCKER_VERSION}/zsh-in-docker.sh)" -- \
  -p git \
  -p fzf \
  -a "source /usr/share/doc/fzf/examples/key-bindings.zsh" \
  -a "source /usr/share/doc/fzf/examples/completion.zsh" \
  -a "export PROMPT_COMMAND='history -a' && export HISTFILE=/commandhistory/.bash_history" \
  -x

# --- Claude Code CLI ---
RUN npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}

# --- Firewall init script (требует CAP_NET_ADMIN на рантайме) ---
USER root
COPY init-firewall.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/init-firewall.sh \
  && echo "node ALL=(root) NOPASSWD: /usr/local/bin/init-firewall.sh" > /etc/sudoers.d/node-firewall \
  && chmod 0440 /etc/sudoers.d/node-firewall

USER node
WORKDIR /workspace