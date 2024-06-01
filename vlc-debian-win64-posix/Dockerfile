FROM debian:bullseye-20231030-slim

# If someone wants to use VideoLAN docker images on a local machine and does
# not want to be disturbed by the videolan user, we should not take an uid/gid
# in the user range of main distributions, which means:
# - Debian based: <1000
# - RPM based: <500 (CentOS, RedHat, etc.)
ARG VIDEOLAN_UID=499
# 8 cores ought to be enough for anybody
ARG CORES=8

ENV IMAGE_DATE=202401260000

ENV TARGET_TUPLE=x86_64-w64-mingw32
ENV PATH=/opt/cmake/bin:/opt/tools/bin:/opt/tools/libexec:$PATH

# The wine SDK path differs from version to version, starting from the one in buster, it's
# located in /usr/include/wine/wine/windows/ instead of
# /usr/include/wine/windows/
ENV WINE_SDK_PATH=/usr/include/wine/wine/windows

COPY scripts/wait_process.sh /opt/wine/

# We still need an i686 cross compiler to build the installer
RUN set -x && \
    addgroup --quiet --gid ${VIDEOLAN_UID} videolan && \
    adduser --quiet --uid ${VIDEOLAN_UID} --ingroup videolan videolan && \
    echo "videolan:videolan" | chpasswd && \
    apt-get update -qq && apt-get install -qqy --no-install-suggests --no-install-recommends \
        git wget bzip2 file libwine-dev unzip libtool libtool-bin libltdl-dev pkg-config ant \
        build-essential automake texinfo yasm p7zip-full autopoint libgl-dev libegl-dev \
        gettext dos2unix zip wine nsis g++-mingw-w64-i686 g++-mingw-w64-x86-64-posix mingw-w64-tools curl gperf flex bison \
        libcurl4-gnutls-dev python3 python3-venv python3-setuptools python3-mako python3-requests \
        lftp gcc make procps ca-certificates \
        openjdk-11-jdk-headless locales nasm jq gnupg \
    && \
    localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8 && \
    dpkg --add-architecture i386 && \
    apt-get update && apt-get -y install --no-install-suggests --no-install-recommends wine32 && \
    echo "deb http://deb.debian.org/debian testing main" > /etc/apt/sources.list.d/testing.list && \
    apt-get update && apt-get -y -t testing --no-install-suggests --no-install-recommends install \
        nasm meson autoconf autoconf2.69 && \
    rm -f /etc/apt/sources.list.d/testing.list && \
    wget -nc https://dl.winehq.org/wine-builds/winehq.key && \
    apt-key add winehq.key && rm -f winehq.key && \
    echo "deb https://dl.winehq.org/wine-builds/debian/ bullseye main" > /etc/apt/sources.list.d/winehq.list && \
    apt-get update && apt-get -y install winehq-stable && \
    rm -f /etc/apt/sources.list.d/winehq.list && \
    apt-get clean -y && rm -rf /var/lib/apt/lists/* && \
    update-alternatives --install /usr/bin/python python /usr/bin/python3.9 1 && \
    mkdir /build/ && \
    \
    chmod +x /opt/wine/wait_process.sh && \
    wget -q https://raw.githubusercontent.com/Winetricks/winetricks/5028127369b7b23b03789aec083ba5b68aa1ec34/src/winetricks -O /opt/wine/winetricks && \
    WINETRICKS_SHA256=7ec9c3e9140e592ef575140417f27e75e16ccebfeee9816cdbab21aa3d5c0841 && \
    echo $WINETRICKS_SHA256 /opt/wine/winetricks | sha256sum -c && \
    chmod +x /opt/wine/winetricks && \
    WINE_MONO_VERSION=4.5.6 && mkdir -p /usr/share/wine/mono && \
    wget -q "https://download.videolan.org/contrib/wine-mono/wine-mono-$WINE_MONO_VERSION.msi" -O /usr/share/wine/mono/wine-mono-$WINE_MONO_VERSION.msi && \
    WINE_MONO_SHA256=ac681f737f83742d786706529eb85f4bc8d6bdddd8dcdfa9e2e336b71973bc25 && \
    echo $WINE_MONO_SHA256 /usr/share/wine/mono/wine-mono-$WINE_MONO_VERSION.msi | sha256sum -c && \
    chmod +x /usr/share/wine/mono/wine-mono-$WINE_MONO_VERSION.msi && \
    cd /build && \
    CMAKE_VERSION=3.24.2 && \
    CMAKE_SHA512=6f0e8e29bf0336f555ba72c4d83f35d820f8a5159cc999d48795dc57a6627b4ee3966dda84ca97d39906e35dd476ea00cf80023672cc0fad862e2996194c0674 && \
    wget -q http://www.cmake.org/files/v3.24/cmake-$CMAKE_VERSION.tar.gz && \
    echo $CMAKE_SHA512 cmake-$CMAKE_VERSION.tar.gz | sha512sum -c && \
    tar xzf cmake-$CMAKE_VERSION.tar.gz && \
    cd cmake-$CMAKE_VERSION && ./configure --prefix=/opt/cmake/ --parallel=$CORES --no-qt-gui -- \
        -DCMAKE_USE_OPENSSL:BOOL=OFF -DBUILD_TESTING:BOOL=OFF && make -j$CORES && make install && \
    cd /build && \
    VLC_TOOLS_HASH=9e089f4edad0ed726dd18e37ea209e360d0e44c9 && \
    git clone -b master https://code.videolan.org/videolan/vlc.git && cd vlc && git checkout ${VLC_TOOLS_HASH} && \
    cd contrib && mkdir build-contribs && cd build-contribs && \
    TARGET_TUPLE=x86_64-w64-mingw32 && \
    ../bootstrap --host=$TARGET_TUPLE --prefix=/opt/tools/dummy --enable-ad-clauses && \
    make -j$CORES list && \
    make -j$CORES tools && \
    cd / && rm -rf /build

RUN git config --global user.name "VideoLAN Buildbot" && \
    git config --global user.email buildbot@videolan.org

ENV LANG=en_US.UTF-8
USER videolan
RUN wine wineboot --init && \
    /opt/wine/wait_process.sh wineserver && \
    /opt/wine/winetricks --unattended dotnet48 dotnet_verifier && \
    rm -rf ~/.cache/winetricks && \
    rm -rf /tmp/.wine-*
