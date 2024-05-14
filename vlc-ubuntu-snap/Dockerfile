# this docker file use the same struture as
# https://github.com/canonical/snapcraft/blob/main/docker/Dockerfile

ARG RISK=8.x/stable
ARG UBUNTU=jammy
ARG KF5VERSION=kf5-5-108-qt-5-15-10-core22

FROM ubuntu:$UBUNTU as builder
ARG RISK
ARG UBUNTU
ARG KF5VERSION
RUN echo "Building snapcraft:$RISK in ubuntu:$UBUNTU"

# Grab dependencies
RUN apt-get update
RUN apt-get dist-upgrade --yes
RUN apt-get install --yes \
      curl \
      jq \
      unzip \
      squashfs-tools

# Grab the core snap (for backwards compatibility) from the stable channel and
# unpack it in the proper place.
RUN curl -L $(curl -H 'X-Ubuntu-Series: 16' 'https://api.snapcraft.io/api/v1/snaps/details/core' | jq '.download_url' -r) --output core.snap
RUN mkdir -p /snap/core
RUN unsquashfs -d /snap/core/current core.snap

# Grab the core20 snap (which snapcraft uses as a base) from the stable channel
# and unpack it in the proper place.
RUN curl -L $(curl -H 'X-Ubuntu-Series: 16' 'https://api.snapcraft.io/api/v1/snaps/details/core20' | jq '.download_url' -r) --output core20.snap
RUN mkdir -p /snap/core20
RUN unsquashfs -d /snap/core20/current core20.snap


# Grab the core22 snap (which snapcraft uses as a base) from the stable channel
# and unpack it in the proper place.
RUN curl -L $(curl -H 'X-Ubuntu-Series: 16' 'https://api.snapcraft.io/api/v1/snaps/details/core22' | jq '.download_url' -r) --output core22.snap
RUN mkdir -p /snap/core22
RUN unsquashfs -d /snap/core22/current core22.snap

# Grab the snapcraft snap from the $RISK channel and unpack it in the proper
# place.
RUN curl -L $(curl -H 'X-Ubuntu-Series: 16' 'https://api.snapcraft.io/api/v1/snaps/details/snapcraft?channel='$RISK | jq '.download_url' -r) --output snapcraft.snap
RUN mkdir -p /snap/snapcraft
RUN unsquashfs -d /snap/snapcraft/current snapcraft.snap

# Fix Python3 installation: Make sure we use the interpreter from
# the snapcraft snap:
RUN unlink /snap/snapcraft/current/usr/bin/python3
RUN ln -s /snap/snapcraft/current/usr/bin/python3.* /snap/snapcraft/current/usr/bin/python3
RUN echo /snap/snapcraft/current/lib/python3.*/site-packages >> /snap/snapcraft/current/usr/lib/python3/dist-packages/site-packages.pth

# Create a snapcraft runner (TODO: move version detection to the core of
# snapcraft).
RUN mkdir -p /snap/bin
RUN echo "#!/bin/sh" > /snap/bin/snapcraft
RUN snap_version="$(awk '/^version:/{print $2}' /snap/snapcraft/current/meta/snap.yaml | tr -d \')" && echo "export SNAP_VERSION=\"$snap_version\"" >> /snap/bin/snapcraft
RUN echo 'exec "$SNAP/usr/bin/python3" "$SNAP/bin/snapcraft" "$@"' >> /snap/bin/snapcraft
RUN chmod +x /snap/bin/snapcraft

# Grab gtk-common-themes snap package
RUN curl -L $(curl -H 'X-Ubuntu-Series: 16' 'https://api.snapcraft.io/api/v1/snaps/details/gtk-common-themes' | jq '.download_url' -r) --output gtk-common-themes.snap
RUN mkdir -p /snap/gtk-common-themes
RUN unsquashfs -d /snap/gtk-common-themes/current gtk-common-themes.snap

# at the moment kde neon kf6 isn't available, adjust when necessary
#
# # Grab the core22 kf5 snap and unpack it in the proper place.
#
# RUN curl -L $(curl -H 'X-Ubuntu-Series: 16' "https://api.snapcraft.io/api/v1/snaps/details/${KF5VERSION}" | jq '.download_url' -r) --output ${KF5VERSION}.snap
# RUN mkdir -p /snap/${KF5VERSION}
# RUN unsquashfs -d /snap/${KF5VERSION}/current ${KF5VERSION}.snap
#
# RUN curl -L $(curl -H 'X-Ubuntu-Series: 16' "https://api.snapcraft.io/api/v1/snaps/details/${KF5VERSION}-sdk" | jq '.download_url' -r) --output ${KF5VERSION}-sdk.snap
# RUN mkdir -p /snap/${KF5VERSION}-sdk
# RUN unsquashfs -d /snap/${KF5VERSION}-sdk/current ${KF5VERSION}-sdk.snap

# Multi-stage build, only need the snaps from the builder. Copy them one at a
# time so they can be cached.

FROM ubuntu:$UBUNTU

MAINTAINER VideoLAN roots <roots@videolan.org>

# If someone wants to use VideoLAN docker images on a local machine and does
# not want to be disturbed by the videolan user, we should not take an uid/gid
# in the user range of main distributions, which means:
# - Debian based: <1000
# - RPM based: <500 (CentOS, RedHat, etc.)
ARG VIDEOLAN_UID=499
ARG KF5VERSION

COPY --from=builder /snap/core /snap/core
COPY --from=builder /snap/core20 /snap/core20
COPY --from=builder /snap/core22 /snap/core22
COPY --from=builder /snap/gtk-common-themes /snap/gtk-common-themes
#COPY --from=builder /snap/${KF5VERSION} /snap/${KF5VERSION}
#COPY --from=builder /snap/${KF5VERSION}-sdk /snap/${KF5VERSION}-sdk
COPY --from=builder /snap/snapcraft /snap/snapcraft
COPY --from=builder /snap/bin/snapcraft /snap/bin/snapcraft

# Generate locale and install dependencies.
# We don't clean apt cache afterwards as snap will use the cache to retreive the packages
RUN set -x && \
    echo "add videolan user" && \
    export DEBIAN_FRONTEND=noninteractive && \
    addgroup --quiet --gid ${VIDEOLAN_UID} videolan && \
    adduser --quiet --uid ${VIDEOLAN_UID} --ingroup videolan videolan && \
    echo "videolan:videolan" | chpasswd &&  \
    echo "install snapd" && \
    apt-get update && \
    apt-get dist-upgrade --yes && \
    apt-get install --yes snapd sudo locales && \
    echo "pre-install VLC dependencies as snap is able to use them directly (should match snapcraft.yaml packages)" && \
    echo "install mesa stage packages" && \
    apt-get install --yes \
      libgl1-mesa-dri \
      libglx-mesa0 \
    && \
    echo "install vlc build packages" && \
    apt-get install --yes \
      ant \
      autopoint \
      git \
      g++ \
      gperf \
      nasm \
      make \
      autoconf \
      libtool \
      libtool-bin \
      cmake \
      automake \
      build-essential \
      curl \
      gettext \
      openjdk-11-jdk-headless \
      python3 \
      python3-pip \
      python3-venv \
      python3-jinja2 \
      lua5.2 \
      pkg-config \
      xz-utils \
      zlib1g-dev \
      bison \
      flex \
      protobuf-compiler \
      glslang-dev \
      libaom-dev \
      libarchive-dev \
      libaribb24-dev \
      libasound2-dev \
      libass-dev \
      libavcodec-dev \
      libavdevice-dev \
      libavfilter-dev \
      libavformat-dev \
      libavutil-dev \
      libbluray-dev \
      libcaca-dev \
      libcddb2-dev \
      libchromaprint-dev \
      libdav1d-dev \
      libdca-dev \
      libdvbpsi-dev \
      libdvdnav-dev \
      libdvdread-dev \
      libebur128-dev \
      libfaad-dev \
      libffi-dev \
      libflac-dev \
      libfluidsynth-dev \
      libgmp-dev \
      libgsm1-dev \
      libjack-dev \
      libjpeg-turbo8-dev \
      libkate-dev \
      libmad0-dev \
      libmatroska-dev \
      libmodplug-dev \
      libmpeg2-4-dev \
      libmpg123-dev \
      libmysofa-dev \
      libnfs-dev \
      libogg-dev \
      liborc-0.4-dev \
      libplacebo-dev \
      libpostproc-dev \
      libsamplerate0-dev \
      libshout-dev \
      libsidplay2-dev \
      libsoxr-dev \
      libspatialaudio-dev \
      libspeex-dev \
      libspeexdsp-dev \
      libsrt-gnutls-dev \
      libssh2-1-dev \
      libswresample-dev \
      libswscale-dev \
      libtag1-dev \
      libtheora-dev \
      libtwolame-dev \
      libupnp-dev \
      libvncserver-dev \
      libvorbis-dev \
      libvpx-dev \
      libx264-dev \
      libx265-dev \
      libxcb1-dev \
      libxcb-xfixes0-dev \
      libasound2-dev \
      libavahi-client-dev \
      libcdio-dev \
      libdbus-1-dev \
      libdirectfb-dev \
      libdrm-dev \
      libegl1-mesa-dev \
      libfreetype6-dev \
      libfribidi-dev \
      libgbm-dev \
      libgles2-mesa-dev \
      libgnutls28-dev \
      libglib2.0-dev \
      libgtk-3-dev \
      libidn-dev \
      libjack-dev \
      liblircclient-dev \
      liblua5.2-dev \
      libmtp-dev \
      libncursesw5-dev \
      libpng-dev \
      libpulse-dev \
      librsvg2-dev \
      libsecret-1-dev \
      libudev-dev \
      libv4l-dev \
      libva-dev \
      libvdpau-dev \
      libx11-dev \
      libxcb-composite0-dev \
      libxcb-damage0-dev \
      libxcb-keysyms1-dev \
      libxcb-randr0-dev \
      libxcb-shm0-dev \
      libxcb-xv0-dev \
      libxcb1-dev \
      libxext-dev \
      libxi-dev \
      libxinerama-dev \
      libxkbcommon-x11-dev \
      libxml2-dev \
      libxpm-dev \
      libzvbi-dev \
      libvulkan-dev \
      libgcrypt20-dev \
      libsystemd-dev \
      libprotobuf-dev \
      wayland-protocols \
      qt6-shader-baker \
      libqt6svg6-dev \
      libqt6opengl6-dev \
      libqt6shadertools6-dev \
      qt6-wayland-dev \
      qt6-base-dev \
      qt6-base-private-dev \
      qt6-declarative-dev \
      qt6-declarative-private-dev \
      qml6-module-qttest \
      qml6-module-qt5compat-graphicaleffects \
      qml6-module-qtcore \
      qml6-module-qtqml \
      qml6-module-qtqml-models \
      qml6-module-qtquick \
      qml6-module-qtquick-controls \
      qml6-module-qtquick-layouts \
      qml6-module-qtquick-templates \
      qml6-module-qtquick-window \
    && \
    echo "install vlc stage packages" && \
    apt-get install --yes \
      i965-va-driver \
      intel-media-va-driver \
      dbus-x11 \
      fonts-freefont-ttf \
      libaacs0 \
      libaom3 \
      libarchive13 \
      libaribb24-0 \
      libasound2 \
      libass9 \
      libatk1.0-0 \
      libavcodec58 \
      libavformat58 \
      libavutil56 \
      libbluray2 \
      libcaca0 \
      libcddb2 \
      libchromaprint1 \
      libdav1d5 \
      libdb5.3 \
      libdca0 \
      libdvbpsi10 \
      libdvdnav4 \
      libdvdread8 \
      libebur128-1 \
      libegl1-mesa \
      libfluidsynth3 \
      libfreetype6 \
      libfribidi0 \
      libgcc1 \
      libgcrypt20 \
      libgl1-mesa-glx \
      libgles2-mesa \
      libglib2.0-0 \
      libglu1-mesa \
      libgtk-3-0 \
      libidn12 \
      libjack0 \
      libkate1 \
      liblua5.2-0 \
      libmad0 \
      libmatroska7 \
      libmodplug1 \
      libmp3lame0 \
      libmpeg2-4 \
      libmpg123-0 \
      libmtp9 \
      libmysofa1 \
      libnfs13 \
      libnotify4 \
      libopenjp2-7 \
      liborc-0.4-0 \
      libplacebo192 \
      libpostproc55 \
      libprotobuf23 \
      libpulse0 \
      librsvg2-2 \
      libsamplerate0 \
      libsecret-1-0 \
      libshout3 \
      libsndio7.0 \
      libsoxr0 \
      libspatialaudio0 \
      libspeex1 \
      libspeexdsp1 \
      libsrt1.4-gnutls \
      libssh2-1 \
      libswresample3 \
      libswscale5 \
      libtag1v5-vanilla \
      libtheora0 \
      libtwolame0 \
      libupnp13 \
      libva-drm2 \
      libva-wayland2 \
      libva-x11-2 \
      libva2 \
      libvdpau1 \
      libvncclient1 \
      libvorbisfile3 \
      libvpx7 \
      libvulkan1 \
      libx11-6 \
      libx264-163 \
      libx265-199 \
      libxcb-composite0 \
      libxcb-damage0 \
      libxcb-keysyms1 \
      libxcb-randr0 \
      libxcb-shm0 \
      libxcb-xv0 \
      libxcb1 \
      libxcomposite1 \
      libxcursor1 \
      libxext6 \
      libxi6 \
      libxinerama1 \
      libxkbcommon-x11-0 \
      libxpm4 \
      libxrandr2 \
      libzvbi0 \
      mesa-utils \
      mesa-va-drivers \
      mesa-vdpau-drivers \
      mesa-vulkan-drivers \
      openjdk-8-jdk \
      vdpau-driver-all \
      zlib1g \
      qt6-wayland \
      qt6-qpa-plugins \
      qml6-module-qt5compat-graphicaleffects \
      qml6-module-qtcore \
      qml6-module-qtqml \
      qml6-module-qtqml-models \
      qml6-module-qtquick \
      qml6-module-qtquick-controls \
      qml6-module-qtquick-layouts \
      qml6-module-qtquick-templates \
      qml6-module-qtquick-window \
      qml6-module-qtqml-workerscript \
      libqt6core6 \
      libqt6gui6 \
      libqt6widgets6 \
      libqt6svg6 \
      libqt6qmlmodels6 \
      libqt6qmlworkerscript6 \
      libqt6quick6 \
      libqt6quickcontrols2-6 \
      libqt6quicklayouts6 \
      libqt6quicktemplates2-6 \
      libqt6qml6 \
      libqt6opengl6 \
      libqt6shadertools6 \
      qt6-xdgdesktopportal-platformtheme \
    && \
    echo "install meson and ninja" \
    pip3 install meson ninja && \
    locale-gen en_US.UTF-8

# Set the proper environment.
ENV LANG="en_US.UTF-8"
ENV LANGUAGE="en_US:en"
ENV LC_ALL="en_US.UTF-8"
ENV PATH="/snap/bin:/snap/snapcraft/current/usr/bin:/snap/snapcraft/current/libexec/snapcraft:$PATH"
ENV SNAP="/snap/snapcraft/current"
ENV SNAP_NAME="snapcraft"
ENV SNAP_ARCH="amd64"
# same as passing --destructive-mode to snapcraft
ENV SNAPCRAFT_BUILD_ENVIRONMENT="host"
# this prevent snapcraft from running "apt-get update" as this requires root privilege.
# all packages should be already installed in the docker
ENV CRAFT_PARTS_PACKAGE_REFRESH=0
USER videolan
