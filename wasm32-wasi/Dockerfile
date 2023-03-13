# syntax=docker/dockerfile:1
FROM ubuntu:22.04

# See https://gitlab.haskell.org/ghc/ghc-wasm-meta#building-guide

# Needed because GHC's utils/genprimopcode/Main.hs contains non-ASCII character
ENV LANG=C.UTF-8

# wasi-sdk requires cmake, ninja-build, clang, ccache, lld
# libffi-wasm requires clang-format
RUN apt-get update && apt-get install -y build-essential curl git libgmp-dev libnuma1 autoconf cmake ninja-build clang ccache lld clang-format

ENV BOOTSTRAP_HASKELL_NONINTERACTIVE=1
ENV BOOTSTRAP_HASKELL_GHC_VERSION=9.2.7
ENV BOOTSTRAP_HASKELL_CABAL_VERSION=3.10.1.0
ENV BOOTSTRAP_HASKELL_INSTALL_NO_STACK=1
RUN curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
# cabal 3.10 uses $HOME/.local and $HOME/.config and $HOME/.cache instead of $HOME/.cabal
ENV PATH=/root/.local/bin:/root/.ghcup/bin:$PATH
RUN cabal update && cabal install alex happy

WORKDIR /work
RUN git clone --recursive --depth 1 --shallow-submodules https://gitlab.haskell.org/ghc/wasi-sdk.git
WORKDIR /work/wasi-sdk
RUN make package
ARG WASI_VERSION=16
WORKDIR /opt
RUN tar xf /work/wasi-sdk/dist/wasi-sdk-${WASI_VERSION}-linux.tar.gz
ENV WASI_SDK_PATH=/opt/wasi-sdk-${WASI_VERSION}

WORKDIR /work
RUN git clone https://gitlab.haskell.org/ghc/libffi-wasm.git
WORKDIR /work/libffi-wasm
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

ARG GHC_VERSION=9.6.1

WORKDIR /work
RUN curl -LO https://downloads.haskell.org/~ghc/$GHC_VERSION/ghc-$GHC_VERSION-src.tar.xz && tar -xJf ghc-$GHC_VERSION-src.tar.xz
WORKDIR /work/ghc-$GHC_VERSION

# Workaround https://gitlab.haskell.org/ghc/ghc/-/issues/22981
ADD hsc2hs.patch .
RUN git apply --directory=utils/hsc2hs hsc2hs.patch

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

FROM ubuntu:22.04

COPY --from=0 /opt /opt
COPY --from=0 /root/.ghcup/bin/cabal /root/.ghcup/bin/cabal
# cabal 3.10 uses $HOME/.local and $HOME/.config and $HOME/.cache instead of $HOME/.cabal
COPY --from=0 /root/.local/bin/* /root/.local/bin/
COPY --from=0 /root/.config/cabal /root/.config/cabal
COPY --from=0 /root/.cache/cabal /root/.cache/cabal
ENV PATH="/opt/ghc/bin:/root/.local/bin:/root/.ghcup/bin:$PATH"
RUN sed -i -e 's|-- with-compiler:|with-compiler: /opt/ghc/bin/wasm32-wasi-ghc|g' -e 's|-- with-hc-pkg:|with-hc-pkg: /opt/ghc/bin/wasm32-wasi-ghc-pkg|g' /root/.config/cabal/config
CMD ["/opt/ghc/bin/wasm32-wasi-ghc"]
