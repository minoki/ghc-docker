# syntax=docker/dockerfile:1
FROM ubuntu:22.04

# emsdk_env.sh needs bash
SHELL ["/bin/bash", "-c"]

# Needed because GHC's utils/genprimopcode/Main.hs contains non-ASCII character
ENV LANG=C.UTF-8

# emsdk requires python
RUN apt-get update && apt-get install -y build-essential curl git libgmp-dev libnuma1 autoconf python3

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

WORKDIR /opt
RUN git clone --depth 1 https://github.com/emscripten-core/emsdk.git
WORKDIR /opt/emsdk
RUN ./emsdk install 3.1.31 && ./emsdk activate 3.1.31

ARG GHC_VERSION=9.8.4

WORKDIR /work
ADD --checksum=sha256:17e8188f3c8a5c2f73fb4e35d01032e8dc258835ec876d52c8ad8ee3d24b2fc5 https://downloads.haskell.org/~ghc/$GHC_VERSION/ghc-$GHC_VERSION-src.tar.xz ghc-$GHC_VERSION-src.tar.xz
RUN tar -xJf ghc-$GHC_VERSION-src.tar.xz
WORKDIR /work/ghc-$GHC_VERSION

# If building with GHC 9.6.7: Workaround https://gitlab.haskell.org/ghc/ghc/-/issues/25963
# The patch should be taken from 189a368ba74dbfe8a6e832fe0f9ffe56cd565824
# ADD 0001-compiler-Fix-CPP-guards-around-ghc_unique_counter64.patch .
# RUN git apply 0001-compiler-Fix-CPP-guards-around-ghc_unique_counter64.patch

# See https://gitlab.haskell.org/ghc/ghc/-/wikis/javascript-backend/building
RUN source /opt/emsdk/emsdk_env.sh && emconfigure ./configure --target=javascript-unknown-ghcjs
RUN hadrian/build --flavour=quick --bignum=native --docs=none -j binary-dist-dir
# 'hadrian/build install' cannot be used because of https://gitlab.haskell.org/ghc/ghc/-/issues/21970
WORKDIR /work/ghc-$GHC_VERSION/_build/bindist/ghc-$GHC_VERSION-javascript-unknown-ghcjs
RUN source /opt/emsdk/emsdk_env.sh && emconfigure ./configure --target=javascript-unknown-ghcjs --host=$(uname -m)-linux-gnu --prefix=/opt/ghc && \
 make install

FROM ubuntu:22.04

# Some cabal commands may need git to be installed
RUN apt-get update && apt-get install -y python3 git

COPY --from=0 /opt/emsdk /opt/emsdk
COPY --from=0 /opt/ghc /opt/ghc
COPY --from=0 /root/.ghcup/bin/cabal /root/.ghcup/bin/cabal
# cabal 3.10 uses $HOME/.local and $HOME/.config and $HOME/.cache instead of $HOME/.cabal
COPY --from=0 /root/.local/bin/* /root/.local/bin/
COPY --from=0 /root/.config/cabal /root/.config/cabal
COPY --from=0 /root/.cache/cabal /root/.cache/cabal
ENV PATH="/opt/ghc/bin:/root/.local/bin:/root/.ghcup/bin:$PATH"
RUN sed -i -e 's|-- with-compiler:|with-compiler: /opt/ghc/bin/javascript-unknown-ghcjs-ghc|g' -e 's|-- with-hc-pkg:|with-hc-pkg: /opt/ghc/bin/javascript-unknown-ghcjs-ghc-pkg|g' -e 's|-- hsc2hs-location:|hsc2hs-location: /opt/ghc/bin/javascript-unknown-ghcjs-hsc2hs|g' /root/.config/cabal/config
CMD ["/opt/ghc/bin/javascript-unknown-ghcjs-ghc"]
