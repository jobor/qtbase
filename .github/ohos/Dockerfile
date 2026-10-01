# Native OpenHarmony build environment: dockerharmony + Harmonybrew + a Qt
# toolchain. arm64 only; meant to run on a native arm64 host, not under qemu.
#
#   docker build -t ohos-qt-builder .github/ohos
#
# The image carries the toolchain only. Build Qt outside it, in a bind-mounted
# work tree, so the build stays on host disk:
#
#   docker run --rm --user "$(id -u):$(id -g)" \
#     -e HOME=/build/home -e TMPDIR=/build/tmp \
#     -v "$PWD:/src:ro" -v /var/tmp/qtbase-ohos-build:/build \
#     ohos-qt-builder cmake -S /src -B /build -G Ninja -DOHOS=ON
#
# --user keeps artifacts owned by the caller rather than root. HOME and TMPDIR
# must point at something writable because that uid has no passwd entry and the
# container's own /tmp is not writable by it.
#
# No toolchain file and no CMAKE_SYSTEM_NAME: this is a native build, so Qt
# takes its native-HarmonyOS path. Qt would set OHOS itself if the host looked
# like HarmonyOS, but uname in a container comes from the host kernel and says
# Linux, so -DOHOS=ON has to be passed by hand.
FROM hqzing/dockerharmony:7.0

ENV HOMEBREW_PREFIX=/storage/Users/currentUser/.harmonybrew \
    HOMEBREW_NO_ANALYTICS=1 \
    HOMEBREW_NO_AUTO_UPDATE=1 \
    HOMEBREW_NO_ENV_HINTS=1

# Harmonybrew's bin/brew is "#!/usr/bin/zsh", but the OpenHarmony rootfs ships
# only mksh. Unpack zsh and its dependencies straight from Harmonybrew's bottle
# server to break the cycle. Bottles are :any_skip_relocation and their RUNPATH
# is absolute, so they only work under the default prefix.
ARG ZSH_BOTTLE=zsh-5.9.2_2
ARG NCURSES_BOTTLE=ncurses-6.6_1
ARG PCRE2_BOTTLE=pcre2-10.48
RUN set -e; \
    b="https://harmonybrew.atomgit.com/bottles"; \
    mkdir -p "$HOMEBREW_PREFIX/Cellar" "$HOMEBREW_PREFIX/opt"; \
    cd /tmp; \
    for f in "$ZSH_BOTTLE" "$NCURSES_BOTTLE" "$PCRE2_BOTTLE"; do \
        name="${f%-*}"; ver="${f##*-}"; \
        curl -fsSL -o b.tgz "$b/$f.arm64_ohos.bottle.tar.gz"; \
        tar -zxf b.tgz -C "$HOMEBREW_PREFIX/Cellar"; \
        ln -sfn "$HOMEBREW_PREFIX/Cellar/$name/$ver" "$HOMEBREW_PREFIX/opt/$name"; \
        rm -f b.tgz; \
    done; \
    mkdir -p /usr/bin; \
    ln -sf "$HOMEBREW_PREFIX/opt/zsh/bin/zsh" /usr/bin/zsh; \
    zsh --version

RUN NONINTERACTIVE=1 zsh -c "$(curl -fsSL https://harmonybrew.atomgit.com/install.sh)"

ENV PATH=${HOMEBREW_PREFIX}/bin:${HOMEBREW_PREFIX}/sbin:${PATH}

RUN brew install cmake ninja python git pkgconf ohos-sdk-native

ENV NATIVE_OHOS_SDK=${HOMEBREW_PREFIX}/opt/ohos-sdk-native
ENV OHOS_TOOLCHAIN_FILE=${NATIVE_OHOS_SDK}/build/cmake/ohos.toolchain.cmake

# The rootfs has no ar/ranlib/strip at all; the only ones are the SDK's
# llvm-* tools. Appended, not prepended, so brew's tools keep priority.
ENV PATH=${PATH}:${NATIVE_OHOS_SDK}/llvm/bin
ENV AR=${NATIVE_OHOS_SDK}/llvm/bin/llvm-ar \
    RANLIB=${NATIVE_OHOS_SDK}/llvm/bin/llvm-ranlib

# Qt's third-party dependencies come from Harmonybrew, which carries arm64_ohos
# bottles for all of them. That replaces the prebuilt additional-packages
# archive: nothing to host, nothing to redistribute, and the versions are
# whatever the bottle server serves.
# Exactly what the archive provided, and no more. Extras are tempting but
# change what Qt configures: harfbuzz in particular pulls in cairo, pixman,
# libxext and libxrender, which makes Qt detect X11 and enable eglfs_x11,
# which then fails to compile because it converts an X11 Window to OHOS's
# NativeWindow *. Qt bundles harfbuzz, md4c, libb2, sqlite and
# double-conversion itself; leave it that way.
RUN brew install brotli expat fontconfig freetype icu4c jpeg-turbo libpng \
        util-linux zlib pcre2

# node-addon-api is header-only and not in Harmonybrew. Upstream plus the OHOS
# workarounds patch from Qt's vcpkg registry, mirroring what the vcpkg port
# does; node-api-headers supplies the C API it includes. Applied with git
# apply because the rootfs patch(1) is toybox and rejects the diff.
ARG NODE_API_HEADERS_VERSION=1.9.0
ARG NODE_ADDON_API_VERSION=8.9.0
ARG QT_VCPKG_BASELINE=a3e9002baa5eb8ac3bd12ef85818ffe5c0c7322d
RUN set -e; \
    inc="$NATIVE_OHOS_SDK/sysroot/usr/include"; \
    cd /tmp; \
    curl -fsSL -o nah.tar.gz "https://github.com/nodejs/node-api-headers/archive/refs/tags/v${NODE_API_HEADERS_VERSION}.tar.gz"; \
    tar -xzf nah.tar.gz; \
    cd "node-api-headers-${NODE_API_HEADERS_VERSION}"; \
    cp include/js_native_api.h include/js_native_api_types.h \
       include/node_api.h include/node_api_types.h "$inc/"; \
    cd /tmp; \
    curl -fsSL -o nan.tar.gz "https://github.com/nodejs/node-addon-api/archive/refs/tags/v${NODE_ADDON_API_VERSION}.tar.gz"; \
    tar -xzf nan.tar.gz; \
    cd "node-addon-api-${NODE_ADDON_API_VERSION}"; \
    curl -fsSL -o /tmp/ohos.patch \
      "https://git.qt.io/qtbuildsystem/vcpkg/-/raw/${QT_VCPKG_BASELINE}/ports/node-addon-api/ohos-napi-workarounds.patch"; \
    git apply -p1 /tmp/ohos.patch; \
    cp napi.h napi-inl.h napi-inl.deprecated.h "$inc/"; \
    cd /tmp; rm -rf node-api-headers-* node-addon-api-* nah.tar.gz nan.tar.gz ohos.patch

ENV PKG_CONFIG_PATH=${HOMEBREW_PREFIX}/lib/pkgconfig:${NATIVE_OHOS_SDK}/sysroot/usr/lib/pkgconfig \
    CMAKE_PREFIX_PATH=${HOMEBREW_PREFIX}:${NATIVE_OHOS_SDK}/sysroot/usr

# The SDK clang is not on PATH as cc/c++; name it so CMake picks it up without
# a toolchain file.
ENV CC=${NATIVE_OHOS_SDK}/llvm/bin/clang \
    CXX=${NATIVE_OHOS_SDK}/llvm/bin/clang++

# dockerharmony is an OpenHarmony userland, not a system image, so it ships
# none of the platform libraries. Without them libQt6Core cannot be loaded and
# no build-time tool linking it can run, which stops the build at qvkgen.
#
# Every link-time stub the SDK declares and the container cannot resolve gets
# a generated no-op stub. Build-time tools never call into the platform: across
# full qtbase, qtshadertools and qtdeclarative builds with OHOS_SHIM_TRACE on,
# not one stub was ever invoked. They exist only to satisfy the loader.
#
# /system/lib64 is first in the container's musl search path.
RUN set -e; mkdir -p /system/lib64
COPY gen-shims.py /tmp/ctx/
RUN python3 /tmp/ctx/gen-shims.py "$NATIVE_OHOS_SDK" /system/lib64 "$CC"

# Qt links some third-party libraries from Harmonybrew rather than the
# additional-packages archive, pcre2 among them, because the archive ships no
# pcre2 at all. Brew's lib directory is on no default search path, so those
# resolve while building, where CMake gives binaries an RPATH covering the
# link directories, and then fail to load once Qt is installed and that RPATH
# is replaced. Anything dlopening a brew library, vcpkg looking for
# libcurl.so.4 for instance, fails the same way.
#
# LD_LIBRARY_PATH rather than a loader config file: OHOS musl resolves
# libraries through the namespace config in /etc/ld-musl-namespace-aarch64.ini,
# not /etc/ld-musl-aarch64.path, and editing the latter has no effect.
ENV LD_LIBRARY_PATH=${HOMEBREW_PREFIX}/lib

RUN cmake --version && ninja --version && python3 --version && git --version && \
    "$CC" --version && \
    ls "$HOMEBREW_PREFIX/lib/libpng16.so" && \
    ls "$NATIVE_OHOS_SDK/sysroot/usr/include/napi.h"
