FROM node:20-bullseye-slim

RUN apt-get update && apt-get install -y \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g @adobe/aio-cli && \
    npm cache clean --force && \
    rm -rf /root/.npm /tmp/*

WORKDIR /workspace
