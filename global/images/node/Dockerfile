# syntax=docker/dockerfile:1.7

# Chetter Agent — Node.js Variant
# =========================================================================
# Node.js development environment. Inherits shared harnesses from
# chetter-agent-base and adds JS/TS tooling.
# =========================================================================

ARG BASE_IMAGE=ghcr.io/flatout-works/chetter-agent-base:main

FROM ${BASE_IMAGE}

RUN --mount=type=cache,target=/root/.npm \
    npm install -g pnpm typescript ts-node eslint prettier

WORKDIR /workspace
