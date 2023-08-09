# syntax=docker/dockerfile:1
FROM ubuntu:22.04

# Needed because GHC's utils/genprimopcode/Main.hs contains non-ASCII character
ENV LANG=C.UTF-8

RUN apt-get update && apt-get install -y build-essential curl git libgmp-dev libnuma1 autoconf llvm-15 gcc-riscv64-linux-gnu g++-riscv64-linux-gnu

ENV BOOTSTRAP_HASKELL_NONINTERACTIVE=1
ENV BOOTSTRAP_HASKELL_GHC_VERSION=9.6.2
ENV BOOTSTRAP_HASKELL_CABAL_VERSION=3.10.1.0
ENV BOOTSTRAP_HASKELL_INSTALL_NO_STACK=1
RUN curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
# cabal 3.10 uses $HOME/.local and $HOME/.config and $HOME/.cache instead of $HOME/.cabal
ENV PATH=/root/.local/bin:/root/.ghcup/bin:$PATH
RUN cabal update && cabal install alex happy

ARG GHC_VERSION=9.8.0.20230727

WORKDIR /work
RUN curl -LO https://downloads.haskell.org/~ghc/$GHC_VERSION/ghc-$GHC_VERSION-src.tar.xz && tar -xJf ghc-$GHC_VERSION-src.tar.xz
WORKDIR /work/ghc-$GHC_VERSION

RUN ./configure --target=riscv64-linux-gnu
RUN hadrian/build --bignum=native -j binary-dist-dir
# 'hadrian/build install' cannot be used because of https://gitlab.haskell.org/ghc/ghc/-/issues/21970
WORKDIR /work/ghc-$GHC_VERSION/_build/bindist/ghc-$GHC_VERSION-riscv64-linux-gnu
RUN ./configure --target=riscv64-linux-gnu --prefix=/opt/ghc CC=riscv64-linux-gnu-gcc CXX=riscv64-linux-gnu-g++ RANLIB=riscv64-linux-gnu-ranlib && \
  make install

FROM ubuntu:22.04

RUN apt-get update && apt-get install -y git llvm-15 gcc-riscv64-linux-gnu g++-riscv64-linux-gnu

COPY --from=0 /opt /opt
COPY --from=0 /root/.ghcup/bin/cabal /root/.ghcup/bin/cabal
# cabal 3.10 uses $HOME/.local and $HOME/.config and $HOME/.cache instead of $HOME/.cabal
COPY --from=0 /root/.local/bin/* /root/.local/bin/
COPY --from=0 /root/.config/cabal /root/.config/cabal
COPY --from=0 /root/.cache/cabal /root/.cache/cabal
ENV PATH="/opt/ghc/bin:/root/.local/bin:/root/.ghcup/bin:$PATH"
RUN sed -i -e 's|-- with-compiler:|with-compiler: /opt/ghc/bin/riscv64-linux-gnu-ghc|g' -e 's|-- with-hc-pkg:|with-hc-pkg: /opt/ghc/bin/riscv64-linux-gnu-ghc-pkg|g' -e 's|-- hsc2hs-location:|hsc2hs-location: /opt/ghc/bin/riscv64-linux-gnu-hsc2hs|g' /root/.config/cabal/config
CMD ["/opt/ghc/bin/riscv64-linux-gnu-ghc"]
