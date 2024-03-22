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
      git \
      g++ \
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
      lua5.2 \
      pkg-config \
      xz-utils \
      zlib1g-dev \
      bison \
      flex \
      libvulkan-dev \
      libgcrypt20-dev \
      libsystemd-dev \
      libprotobuf-dev \
      protobuf-compiler \
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
      fonts-freefont-ttf \
      dbus-x11 \
      libegl1-mesa \
      libfreetype6 \
      libfribidi0 \
      libdb5.3 \
      libgcc1 \
      libgl1-mesa-glx \
      libgles2-mesa \
      libglib2.0-0 \
      libpulse0 \
      librsvg2-2 \
      libsecret-1-0 \
      libva-drm2 \
      libva-x11-2 \
      libva2 \
      libvdpau1 \
      libx11-6 \
      libxcb-composite0 \
      libxcb-damage0 \
      libxcb-keysyms1 \
      libxcb-randr0 \
      libxcb-shm0 \
      libxcb-xv0 \
      libxcb1 \
      libxext6 \
      libxi6 \
      libxinerama1 \
      libxkbcommon-x11-0 \
      libxpm4 \
      vdpau-driver-all \
      i965-va-driver \
      intel-media-va-driver \
      mesa-utils \
      mesa-vdpau-drivers \
      mesa-vulkan-drivers \
      mesa-va-drivers \
      zlib1g \
      libjack0 \
      libasound2 \
      libgtk-3-0 \
      liblua5.2-0 \
      libmtp9 \
      libzvbi0 \
      libnotify4 \
      libvulkan1 \
      libaacs0 \
      libvulkan-dev \
      openjdk-8-jdk \
      libgcrypt20 \
      libidn12 \
      libxcomposite1 \
      libxcursor1 \
      libxrandr2 \
      libatk1.0-0 \
      libva-wayland2 \
      libprotobuf23 \
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
      libqt6quickshapes6 \
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

USER videolan
