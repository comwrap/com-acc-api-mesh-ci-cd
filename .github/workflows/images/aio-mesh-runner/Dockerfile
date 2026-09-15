FROM node:20-bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# oclif resolves installed plugins under $XDG_DATA_HOME, falling back to
# $HOME/.local/share. GitHub container jobs override HOME (/github/home), so a
# plugin installed here under /root would be invisible at run time — which is why
# the plugin used to be installed on every workflow run instead. Pinning
# XDG_DATA_HOME to an absolute path makes the baked plugin resolve under any $HOME.
ENV XDG_DATA_HOME=/usr/local/share

# The HOME=... smoke check is the point of this layer: it reproduces the exact
# failure mode the runtime install was working around, so the build fails if the
# plugin ever stops resolving independently of $HOME.
RUN npm install -g @adobe/aio-cli \
    && aio plugins:install @adobe/aio-cli-plugin-api-mesh \
    && mkdir -p /tmp/homecheck \
    && HOME=/tmp/homecheck aio api-mesh --help > /dev/null \
    && npm cache clean --force \
    && rm -rf /root/.npm /tmp/*

WORKDIR /workspace
