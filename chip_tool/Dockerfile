ARG BUILD_FROM
ARG BUILD_ARCH

FROM $BUILD_FROM AS builder-base

# hadolint ignore=DL3009
RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       git \
       patch \
       gcc g++ \
       make \
       pkg-config

FROM builder-base AS builder-ttyd

ARG LIBWEBSOCKETS_VERSION
ARG TTYD_VERSION

WORKDIR /usr/src

RUN \
    set -x \
    && apt-get install -y --no-install-recommends \
       cmake \
       libuv1-dev \
       libc-dev \
       libssl-dev \
       zlib1g-dev \
       libjson-c-dev \
    && git clone --branch "v${LIBWEBSOCKETS_VERSION}" --depth=1 \
        https://github.com/warmcat/libwebsockets \
    && cd libwebsockets \
    && mkdir -p build \
    && cd build \
    && cmake .. \
        -DCMAKE_BUILD_TYPE=MinSizeRel \
        -DCMAKE_INSTALL_PREFIX=/usr/local \
        -DCMAKE_VERBOSE_MAKEFILE=TRUE \
        -DLWS_IPV6=ON \
        -DLWS_STATIC_PIC=ON \
        -DLWS_UNIX_SOCK=ON \
        -DLWS_WITH_LIBUV=ON \
        -DLWS_WITH_SHARED=ON \
        -DLWS_WITHOUT_TESTAPPS=ON \
    && make \
    && make install \
    && make DESTDIR=./install install

RUN \
    cd /usr/src \
    && git clone --branch main --single-branch \
        https://github.com/tsl0922/ttyd \
    && cd ttyd \
    && git checkout "${TTYD_VERSION}" \
    && mkdir -p build \
    && cd build \
    && cmake .. \
        -DCMAKE_BUILD_TYPE=MinSizeRel \
        -DCMAKE_INSTALL_PREFIX=/usr/local \
        -DCMAKE_VERBOSE_MAKEFILE=TRUE \
    && make \
    && make DESTDIR=./install install

# Use amd64 stage to generate ZAP files (zap-cli is only available for amd64)
FROM --platform=linux/amd64 debian:bullseye AS builder-amd64-base

ARG CHIP_VERSION

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

WORKDIR /usr/src

RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       git \
       ca-certificates \
       python3 \
    && git clone --depth 1 --no-single-branch -b master \
       https://github.com/project-chip/connectedhomeip \
    && cd connectedhomeip \
    && git fetch origin ${CHIP_VERSION} \
    && git checkout ${CHIP_VERSION}

FROM $BUILD_FROM

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG BUILD_ARCH
ARG CHIP_VERSION

RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       libuv1 \
       openssl \
       zlib1g \
       libjson-c5 \
       libdbus-1-3 \
       libglib2.0 \
       libnl-3-200 \
       libnl-route-3-200 \
       tmux \
       curl \
       jq \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/src/*

COPY --from=builder-ttyd /usr/src/libwebsockets/build/install /
COPY --from=builder-ttyd /usr/src/ttyd/build/install /

RUN \
    set -x \
    && export REPO_OWNER="agners" \
    && export REPO_NAME="matter-linux-example-apps" \
    && if [ "${BUILD_ARCH}" = "amd64" ]; then \
            export EXAMPLE_ARCH="x86-64"; \
       elif [ "${BUILD_ARCH}" = "aarch64" ]; then \
            export EXAMPLE_ARCH="aarch64"; \
       else \
            exit 1; \
       fi \
    && export API_URL="https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/releases/tags/${CHIP_VERSION}" \
    && curl -s $API_URL | \
       jq -r --arg suffix "-${EXAMPLE_ARCH}" '.assets[] | select(.name | endswith($suffix)) | .browser_download_url, (.name | rtrimstr($suffix))' | \
       while read -r url; read -r name; do \
         echo "Downloading $url..."; \
         curl -Lo "/usr/local/bin/$name" "$url"; \
         chmod +x "/usr/local/bin/$name"; \
       done

RUN ldconfig

WORKDIR /root

COPY --from=builder-amd64-base /usr/src/connectedhomeip/credentials /root/credentials

COPY rootfs /
