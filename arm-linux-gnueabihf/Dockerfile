# syntax=docker/dockerfile:1
FROM ubuntu:22.04

# Needed because GHC's utils/genprimopcode/Main.hs contains non-ASCII character
ENV LANG=C.UTF-8

RUN apt-get update && apt-get install -y build-essential curl git libgmp-dev libnuma1 autoconf llvm-15 gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf

ENV BOOTSTRAP_HASKELL_NONINTERACTIVE=1
ENV BOOTSTRAP_HASKELL_GHC_VERSION=9.2.8
ENV BOOTSTRAP_HASKELL_CABAL_VERSION=3.10.1.0
ENV BOOTSTRAP_HASKELL_INSTALL_NO_STACK=1
RUN curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
# cabal 3.10 uses $HOME/.local and $HOME/.config and $HOME/.cache instead of $HOME/.cabal
ENV PATH=/root/.local/bin:/root/.ghcup/bin:$PATH
RUN cabal update && cabal install alex happy

ARG GHC_VERSION=9.6.2

WORKDIR /work
RUN curl -LO https://downloads.haskell.org/~ghc/$GHC_VERSION/ghc-$GHC_VERSION-src.tar.xz && tar -xJf ghc-$GHC_VERSION-src.tar.xz
WORKDIR /work/ghc-$GHC_VERSION

RUN ./configure --target=arm-linux-gnueabihf
RUN hadrian/build --bignum=native -j binary-dist-dir
# 'hadrian/build install' cannot be used because of https://gitlab.haskell.org/ghc/ghc/-/issues/21970
#WORKDIR /work/ghc-$GHC_VERSION/_build/bindist/ghc-$GHC_VERSION-arm-linux-gnueabihf
#RUN ./configure --target=arm-linux-gnueabihf --host=$(uname -m)-linux-gnu --prefix=/opt/ghc && \
#  make install
