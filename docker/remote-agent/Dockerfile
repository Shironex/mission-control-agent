# Mission Control remote agent image for Railway / VM deployments.
# Build context: public mission-control-agent repo root.
FROM node:24-bookworm AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN if [ -f package-lock.json ]; then npm ci --ignore-scripts; else npm install --ignore-scripts; fi
COPY src ./src
COPY build.mjs tsconfig.json tsconfig.build.json ./
RUN npm run build

FROM node:24-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    HOME=/home/workspace \
    PATH=/home/workspace/.local/bin:$PATH
RUN apt-get update && apt-get install -y --no-install-recommends \
    bash build-essential ca-certificates curl git gnupg jq less openssh-client procps \
    python3 python3-pip python3-venv ripgrep sudo unzip xz-utils zip zsh \
  && rm -rf /var/lib/apt/lists/*
RUN usermod -l workspace -d /home/workspace -m node \
  && groupmod -n workspace node \
  && echo "workspace ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/workspace \
  && chmod 0440 /etc/sudoers.d/workspace
RUN corepack enable \
  && corepack prepare pnpm@11.1.2 --activate \
  && npm install -g @openai/codex@latest @anthropic-ai/claude-code@latest opencode-ai@latest
# The root `npm install -g` above (opencode) creates /home/workspace/.local owned
# by root. The cursor-agent installer below runs as `workspace` and writes under
# ~/.local/share/cursor-agent — pre-create and chown ~/.local first, otherwise its
# mkdir fails with EACCES, the installer ignores the error, and the tar step then
# fails ("Cannot open ... No such file or directory" / curl 23).
RUN mkdir -p /home/workspace/.local/bin /home/workspace/.local/share \
  && chown -R workspace:workspace /home/workspace/.local
USER workspace
RUN for i in 1 2 3; do \
      curl https://cursor.com/install -fsS | bash && break; \
      echo "cursor-agent install attempt $i failed; retrying in 5s..."; sleep 5; \
    done || echo "WARNING: cursor-agent install failed; continuing without it"
USER root
RUN ln -sf /home/workspace/.local/bin/cursor-agent /usr/local/bin/cursor-agent \
  && ln -sf /home/workspace/.local/bin/agent /usr/local/bin/agent || true
COPY package.json /opt/mission-control-agent/package.json
COPY --from=build /app/dist /opt/mission-control-agent/dist
RUN cd /opt/mission-control-agent \
  && npm install --omit=dev --no-package-lock \
  && npm link \
  && chown -R workspace:workspace /opt/mission-control-agent
RUN mkdir -p /workspace /home/workspace/.ssh /home/workspace/.config \
  && chown -R workspace:workspace /workspace /home/workspace \
  && chmod 700 /home/workspace/.ssh
ENV MC_WORKSPACE_ROOT=/workspace \
    MC_AGENT_BIND_HOST=0.0.0.0 \
    MC_LINK_WORKSPACE_TO_HOME=1 \
    CLAUDE_CONFIG_DIR=/home/workspace/.claude
EXPOSE 9333
WORKDIR /workspace
USER workspace
RUN { claude --version; codex --version; cursor-agent --version; opencode --version; } || true
USER root
COPY docker-entrypoint.sh /usr/local/bin/mc-entrypoint.sh
RUN chmod +x /usr/local/bin/mc-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/mc-entrypoint.sh"]
CMD ["mission-control-agent"]
