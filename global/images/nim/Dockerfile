# syntax=docker/dockerfile:1.7

# Chetter Agent - Nim Variant
# =========================================================================
# Nim development environment for Nim and mixed Nim/Go repositories
# (BuddyDrive, BuddyDrive Relay, Niffler).
# =========================================================================

ARG BASE_IMAGE=ghcr.io/flatout-works/chetter-agent-base:main

FROM ${BASE_IMAGE}

ARG NIM_VERSION=2.2.8
ARG GO_VERSION=1.26.4

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    clang \
    g++ \
    libgtk-4-dev \
    libclang-dev \
    liblz4-dev \
    libmariadb-dev \
    libsqlite3-dev \
    libsodium-dev \
    libssl-dev \
    pkg-config \
    xz-utils \
    && rm -rf /var/lib/apt/lists/*

# Go toolchain: mixed Nim/Go repositories (e.g. Niffler) build Go components
# alongside the Nim core.
RUN curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz" | tar -C /usr/local -xzf -

RUN curl -fsSL https://nim-lang.org/choosenim/init.sh \
    | bash -s -- -y \
    && /opt/opencode/.nimble/bin/choosenim ${NIM_VERSION}

ENV PATH="/usr/local/go/bin:/opt/opencode/.nimble/bin:/opt/opencode/.choosenim/toolchains/nim-${NIM_VERSION}/bin:${PATH}"

RUN for tool in choosenim nim nimble nimgrep nimpretty nimsuggest testament; do \
      ln -sf "/opt/opencode/.nimble/bin/${tool}" "/usr/local/bin/${tool}"; \
    done \
    && nim --version \
    && nimble --version \
    && go version \
    && command -v testament

WORKDIR /workspace
