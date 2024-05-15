FROM debian:bookworm-20231009-slim

ENV IMAGE_DATE=202403080000

ARG VIDEOLAN_UID=499
ARG EMSDK_VERSION=3.1.55
ARG CORES=8

ENV PATH=/opt/tools/bin:/opt/tools/libexec:$PATH

RUN set -x && \
    apt-get update && \
    apt-get -y dist-upgrade && \
    apt-get install -y --no-install-suggests --no-install-recommends \
        git libtool libltdl-dev automake autoconf autopoint make \
        gettext pkg-config cmake zip bzip2 p7zip-full wget dos2unix \
        ragel yasm g++ m4 ninja-build ant build-essential libtool-bin gperf \
        flex bison curl nasm meson python3 python3-venv python3-setuptools python3-mako \
        protobuf-compiler && \
    apt-get clean -y && rm -rf /var/lib/apt/lists/* && \
    addgroup --quiet --gid ${VIDEOLAN_UID} videolan && \
    adduser --quiet --uid ${VIDEOLAN_UID} --ingroup videolan videolan && \
    echo "videolan:videolan" | chpasswd && \
    mkdir /build && cd /build && \
    VLC_TOOLS_HASH=7c260ce99e870757f78cd6a2fda9212944a000ea && \
    git clone -b master https://code.videolan.org/videolan/vlc.git && cd vlc && git checkout ${VLC_TOOLS_HASH} && \
    cd contrib && mkdir build-contribs && cd build-contribs && \
    TARGET_TUPLE=wasm32-unknown-emscripten && \
    ../bootstrap --host=$TARGET_TUPLE --prefix=/opt/tools/dummy --enable-ad-clauses && \
    make -j$CORES list && \
    make -j$CORES tools && \
    rm -rf /build/*

USER videolan

WORKDIR /home/videolan

RUN git clone --depth=1 https://github.com/emscripten-core/emsdk/ && \
    cd emsdk && ./emsdk install $EMSDK_VERSION && \
    ./emsdk activate $EMSDK_VERSION

RUN git config --global user.name "VLC.js" && \
    git config --global user.email buildbot@videolan.org

ENV EMSCRIPTEN_SDK="/home/videolan/emsdk"
