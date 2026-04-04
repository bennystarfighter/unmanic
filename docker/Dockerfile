# =============================================================================
# Stage 1 — system-base
# All runtime APT packages: drivers, FFmpeg, Node.js, Python runtime
# =============================================================================
FROM ubuntu:24.04 AS system-base
LABEL maintainer="Josh.5 <jsunnex@gmail.com>"

ARG DEBIAN_FRONTEND=noninteractive
ARG JELLYFIN_FFMPEG_VERSION="7"
ARG BTBN_FFMPEG_RELEASE="latest"
ARG BTBN_FFMPEG_VERSION="8.1"

ENV \
    LIBVA_DRIVERS_PATH="/usr/lib/x86_64-linux-gnu/dri" \
    NVIDIA_DRIVER_CAPABILITIES="compute,video,utility" \
    NVIDIA_VISIBLE_DEVICES="all"

# Base runtime libraries and tools.
# va-driver-all is a meta-package covering all available VAAPI drivers
# (i965-va-driver, intel-media-va-driver, mesa-va-drivers, libdrm2, etc.)
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        git \
        gnupg \
        gosu \
        grc \
        libexpat1 \
        libglib2.0-0 \
        libgomp1 \
        libharfbuzz0b \
        libimage-exiftool-perl \
        libmediainfo0v5 \
        libv4l-0 \
        libvulkan1 \
        libx11-6 \
        libxcb1 \
        libxext6 \
        libxml2 \
        mesa-vulkan-drivers \
        python3 \
        python3-venv \
        sqlite3 \
        va-driver-all \
        vainfo \
        wget \
        xz-utils \
    && rm -rf /var/lib/apt/lists/*

# x86 only: Intel GPU drivers and media libraries.
#   intel-media-va-driver-non-free  primary VAAPI driver for Intel graphics
#   libigdgmm12                     Intel Graphics Memory Management Library
#   libmfx1 / libmfxgen1            Intel Media SDK runtime (Quick Sync Video)
#   libva-drm2 / libva2             VAAPI/DRM interface libraries
#   libvpl2                         successor to Intel Media SDK
RUN \
    if uname -m | grep -q x86; then \
        wget -qO - https://repositories.intel.com/gpu/intel-graphics.key \
            | gpg --dearmor -o /usr/share/keyrings/intel-graphics.gpg \
        && echo "deb [arch=amd64,i386 signed-by=/usr/share/keyrings/intel-graphics.gpg] \
                https://repositories.intel.com/gpu/ubuntu jammy client" \
            > /etc/apt/sources.list.d/intel-gpu-jammy.list \
        && apt-get update \
        && apt-get install -y \
            intel-media-va-driver-non-free \
            libigdgmm12 \
            libmfx1 \
            libmfxgen1 \
            libva-drm2 \
            libva2 \
            libvpl2 \
        && rm -rf /var/lib/apt/lists/* ; \
    fi

# Jellyfin FFmpeg: add repository
RUN . /etc/os-release \
    && JELLYFIN_CODENAME="${VERSION_CODENAME:-noble}" \
    && wget -qO - https://repo.jellyfin.org/jellyfin_team.gpg.key \
        | gpg --dearmor -o /usr/share/keyrings/jellyfin_team.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/jellyfin_team.gpg] \
            https://repo.jellyfin.org/ubuntu ${JELLYFIN_CODENAME} main" \
        > /etc/apt/sources.list.d/jellyfin.list

# Jellyfin FFmpeg: install hardware-accelerated build with arch prerequisites.
# aarch64 additionally needs OpenMAX and font libraries.
RUN apt-get update \
    && if uname -m | grep -q x86; then \
        apt-get install --no-install-recommends -y \
            locales \
            openssl ; \
    elif uname -m | grep -q aarch64; then \
        apt-get install --no-install-recommends -y \
            libfontconfig1 \
            libfreetype6 \
            libomxil-bellagio-bin \
            libomxil-bellagio0 \
            libssl-dev \
            locales ; \
    fi \
    && apt-get install -y jellyfin-ffmpeg${JELLYFIN_FFMPEG_VERSION} \
    && ln -s /usr/lib/jellyfin-ffmpeg/ffmpeg /usr/local/bin/ffmpeg \
    && ln -s /usr/lib/jellyfin-ffmpeg/ffprobe /usr/local/bin/ffprobe \
    && rm -rf /var/lib/apt/lists/*

# BtbN FFmpeg: GPL static build for broader codec coverage
RUN \
    if uname -m | grep -q x86; then \
        BTBN_ARCH="linux64" ; \
    else \
        BTBN_ARCH="linuxarm64" ; \
    fi \
    && BTBN_ASSET_URL="$(curl -fsSL \
            "https://api.github.com/repos/BtbN/FFmpeg-Builds/releases/tags/${BTBN_FFMPEG_RELEASE}" \
        | grep -oE "https://[^[:space:]\"]*ffmpeg-n${BTBN_FFMPEG_VERSION}[^[:space:]\"]*-${BTBN_ARCH}-gpl-${BTBN_FFMPEG_VERSION}\.tar\.xz" \
        | head -n1)" \
    && { \
        test -n "${BTBN_ASSET_URL}" \
        || { echo "ERROR: BtbN FFmpeg asset URL not found for release ${BTBN_FFMPEG_RELEASE}"; exit 1; } ; \
    } \
    && mkdir -p /usr/lib/btbn-ffmpeg \
    && wget -qO /tmp/btbn-ffmpeg.tar.xz "${BTBN_ASSET_URL}" \
    && tar -xJf /tmp/btbn-ffmpeg.tar.xz -C /usr/lib/btbn-ffmpeg --strip-components=1 \
    && rm -f /tmp/btbn-ffmpeg.tar.xz

# Remove packages only needed while assembling the image.
RUN apt-get update \
    && apt-get purge -y --auto-remove \
        gnupg \
        wget \
        xz-utils \
    && rm -rf /var/lib/apt/lists/*


# =============================================================================
# Stage 2 — python-builder
# Build the Python virtualenv and install Unmanic from source
# =============================================================================
FROM ubuntu:24.04 AS python-builder

ARG DEBIAN_FRONTEND=noninteractive

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

ENV \
    VIRTUAL_ENV=/opt/venv \
    PATH="/opt/venv/bin:${PATH}"

# Build dependencies and virtualenv (cached unless this block changes)
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        gcc \
        git \
        nodejs \
        npm \
        python3 \
        python3-dev \
        python3-setuptools \
        python3-venv \
        unzip \
    && rm -rf /var/lib/apt/lists/* \
    && uv venv /opt/venv --python python3

# Resolve and install locked runtime dependencies before copying the full source
# tree so this layer stays cached unless dependency metadata changes.
WORKDIR /build/unmanic
COPY pyproject.toml uv.lock README.md ./
RUN uv sync --active --frozen --no-dev --no-install-project

# Install the project itself from source using the existing lockfile.
COPY . /build/unmanic/
RUN uv sync --active --frozen --no-dev \
    && rm -rf /build/


# =============================================================================
# Stage 3 — final
# =============================================================================
FROM system-base

COPY --from=python-builder /opt/venv /opt/venv

RUN mkdir -p /app /config /library /tmp/unmanic \
    && chown 1000:1000 /app /config /tmp/unmanic

ENV \
    HOME_DIR="/config" \
    VIRTUAL_ENV=/opt/venv

COPY docker/root /

ENTRYPOINT ["/entrypoint.sh"]
CMD ["/usr/bin/unmanic"]

EXPOSE 8888/tcp
