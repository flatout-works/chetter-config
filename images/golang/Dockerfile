# syntax=docker/dockerfile:1.7

# Chetter Agent — Go Variant
# =========================================================================
# Go development environment. Inherits shared harnesses from
# chetter-agent-base and adds Go-specific compilers and tools.
# =========================================================================

ARG BASE_IMAGE=ghcr.io/flatout-works/chetter-agent-base:main

FROM golang:1.26-bookworm AS go-tools-builder

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go install github.com/pressly/goose/v3/cmd/goose@latest \
    && go install golang.org/x/vuln/cmd/govulncheck@latest

FROM debian:bookworm-slim AS binary-downloader

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends ca-certificates curl

ARG GO_VERSION=1.26.2
RUN curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz" | tar -C /usr/local -xzf -

ARG BUF_VERSION=v1.69.0
RUN curl -fsSL "https://github.com/bufbuild/buf/releases/download/${BUF_VERSION}/buf-$(uname -s)-$(uname -m).tar.gz" \
    | tar -C /usr/local --strip-components=1 -xzf -

ARG SQLC_VERSION=v1.31.1
RUN SQLC_ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/') \
    && curl -fsSL -o /tmp/sqlc.tar.gz \
       "https://github.com/sqlc-dev/sqlc/releases/download/${SQLC_VERSION}/sqlc_${SQLC_VERSION#v}_linux_${SQLC_ARCH}.tar.gz" \
    && tar -C /usr/local/bin -xzf /tmp/sqlc.tar.gz sqlc \
    && rm /tmp/sqlc.tar.gz && chmod +x /usr/local/bin/sqlc

ARG OSV_SCANNER_VERSION=v2.3.8
RUN curl -sL "https://github.com/google/osv-scanner/releases/download/${OSV_SCANNER_VERSION}/osv-scanner_linux_amd64" \
    -o /usr/local/bin/osv-scanner && chmod +x /usr/local/bin/osv-scanner

ARG HCLOUD_VERSION=v1.51.0
RUN HCLOUD_ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/') \
    && curl -fsSL -o /tmp/hcloud.tar.gz \
       "https://github.com/hetznercloud/cli/releases/download/${HCLOUD_VERSION}/hcloud-linux-${HCLOUD_ARCH}.tar.gz" \
    && tar -C /usr/local/bin -xzf /tmp/hcloud.tar.gz hcloud \
    && rm /tmp/hcloud.tar.gz && chmod +x /usr/local/bin/hcloud

FROM ${BASE_IMAGE}

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends default-mysql-client \
    && rm -rf /var/lib/apt/lists/*

ENV GOPATH=/go
ENV PATH="/usr/local/go/bin:/go/bin:/usr/local/bin:${PATH}"

COPY --from=binary-downloader /usr/local/go /usr/local/go
COPY --from=binary-downloader /usr/local/bin/buf /usr/local/bin/buf
COPY --from=binary-downloader /usr/local/bin/sqlc /usr/local/bin/sqlc
COPY --from=binary-downloader /usr/local/bin/osv-scanner /usr/local/bin/osv-scanner
COPY --from=binary-downloader /usr/local/bin/hcloud /usr/local/bin/hcloud
COPY --from=go-tools-builder /go/bin/goose /usr/local/bin/goose
COPY --from=go-tools-builder /go/bin/govulncheck /usr/local/bin/govulncheck

RUN go version && govulncheck -h && osv-scanner --version && hcloud version

WORKDIR /workspace
