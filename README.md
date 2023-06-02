# Dockerfiles for GHC

## Checkout

```
$ git clone https://github.com/minoki/ghc-docker.git
$ cd ghc-docker
```

## JavaScript backend

### Build

```
$ docker build -t ghc-9.6.2-js js
```

### Run

GHC (`javascript-unknown-ghcjs-ghc`) and cabal-install (`cabal`) are available.

```
$ docker run --rm ghc-9.6.2-js javascript-unknown-ghcjs-ghc --version
The Glorious Glasgow Haskell Compilation System, version 9.6.2
$ docker run --rm ghc-9.6.2-js cabal --version
cabal-install version 3.10.1.0
compiled using version 3.10.1.0 of the Cabal library 
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.6.2-js javascript-unknown-ghcjs-ghc hello.hs
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.6.2-js cabal build
```

## WebAssembly backend (`wasm32-wasi`)

### Build

```
$ docker build -t ghc-9.6.2-wasm32-wasi wasm32-wasi
```

Note: In addition to GHC, LLVM is built during `docker build`.

### Run

GHC (`wasm32-wasi-ghc`) and cabal-install (`cabal`) are available.

```
$ docker run --rm ghc-9.6.2-wasm32-wasi wasm32-wasi-ghc --version
The Glorious Glasgow Haskell Compilation System, version 9.6.2
$ docker run --rm ghc-9.6.2-wasm32-wasi cabal --version
cabal-install version 3.10.1.0
compiled using version 3.10.1.0 of the Cabal library 
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.6.2-wasm32-wasi wasm32-wasi-ghc hello.hs
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.6.2-wasm32-wasi cabal build
```
