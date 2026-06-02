# Copyright IBM Corp. 2016, 2025
# SPDX-License-Identifier: BUSL-1.1

# Multi-stage builder to avoid polluting users environment with wrong
# architecture binaries.
ARG VERSION

FROM golang:${VERSION} AS builder

ARG CGO_ENABLED=0
ARG BUILD_TAGS

WORKDIR /go/src/github.com/hashicorp/vault
COPY . .

RUN git init -q && \
    git config user.email "build@example.com" && \
    git config user.name "Build" && \
    git commit -q --allow-empty -m "build"

RUN make bootstrap \
  && CGO_ENABLED=$CGO_ENABLED BUILD_TAGS="${BUILD_TAGS}" VAULT_DEV_BUILD=1 sh -c "'./scripts/build.sh'"

# Docker Image

FROM alpine:3

# Create a vault user and group first so the IDs get set the same way,
# even as the rest of this may change over time.
RUN addgroup vault && \
    adduser -S -G vault vault

# Set up certificates, our base tools, and Vault.
# apk add --upgrade ensures latest patched versions of each package.
RUN apk update && apk add --upgrade --no-cache ca-certificates libcap su-exec dumb-init tzdata

COPY --from=builder /go/src/github.com/hashicorp/vault/bin/vault /bin/vault

# /vault/logs is made available to use as a location to store audit logs, if
# desired; /vault/file is made available to use as a location with the file
# storage backend, if desired; the server will be started with /vault/config as
# the configuration directory so you can add additional config files in that
# location.
RUN mkdir -p /vault/logs && \
    mkdir -p /vault/file && \
    mkdir -p /vault/config && \
    chown -R vault:vault /vault

# Expose the logs directory as a volume since there's potentially long-running
# state in there
VOLUME /vault/logs

# Expose the file directory as a volume since there's potentially long-running
# state in there
VOLUME /vault/file

# 8200/tcp is the primary interface that applications use to interact with
# Vault.
EXPOSE 8200

# The entry point script uses dumb-init as the top-level process to reap any
# zombie processes created by Vault sub-processes.
#
# For production derivatives of this container, you should add the IPC_LOCK
# capability so that Vault can mlock memory.
COPY ./scripts/docker/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]

# CIS Docker 4.6 - add a HEALTHCHECK so the container reports its own health.
# Uses wget (available in Alpine busybox) to probe the Vault health endpoint.
# sealedcode/uninitcode=204 so the check passes even before unseal/init.
HEALTHCHECK --interval=10s --timeout=3s --start-period=30s --retries=3 \
    CMD wget -qO- http://localhost:8200/v1/sys/health?standbyok=true\&sealedcode=204\&uninitcode=204 || exit 1

# Use the Vault user as the default user for starting this container.
USER vault

# By default you'll get a single-node development server that stores everything
# in RAM and bootstraps itself. Don't use this configuration for production.
CMD ["server", "-dev"]
