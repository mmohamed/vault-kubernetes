# Base dockerfile : https://github.com/hashicorp/vault-k8s/blob/v0.8.0/build/docker/Release.dockerfile
# At 02/2021 only amd64 arch are distributed by hashicorp
FROM alpine:latest

RUN addgroup vault && \
    adduser -S -G vault vault

RUN set -eux && \
    mkdir -p /tmp/build

COPY bin/ /tmp/build

RUN set -eux && \
    apk add --no-cache ca-certificates libcap su-exec iputils && \
    mkdir -p /tmp/build && \
    cd /tmp/build && \
    apkArch="$(apk --print-arch)" && \
    case "${apkArch}" in \
        aarch64) ARCH='arm64' ;; \
        x86_64) ARCH='amd64' ;; \
        *) echo >&2 "error: unsupported architecture: ${apkArch}" && exit 1 ;; \
    esac && \
    mv /tmp/build/vault-k8s_linux_${ARCH}_0.8.0 /bin/vault-k8s && \
    rm -rf /tmp/build

USER vault

ENTRYPOINT ["/bin/vault-k8s"]