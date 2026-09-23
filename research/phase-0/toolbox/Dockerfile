# syntax=docker/dockerfile:1
#
# Phase 0 research toolbox (task T1).
# Static analysis only: nothing in this image executes the binaries it analyzes.
#
# Every version and checksum below is pinned on purpose. Values marked TODO(T1)
# must be filled from the upstream release pages before the first build; the
# build fails while any of them is still TODO. Record the chosen versions in the
# journal and in the run log (plan §12.2).

ARG DEBIAN_IMAGE=debian:trixie-slim@sha256:TODO_T1
ARG FEDORA_IMAGE=registry.fedoraproject.org/fedora:TODO_T1@sha256:TODO_T1

# --- Stage 1: hardening-check comes from Debian devscripts --------------------
FROM ${DEBIAN_IMAGE} AS debian-tools
RUN apt-get update \
 && apt-get install -y --no-install-recommends devscripts \
 && cp /usr/bin/hardening-check /hardening-check \
 && rm -rf /var/lib/apt/lists/*

# --- Stage 2: the toolbox, on Fedora for annocheck ----------------------------
FROM ${FEDORA_IMAGE}

# binutils: readelf · pax-utils: scanelf, lddtree · annobin-annocheck: annocheck
# perl-interpreter: runs hardening-check · python3: collection scripts
RUN dnf -y install --setopt=install_weakdeps=False \
      binutils pax-utils annobin-annocheck perl-interpreter python3 \
      tar gzip ca-certificates \
 && dnf clean all

COPY --from=debian-tools /hardening-check /usr/local/bin/hardening-check

# Release-pinned binaries: version, download URL, and SHA-256 of the archive.
ARG CHECKSEC_URL=TODO_T1
ARG CHECKSEC_SHA256=TODO_T1
ARG SYFT_VERSION=TODO_T1
ARG SYFT_SHA256=TODO_T1
ARG CRANE_VERSION=TODO_T1
ARG CRANE_SHA256=TODO_T1

RUN set -eu; \
    for v in "$CHECKSEC_URL" "$CHECKSEC_SHA256" "$SYFT_VERSION" "$SYFT_SHA256" "$CRANE_VERSION" "$CRANE_SHA256"; do \
      case "$v" in *TODO_T1*) echo "Unpinned tool version or checksum: fill every TODO_T1 build argument" >&2; exit 1;; esac; \
    done; \
    fetch() { curl -fsSL --proto '=https' --tlsv1.2 -o "$1" "$2"; echo "$3  $1" | sha256sum -c -; }; \
    fetch /tmp/syft.tgz \
      "https://github.com/anchore/syft/releases/download/v${SYFT_VERSION}/syft_${SYFT_VERSION}_linux_amd64.tar.gz" \
      "$SYFT_SHA256"; \
    tar -xzf /tmp/syft.tgz -C /usr/local/bin syft; \
    fetch /tmp/gcr.tgz \
      "https://github.com/google/go-containerregistry/releases/download/v${CRANE_VERSION}/go-containerregistry_Linux_x86_64.tar.gz" \
      "$CRANE_SHA256"; \
    tar -xzf /tmp/gcr.tgz -C /usr/local/bin crane; \
    fetch /tmp/checksec.tgz "$CHECKSEC_URL" "$CHECKSEC_SHA256"; \
    mkdir /tmp/checksec && tar -xzf /tmp/checksec.tgz -C /tmp/checksec; \
    install -m 0755 "$(find /tmp/checksec -type f -name checksec | head -n1)" /usr/local/bin/checksec; \
    rm -rf /tmp/*.tgz /tmp/checksec

# Record tool versions inside the image; collection scripts copy this into run.json.
RUN { \
      echo "readelf=$(readelf --version | head -n1)"; \
      echo "scanelf=$(scanelf --version 2>&1 | head -n1)"; \
      echo "annocheck=$(annocheck --version 2>&1 | head -n1)"; \
      echo "checksec=$(checksec --version 2>&1 | head -n1)"; \
      echo "syft=$(syft version 2>&1 | grep -i '^Version' | head -n1)"; \
      echo "crane=$(crane version 2>&1 | head -n1)"; \
    } > /etc/research-toolbox-versions

# Batch tool, not a long-running service: there is nothing to health-check.
HEALTHCHECK NONE

RUN useradd --create-home --uid 10001 researcher
USER researcher
WORKDIR /home/researcher
