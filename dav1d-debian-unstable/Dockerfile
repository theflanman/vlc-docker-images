FROM debian:sid-20240330-slim

MAINTAINER VideoLAN roots <roots@videolan.org>

ENV IMAGE_DATE=202205141200

# If someone wants to use VideoLAN docker images on a local machine and does
# not want to be disturbed by the videolan user, we should not take an uid/gid
# in the user range of main distributions, which means:
# - Debian based: <1000
# - RPM based: <500 (CentOS, RedHat, etc.)
ARG VIDEOLAN_UID=499

RUN groupadd --gid ${VIDEOLAN_UID} videolan && \
    useradd --uid ${VIDEOLAN_UID} --gid videolan --create-home --shell /bin/bash videolan && \
    echo "videolan:videolan" | chpasswd && \
    apt-get update && \
    apt-get -y dist-upgrade && \
    apt-get install --no-install-suggests --no-install-recommends -y \
        lftp ca-certificates curl git build-essential \
        nasm clang-18 libclang-rt-18-dev mold meson ninja-build gcovr \
        wine wine64 procps doxygen graphviz libsdl2-dev ripgrep \
        zstd && \
    ln -s clang-18 /usr/bin/clang && \
    ln -s clang++-18 /usr/bin/clang++ && \
    dpkg --add-architecture i386 && \
    apt-get update && \
    apt-get install --no-install-suggests --no-install-recommends -y \
        gcc-mingw-w64-x86-64 g++-mingw-w64-x86-64 \
        gcc-mingw-w64-i686 g++-mingw-w64-i686 mingw-w64-tools \
        gcc-multilib g++-multilib \
        libc6-dev:i386 libgcc-13-dev:i386 wine32 qemu-user && \
    apt-get install --no-install-suggests --no-install-recommends -y \
        gcc-riscv64-linux-gnu g++-riscv64-linux-gnu libc6-dev-riscv64-cross && \
    apt-get install --no-install-suggests --no-install-recommends -y \
        gcc-aarch64-linux-gnu g++-aarch64-linux-gnu libc6-dev-arm64-cross && \
    apt-get clean -y && rm -rf /var/lib/apt/lists/*

RUN LOONG_TOOLCHAIN_NAME="CLFS-loongarch64-8.1-x86_64-cross-tools-gcc-glibc.tar.xz" && \
    LOONG_TOOLCHAIN_SHA256="cd7c98499e1d7476df144cca22ade2140d3f311be0b4204591bdf8466971ba27" && \
    curl --location --remote-name "https://github.com/loongson/build-tools/releases/download/2023.08.08/$LOONG_TOOLCHAIN_NAME" && \
    echo "$LOONG_TOOLCHAIN_SHA256 $LOONG_TOOLCHAIN_NAME" | sha256sum -c && \
    tar -xf "$LOONG_TOOLCHAIN_NAME" -C /opt && \
    rm "$LOONG_TOOLCHAIN_NAME"

USER videolan

COPY scripts/wait_process.sh /opt/wine/
RUN wine wineboot --init && \
    /opt/wine/wait_process.sh wineserver && \
    rm -rf /tmp/wine-*

ENV PATH="/opt/cross-tools/bin:$PATH"
