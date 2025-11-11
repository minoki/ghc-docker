# syntax=docker/dockerfile:1
FROM rust:1.91.1-slim-trixie AS rust-builder

RUN cargo install --root /opt/wasm-tools \
 wasm-tools@1.240.0 \
 wasm-component-ld@0.5.18 \
 wasmi_cli@0.51.1 \
 wit-bindgen-cli@0.47.0 \
 wizer@10.0.0 \
 wasmtime-cli@38.0.3

FROM ubuntu:24.04 AS wasi-sdk-builder

# wasi-sdk requires cmake, ninja-build, clang, ccache, lld
# libffi-wasm requires clang-format
RUN apt-get update && apt-get install -y build-essential curl git libgmp-dev libnuma1 autoconf cmake ninja-build clang ccache lld clang-format

WORKDIR /work
# RUN git clone --recursive --depth 1 --shallow-submodules https://gitlab.haskell.org/haskell-wasm/wasi-sdk.git
RUN git clone --depth 1 https://gitlab.haskell.org/haskell-wasm/wasi-sdk.git
WORKDIR /work/wasi-sdk
RUN git fetch --depth 1 origin 024da5dee1ec2bc2eff0163bf07f03ff5e30e2b3 && git checkout 024da5dee1ec2bc2eff0163bf07f03ff5e30e2b3
RUN git submodule update --init src/config && git submodule update --init --depth 1
RUN cmake -G Ninja -B build/toolchain -S . -DWASI_SDK_BUILD_TOOLCHAIN=ON -DCMAKE_INSTALL_PREFIX=build/install
RUN cmake --build build/toolchain --target install --target dist
RUN cmake -G Ninja -B build/sysroot -S . \
    -DCMAKE_INSTALL_PREFIX=build/install \
    -DCMAKE_TOOLCHAIN_FILE=build/install/share/cmake/wasi-sdk.cmake \
    -DCMAKE_C_COMPILER_WORKS=ON \
    -DCMAKE_CXX_COMPILER_WORKS=ON
COPY --from=rust-builder /opt/wasm-tools /opt/wasm-tools
ENV PATH=/opt/wasm-tools/bin:$PATH
RUN cmake --build build/sysroot --target install --target dist
RUN mkdir dist-`uname -m`-linux && \
 cp build/toolchain/dist/* build/sysroot/dist/* dist-`uname -m`-linux && \
 ./ci/merge-artifacts.sh
WORKDIR /opt
RUN tar xf /work/wasi-sdk/dist/wasi-sdk-*-linux.tar.gz && ln -s wasi-sdk-* wasi-sdk
ENV WASI_SDK_PATH=/opt/wasi-sdk

FROM ubuntu:24.04 AS ghc-builder

# See https://gitlab.haskell.org/haskell-wasm/ghc-wasm-meta#building-guide

# Needed because GHC's utils/genprimopcode/Main.hs contains non-ASCII character
ENV LANG=C.UTF-8

# wasi-sdk requires cmake, ninja-build, clang, ccache, lld
# libffi-wasm requires clang-format
RUN apt-get update && apt-get install -y build-essential curl git libgmp-dev libnuma1 autoconf cmake ninja-build clang ccache lld clang-format

ENV BOOTSTRAP_HASKELL_NONINTERACTIVE=1
ENV BOOTSTRAP_HASKELL_GHC_VERSION=9.8.4
ENV BOOTSTRAP_HASKELL_CABAL_VERSION=3.12.1.0
ENV BOOTSTRAP_HASKELL_INSTALL_NO_STACK=1
ENV BOOTSTRAP_HASKELL_CABAL_XDG=1
RUN curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
# By default, cabal 3.10 uses $HOME/.local and $HOME/.config and $HOME/.cache instead of $HOME/.cabal.
# GHCup's default is to use .cabal instead of XDG directories.
# GHCup's preference can be configured with BOOTSTRAP_HASKELL_CABAL_XDG variable
ENV PATH=/root/.local/bin:/root/.ghcup/bin:$PATH
RUN cabal update && cabal install --constraint="alex==3.5.*" --constraint="happy==2.1.*" alex happy

ENV WASI_SDK_PATH=/opt/wasi-sdk
COPY --from=wasi-sdk-builder /opt/wasi-sdk /opt/wasi-sdk

WORKDIR /work
RUN git clone https://gitlab.haskell.org/haskell-wasm/libffi-wasm.git
WORKDIR /work/libffi-wasm
RUN git checkout e2dbdd8ded953e7a4df42c192da65ab36b9f0345
RUN cabal v2-run libffi-wasm && \
 ${WASI_SDK_PATH}/bin/clang -Wall -Wextra -O3 -DNDEBUG -Icbits -c cbits/ffi.c -o cbits/ffi.o && \
 ${WASI_SDK_PATH}/bin/clang -Wall -Wextra -O3 -DNDEBUG -Icbits -c cbits/ffi_call.c -o cbits/ffi_call.o && \
 ${WASI_SDK_PATH}/bin/clang -Wall -Wextra -O3 -DNDEBUG -Icbits -c cbits/ffi_closure.c -o cbits/ffi_closure.o && \
 mkdir -p out/libffi-wasm/include && \
 cp cbits/*.h out/libffi-wasm/include && \
 mkdir -p out/libffi-wasm/lib && \
 ${WASI_SDK_PATH}/bin/llvm-ar -r out/libffi-wasm/lib/libffi.a cbits/*.o
RUN cp out/libffi-wasm/include/*.h ${WASI_SDK_PATH}/share/wasi-sysroot/include && \
 cp out/libffi-wasm/lib/*.a ${WASI_SDK_PATH}/share/wasi-sysroot/lib/wasm32-wasi

ARG GHC_VERSION=9.8.4

WORKDIR /work
ADD --checksum=sha256:17e8188f3c8a5c2f73fb4e35d01032e8dc258835ec876d52c8ad8ee3d24b2fc5 https://downloads.haskell.org/~ghc/$GHC_VERSION/ghc-$GHC_VERSION-src.tar.xz ghc-$GHC_VERSION-src.tar.xz
RUN tar -xJf ghc-$GHC_VERSION-src.tar.xz
WORKDIR /work/ghc-$GHC_VERSION

# If building with GHC 9.6.7: Workaround https://gitlab.haskell.org/ghc/ghc/-/issues/25963
# The patch should be taken from 189a368ba74dbfe8a6e832fe0f9ffe56cd565824
# ADD 0001-compiler-Fix-CPP-guards-around-ghc_unique_counter64.patch .
# RUN git apply 0001-compiler-Fix-CPP-guards-around-ghc_unique_counter64.patch

RUN ./configure --target=wasm32-wasi --with-intree-gmp --with-system-libffi \
 AR=${WASI_SDK_PATH}/bin/llvm-ar \
 CC=${WASI_SDK_PATH}/bin/clang \
 CFLAGS=--sysroot=${WASI_SDK_PATH}/share/wasi-sysroot \
 CC_FOR_BUILD=cc \
 CXX=${WASI_SDK_PATH}/bin/clang++ \
 CXXFLAGS=--sysroot=${WASI_SDK_PATH}/share/wasi-sysroot \
 LD=${WASI_SDK_PATH}/bin/wasm-ld \
 NM=${WASI_SDK_PATH}/bin/llvm-nm \
 OBJCOPY=${WASI_SDK_PATH}/bin/llvm-objcopy \
 OBJDUMP=${WASI_SDK_PATH}/bin/llvm-objdump \
 RANLIB=${WASI_SDK_PATH}/bin/llvm-ranlib \
 SIZE=${WASI_SDK_PATH}/bin/llvm-size \
 STRINGS=${WASI_SDK_PATH}/bin/llvm-strings \
 STRIP=${WASI_SDK_PATH}/bin/llvm-strip \
 CONF_CC_OPTS_STAGE2="-Wno-error=int-conversion -Wno-error=strict-prototypes -Wno-error=implicit-function-declaration -Oz -mnontrapping-fptoint -msign-ext -mbulk-memory -mmutable-globals -mreference-types" \
 CONF_CXX_OPTS_STAGE2="-Wno-error=int-conversion -Wno-error=strict-prototypes -Wno-error=implicit-function-declaration -fno-exceptions -Oz -mnontrapping-fptoint -msign-ext -mbulk-memory -mmutable-globals -mreference-types" \
 CONF_GCC_LINKER_OPTS_STAGE2="-Wl,--compress-relocations,--error-limit=0,--growable-table,--stack-first,--strip-debug -Wno-unused-command-line-argument"
RUN hadrian/build --flavour=quick --docs=none -j binary-dist-dir
# 'hadrian/build install' cannot be used because of https://gitlab.haskell.org/ghc/ghc/-/issues/21970
WORKDIR /work/ghc-$GHC_VERSION/_build/bindist/ghc-$GHC_VERSION-wasm32-wasi
RUN ./configure --target=wasm32-wasi --host=$(uname -m)-linux-gnu --prefix=/opt/ghc \
 AR=${WASI_SDK_PATH}/bin/llvm-ar \
 CC=${WASI_SDK_PATH}/bin/clang \
 CFLAGS=--sysroot=${WASI_SDK_PATH}/share/wasi-sysroot \
 CC_FOR_BUILD=cc \
 CXX=${WASI_SDK_PATH}/bin/clang++ \
 CXXFLAGS=--sysroot=${WASI_SDK_PATH}/share/wasi-sysroot \
 LD=${WASI_SDK_PATH}/bin/wasm-ld \
 NM=${WASI_SDK_PATH}/bin/llvm-nm \
 OBJCOPY=${WASI_SDK_PATH}/bin/llvm-objcopy \
 OBJDUMP=${WASI_SDK_PATH}/bin/llvm-objdump \
 RANLIB=${WASI_SDK_PATH}/bin/llvm-ranlib \
 SIZE=${WASI_SDK_PATH}/bin/llvm-size \
 STRINGS=${WASI_SDK_PATH}/bin/llvm-strings \
 STRIP=${WASI_SDK_PATH}/bin/llvm-strip \
 CONF_CC_OPTS_STAGE2="-Wno-error=int-conversion -Wno-error=strict-prototypes -Wno-error=implicit-function-declaration -Oz -mnontrapping-fptoint -msign-ext -mbulk-memory -mmutable-globals -mreference-types" \
 CONF_CXX_OPTS_STAGE2="-Wno-error=int-conversion -Wno-error=strict-prototypes -Wno-error=implicit-function-declaration -fno-exceptions -Oz -mnontrapping-fptoint -msign-ext -mbulk-memory -mmutable-globals -mreference-types" \
 CONF_GCC_LINKER_OPTS_STAGE2="-Wl,--compress-relocations,--error-limit=0,--growable-table,--stack-first,--strip-debug -Wno-unused-command-line-argument" && \
 make install

FROM ubuntu:24.04

# Some cabal commands may need git to be installed
RUN apt-get update && apt-get install -y git

COPY --from=ghc-builder /opt /opt
COPY --from=ghc-builder /root/.ghcup/bin/cabal /root/.ghcup/bin/cabal
# cabal 3.10 uses $HOME/.local and $HOME/.config and $HOME/.cache instead of $HOME/.cabal
COPY --from=ghc-builder /root/.local/bin/* /root/.local/bin/
COPY --from=ghc-builder /root/.config/cabal /root/.config/cabal
COPY --from=ghc-builder /root/.cache/cabal /root/.cache/cabal
ENV PATH="/opt/ghc/bin:/root/.local/bin:/root/.ghcup/bin:$PATH"
RUN sed -i -e 's|-- with-compiler:|with-compiler: /opt/ghc/bin/wasm32-wasi-ghc|g' -e 's|-- with-hc-pkg:|with-hc-pkg: /opt/ghc/bin/wasm32-wasi-ghc-pkg|g' -e 's|-- hsc2hs-location:|hsc2hs-location: /opt/ghc/bin/wasm32-wasi-hsc2hs|g' /root/.config/cabal/config
CMD ["/opt/ghc/bin/wasm32-wasi-ghc"]
