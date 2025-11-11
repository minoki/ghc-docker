# syntax=docker/dockerfile:1
FROM ubuntu:24.04

# Needed because GHC's utils/genprimopcode/Main.hs contains non-ASCII character
ENV LANG=C.UTF-8

RUN apt-get update && apt-get install -y build-essential curl git libgmp-dev libnuma1 autoconf llvm-15 gcc-riscv64-linux-gnu g++-riscv64-linux-gnu

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

ARG GHC_VERSION=9.8.4

WORKDIR /work
ADD --checksum=sha256:17e8188f3c8a5c2f73fb4e35d01032e8dc258835ec876d52c8ad8ee3d24b2fc5 https://downloads.haskell.org/~ghc/$GHC_VERSION/ghc-$GHC_VERSION-src.tar.xz ghc-$GHC_VERSION-src.tar.xz
RUN tar -xJf ghc-$GHC_VERSION-src.tar.xz
WORKDIR /work/ghc-$GHC_VERSION

# If building with GHC 9.6.7: Workaround https://gitlab.haskell.org/ghc/ghc/-/issues/25963
# The patch should be taken from 189a368ba74dbfe8a6e832fe0f9ffe56cd565824
# ADD 0001-compiler-Fix-CPP-guards-around-ghc_unique_counter64.patch .
# RUN git apply 0001-compiler-Fix-CPP-guards-around-ghc_unique_counter64.patch

RUN ./configure --target=riscv64-linux-gnu
RUN hadrian/build --bignum=native -j binary-dist-dir
# 'hadrian/build install' cannot be used because of https://gitlab.haskell.org/ghc/ghc/-/issues/21970
WORKDIR /work/ghc-$GHC_VERSION/_build/bindist/ghc-$GHC_VERSION-riscv64-linux-gnu
RUN ./configure --target=riscv64-linux-gnu --prefix=/opt/ghc CC=riscv64-linux-gnu-gcc CXX=riscv64-linux-gnu-g++ RANLIB=riscv64-linux-gnu-ranlib && \
  make install

FROM ubuntu:24.04

# Some cabal commands may need git to be installed
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
