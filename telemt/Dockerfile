FROM --platform=$BUILDPLATFORM rust:1.94.1-alpine AS builder-telemt
ARG TARGETOS
ARG TARGETARCH
ARG VERSION
WORKDIR /tmp/src
ENV PATH="/opt/x86_64-linux-musl-cross/bin:${PATH}"
ENV PATH="/opt/aarch64-linux-musl-cross/bin:${PATH}"
ENV PATH="/opt/riscv64-linux-musl-cross/bin:${PATH}"
ADD --unpack=true https://musl.cc/x86_64-linux-musl-cross.tgz /opt
ADD --unpack=true https://musl.cc/aarch64-linux-musl-cross.tgz /opt
ADD --unpack=true https://musl.cc/riscv64-linux-musl-cross.tgz /opt
ADD https://github.com/telemt/telemt.git${VERSION:+#}${VERSION} .
RUN install -m 644 /dev/stdin -D ./.cargo/config.toml <<-EOF
        [target.x86_64-unknown-linux-musl]
        linker = "x86_64-linux-musl-gcc"
        ar = "x86_64-linux-musl-ar"

        [target.aarch64-unknown-linux-musl]
        linker = "aarch64-linux-musl-gcc"
        ar = "aarch64-linux-musl-ar"

        [target.riscv64gc-unknown-linux-musl]
        linker = "riscv64-linux-musl-gcc"
        ar = "riscv64-linux-musl-ar"
EOF
RUN apk add --no-cache \
        build-base \
        yq \
    && case ${TARGETARCH} in \
        "amd64")   export TARGETARCH="x86_64" ;; \
        "arm64")   export TARGETARCH="aarch64" ;; \
        "riscv64") export TARGETARCH="riscv64gc" ;; \
        *)         echo "Unsupported architecture: ${TARGETARCH}" && exit 1 ;; \
    esac \
    && echo "export TARGETARCH=${TARGETARCH}" > ./.env \
    && rustup target add ${TARGETARCH}-unknown-linux-musl \
    && yq --input-format toml \
        '.general.proxy_secret_path = "/run/secrets/telemt"' \
        '.general.proxy_config_v4_cache_path = "/var/cache/telemt/proxy-config-v4.txt"' \
        '.general.proxy_config_v6_cache_path = "/var/cache/telemt/proxy-config-v6.txt"' \
        '.general.beobachten_file = "/var/cache/telemt/beobachten.txt"' \
        '.general.modes.classic = true' \
        '.general.modes.secure = true' \
        '.general.modes.tls = true' \
        '.server.port = 443' \
        '.server.metrics_port = 9090' \
        '.server.metrics_listen = "127.0.0.1:9091"' \
        '.server.metrics_whitelist = ["127.0.0.1/32", "::1/128"]' \
        '.server.api.enabled = true' \
        '.server.api.listen = "127.0.0.1:9091"' \
        '.server.api.whitelist = ["127.0.0.1/32", "::1/128"]' \
        ./config.toml
WORKDIR ../out-stage
RUN . ../.env \
    && cargo build --release --target ${TARGETARCH}-unknown-linux-musl \
    && mkdir -p ./usr/bin ./etc/telemt ./var/cache/telemt ./run/secrets/telemt \
    && cp -p ../src/target/${TARGETARCH}-unknown-linux-musl/release/telemt ./usr/bin/ \
    && cp -p ../src/config.toml ./etc/telemt/config.toml

FROM alpine:3.23.4 AS app
COPY --from=builder-telemt /tmp/out-stage /
RUN apk add --no-cache \
        curl \
    && addgroup -g 1000 telemt \
    && adduser -D -G telemt -u 1000 -s /bin/sh telemt \
    && chown -R telemt:telemt /var/cache/telemt /run/secrets/telemt
WORKDIR /opt/telemt
USER telemt:telemt
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --start-interval=15s --retries=3 \
  CMD curl -f http://localhost:9091/api/health || exit 1
ENTRYPOINT ["telemt"]
