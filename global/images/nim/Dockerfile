# syntax=docker/dockerfile:1.7

# Chetter Agent - Nim Variant
# =========================================================================
# Nim development environment for BuddyDrive and BuddyDrive Relay.
# =========================================================================

ARG BASE_IMAGE=ghcr.io/flatout-works/chetter-agent-base:main

FROM ${BASE_IMAGE}

ARG NIM_VERSION=2.2.8

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

RUN curl -fsSL https://nim-lang.org/choosenim/init.sh \
    | bash -s -- -y \
    && /opt/opencode/.nimble/bin/choosenim ${NIM_VERSION}

ENV PATH="/opt/opencode/.nimble/bin:/opt/opencode/.choosenim/toolchains/nim-${NIM_VERSION}/bin:${PATH}"

RUN for tool in choosenim nim nimble nimgrep nimpretty nimsuggest testament; do \
      ln -sf "/opt/opencode/.nimble/bin/${tool}" "/usr/local/bin/${tool}"; \
    done \
    && nim --version \
    && nimble --version \
    && command -v testament

WORKDIR /workspace
