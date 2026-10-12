FROM debian:trixie-slim@sha256:a99cfc517144bc59b1978475ec53b46ecabec7e43635402ee5b77cc54cd1b20a AS glyphs

ARG FONT_MAKER_REF=c56771948c59a01b89d0498d47dc4abf56c25338

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    bash \
    build-essential \
    ca-certificates \
    clang \
    cmake \
    curl \
    fonts-noto-core \
    git \
    libboost-dev \
    libfreetype6-dev \
    pkg-config \
    zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /tmp

RUN git clone https://github.com/maplibre/font-maker.git /tmp/font-maker \
    && cd /tmp/font-maker \
    && git checkout "$FONT_MAKER_REF" \
    && git submodule update --init --recursive \
    && cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
    && cmake --build build --parallel

WORKDIR /glyphs

COPY tiles/glyphs/generate-glyphs.sh /glyphs/generate-glyphs.sh

RUN chmod +x /glyphs/generate-glyphs.sh \
    && /glyphs/generate-glyphs.sh /glyphs/output /tmp/font-maker/build/font-maker

# Martin's release musl builds are statically linked and include every feature
# this image uses (mbtiles, sprites, fonts, styles, metrics). Rendering is not
# part of the musl build and is not used here. The download runs on the build
# platform so cross-arch builds need no emulation.
FROM --platform=$BUILDPLATFORM nginxinc/nginx-unprivileged:1.30.5-alpine-slim@sha256:7f79f463c6eb0d0bf8782992bb9636434398e3e5848398805aef2c9b025d8609 AS martin

ARG TARGETARCH
# renovate: datasource=github-releases depName=maplibre/martin extractVersion=^martin-v(?<version>.*)$
ARG MARTIN_VERSION=1.16.1

# Build-only stage; the runtime stage switches back to 101.
# hadolint ignore=DL3002
USER 0

WORKDIR /tmp/martin

# hadolint ignore=DL3018
RUN apk add --no-cache jq

# The expected SHA256 comes from GitHub's release asset metadata, so a
# Renovate version bump needs no manual checksum update. Pass a token with
# `--secret id=github_token,env=GITHUB_TOKEN` to avoid API rate limits.
RUN --mount=type=secret,id=github_token,required=false \
    case "${TARGETARCH}" in \
    amd64) arch=x86_64 ;; \
    arm64) arch=aarch64 ;; \
    *) echo "Unsupported TARGETARCH: ${TARGETARCH}" >&2; exit 1 ;; \
    esac \
    && asset="martin-${arch}-unknown-linux-musl.tar.gz" \
    && if [ -s /run/secrets/github_token ]; then \
    set -- --header "Authorization: Bearer $(cat /run/secrets/github_token)"; \
    else set --; fi \
    && wget -q -O release.json "$@" \
    "https://api.github.com/repos/maplibre/martin/releases/tags/martin-v${MARTIN_VERSION}" \
    && sha256="$(jq -er --arg name "${asset}" \
    '.assets[] | select(.name == $name) | .digest | select(startswith("sha256:")) | ltrimstr("sha256:")' \
    release.json)" \
    && wget -q -O martin.tar.gz \
    "https://github.com/maplibre/martin/releases/download/martin-v${MARTIN_VERSION}/${asset}" \
    && printf '%s  %s\n' "${sha256}" martin.tar.gz > martin.tar.gz.sha256 \
    && sha256sum -c martin.tar.gz.sha256 \
    && tar -xzf martin.tar.gz martin

FROM nginxinc/nginx-unprivileged:1.30.5-alpine-slim@sha256:7f79f463c6eb0d0bf8782992bb9636434398e3e5848398805aef2c9b025d8609 AS base

USER 0

# hadolint ignore=DL3018
RUN apk add --no-cache bash \
    && rm -rf /etc/nginx/conf.d /usr/share/nginx/html /docker-entrypoint.sh /docker-entrypoint.d

COPY --from=martin /tmp/martin/martin /usr/local/bin/martin
COPY --chown=101:101 styles /styles
COPY --from=glyphs --chown=101:101 /glyphs/output /styles/fonts
COPY --chown=101:101 tiles/nginx.conf /etc/nginx/nginx.conf
COPY --chmod=755 --chown=101:101 tiles/entrypoint.sh /entrypoint.sh

USER 101:101

# 8080 serves tiles and styles. 9090 serves Prometheus metrics at /metrics
# and should stay internal.
EXPOSE 8080 9090

# The nginx base image uses SIGQUIT; entrypoint.sh handles SIGTERM/SIGINT.
STOPSIGNAL SIGTERM

ENTRYPOINT ["/entrypoint.sh"]

# Production image: MBTiles baked in (built by publish-image.yml).
FROM base AS bundle

COPY --chown=101:101 bundle/lux-railway-map-overlay.mbtiles /app/data/lux-railway-map-overlay.mbtiles

# Default target: no data baked in; mount MBTiles at runtime (docker-compose.yml).
FROM base AS runtime
