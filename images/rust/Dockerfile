# syntax=docker/dockerfile:1.7

# Chetter Agent — Rust Variant
# =========================================================================
# Rust development environment. Inherits shared harnesses from
# chetter-agent-base and adds Rust-specific tooling.
# =========================================================================

ARG BASE_IMAGE=ghcr.io/flatout-works/chetter-agent-base:main

FROM ${BASE_IMAGE}

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/opt/opencode/.cargo/bin:${PATH}"
RUN rustup component add clippy rustfmt && cargo install cargo-audit

WORKDIR /workspace
