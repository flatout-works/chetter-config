# syntax=docker/dockerfile:1.7

# Chetter Agent — Python Variant
# =========================================================================
# Python development environment. Inherits shared harnesses from
# chetter-agent-base and adds Python-specific tooling.
# =========================================================================

ARG BASE_IMAGE=ghcr.io/flatout-works/chetter-agent-base:main

FROM ${BASE_IMAGE}

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    python3-venv \
    && rm -rf /var/lib/apt/lists/* \
    && ln -sf /usr/bin/python3 /usr/local/bin/python

RUN pip3 install --no-cache-dir --break-system-packages \
    ruff \
    mypy \
    pytest \
    black \
    httpx

WORKDIR /workspace
