# Dockerfile for GHC

## JavaScript backend

### Build

```
$ git clone https://github.com/minoki/ghc-docker.git
$ cd ghc-docker/js
$ docker build -t ghc-9.6.0.20230302-js .
```

### Run

GHC (`javascript-unknown-ghcjs-ghc`) and cabal-install (`cabal`) are available.

```
$ docker run --rm ghc-9.6.0.20230302-js javascript-unknown-ghcjs-ghc --version
The Glorious Glasgow Haskell Compilation System, version 9.6.0.20230302
$ docker run --rm ghc-9.6.0.20230302-js cabal --version
cabal-install version 3.8.1.0
compiled using version 3.8.1.0 of the Cabal library
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.6.0.20230302-js javascript-unknown-ghcjs-ghc hello.hs
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.6.0.20230302-js cabal build
```
