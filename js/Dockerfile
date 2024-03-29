# syntax=docker/dockerfile:1
FROM ubuntu:22.04

# emsdk_env.sh needs bash
SHELL ["/bin/bash", "-c"]

# Needed because GHC's utils/genprimopcode/Main.hs contains non-ASCII character
ENV LANG=C.UTF-8

# emsdk requires python
RUN apt-get update && apt-get install -y build-essential curl git libgmp-dev libnuma1 autoconf python3

ENV BOOTSTRAP_HASKELL_NONINTERACTIVE=1
ENV BOOTSTRAP_HASKELL_GHC_VERSION=9.2.8
ENV BOOTSTRAP_HASKELL_CABAL_VERSION=3.10.1.0
ENV BOOTSTRAP_HASKELL_INSTALL_NO_STACK=1
RUN curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
# cabal 3.10 uses $HOME/.local and $HOME/.config and $HOME/.cache instead of $HOME/.cabal
ENV PATH=/root/.local/bin:/root/.ghcup/bin:$PATH
RUN cabal update && cabal install alex happy

WORKDIR /opt
RUN git clone --depth 1 https://github.com/emscripten-core/emsdk.git
WORKDIR /opt/emsdk
RUN ./emsdk install latest && ./emsdk activate latest

ARG GHC_VERSION=9.6.3

WORKDIR /work
RUN curl -LO https://downloads.haskell.org/~ghc/$GHC_VERSION/ghc-$GHC_VERSION-src.tar.xz && tar -xJf ghc-$GHC_VERSION-src.tar.xz

# See https://gitlab.haskell.org/ghc/ghc/-/wikis/javascript-backend/building
WORKDIR /work/ghc-$GHC_VERSION
RUN source /opt/emsdk/emsdk_env.sh && emconfigure ./configure --target=javascript-unknown-ghcjs
RUN hadrian/build --flavour=quick --bignum=native --docs=none -j binary-dist-dir
# 'hadrian/build install' cannot be used because of https://gitlab.haskell.org/ghc/ghc/-/issues/21970
WORKDIR /work/ghc-$GHC_VERSION/_build/bindist/ghc-$GHC_VERSION-javascript-unknown-ghcjs
RUN source /opt/emsdk/emsdk_env.sh && emconfigure ./configure --target=javascript-unknown-ghcjs --host=$(uname -m)-linux-gnu --prefix=/opt/ghc && \
 make install

FROM ubuntu:22.04

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
