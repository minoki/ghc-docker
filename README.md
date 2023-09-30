# Dockerfiles for GHC

## Checkout

```
$ git clone -b ghc-9.8 https://github.com/minoki/ghc-docker.git
$ cd ghc-docker
```

## JavaScript backend

### Build

```
$ docker build -t ghc-9.8.0.20230929-js js
```

### Run

GHC (`javascript-unknown-ghcjs-ghc`) and cabal-install (`cabal`) are available.

```
$ docker run --rm ghc-9.8.0.20230929-js javascript-unknown-ghcjs-ghc --version
The Glorious Glasgow Haskell Compilation System, version 9.8.0.20230929
$ docker run --rm ghc-9.8.0.20230929-js cabal --version
cabal-install version 3.10.1.0
compiled using version 3.10.1.0 of the Cabal library 
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.8.0.20230929-js javascript-unknown-ghcjs-ghc hello.hs
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.8.0.20230929-js cabal build
```

## WebAssembly backend (`wasm32-wasi`)

### Build

```
$ docker build -t ghc-9.8.0.20230929-wasm32-wasi wasm32-wasi
```

Note: In addition to GHC, LLVM is built during `docker build`.

### Run

GHC (`wasm32-wasi-ghc`) and cabal-install (`cabal`) are available.

```
$ docker run --rm ghc-9.8.0.20230929-wasm32-wasi wasm32-wasi-ghc --version
The Glorious Glasgow Haskell Compilation System, version 9.8.0.20230929
$ docker run --rm ghc-9.8.0.20230929-wasm32-wasi cabal --version
cabal-install version 3.10.1.0
compiled using version 3.10.1.0 of the Cabal library 
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.8.0.20230929-wasm32-wasi wasm32-wasi-ghc hello.hs
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.8.0.20230929-wasm32-wasi cabal build
```

## RISC-V (`riscv64-linux-gnu`)

### Build

```
$ docker build -t ghc-9.8.0.20230929-riscv64-linux-gnu riscv64-linux-gnu
```

### Run

GHC (`riscv64-linux-gnu-ghc`) and cabal-install (`cabal`) are available.

```
$ docker run --rm ghc-9.8.0.20230929-riscv64-linux-gnu riscv64-linux-gnu-ghc --version
The Glorious Glasgow Haskell Compilation System, version 9.8.0.20230929
$ docker run --rm ghc-9.8.0.20230929-riscv64-linux-gnu cabal --version
cabal-install version 3.10.1.0
compiled using version 3.10.1.0 of the Cabal library 
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.8.0.20230929-riscv64-linux-gnu riscv64-linux-gnu-ghc hello.hs
$ docker run --rm -v "$(pwd)":/work -w /work ghc-9.8.0.20230929-riscv64-linux-gnu cabal build
```
