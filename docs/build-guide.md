# Build Guide

This is for those who want to build `rathole` themselves, possibly because the need of latest features or the minimal binary size.

## Build

To use default build settings, run:

```sh
cargo build --release
```

You may need to pre-install [openssl](https://docs.rs/openssl/latest/openssl/index.html) dependencies in Unix-like systems.

## Customize the Build

`rathole` comes with lots of *crate features* that determine whether a certain feature will be compiled or not. Supported features can be checked out in `[features]` of [Cargo.toml](../Cargo.toml).

For example, to build `rathole` with the `client` and `noise` feature:

```sh
cargo build --release --no-default-features --features client,noise
```

## Rustls Support

`rathole` provides optional `rustls` support. It's an almost drop-in replacement of `native-tls` support. (See [Transport](transport.md) for more information.)

To enable this, disable the default features and enable `rustls` feature. And for websocket feature, enable `websocket-rustls` feature as well.

You can also use command line option for this. For example, to replace all default features with `rustls`:

```sh
cargo build --release --no-default-features --features server,client,rustls,noise,websocket-rustls,hot-reload
```

Feature `rustls` and `websocket-rustls` cannot be enabled with `native-tls` and `websocket-native-tls` at the same time, as they are mutually exclusive. Enabling both will result in a compile error.

(Note that default features contains `native-tls` and `websocket-native-tls`.)

## Minimalize the binary

1. Build with the `minimal` profile

The `release` build profile optimize for the program running time, not the binary size.

However, the `minimal` profile enables lots of optimization for the binary size to produce a much smaller binary.

For example, to build `rathole` with `client` feature with the `minimal` profile:

```sh
cargo build --profile minimal --no-default-features --features client
```

2. `strip` and `upx`

The binary that step 1 produces can be even smaller, by using `strip` and `upx` to remove the symbols and compress the binary.

Like:

```sh
strip rathole
upx --best --lzma rathole
```

At the time of writting the build guide, the produced binary for `x86_64-unknown-linux-glibc` has the size of **574 KiB**, while `frpc` has the size of **~10 MiB**, which is much larger.

## Cross-compilation (macOS → Linux)

Cross-compiling for Linux from macOS is supported via [`cross`](https://github.com/cross-rs/cross) — a tool that uses Docker under the hood to build for foreign targets.

Install `cross`:
```sh
cargo install cross
```

Build for Linux targets:
```sh
# x86_64 (amd64)
cross build --release --target x86_64-unknown-linux-musl --features embedded --no-default-features

# ARM64 (aarch64)
cross build --release --target aarch64-unknown-linux-musl --features embedded --no-default-features

# ARMv7 (armhf)
cross build --release --target armv7-unknown-linux-musleabihf --features embedded --no-default-features
```

All cross-built binaries use the musl-stdlib, ensuring compatibility across Linux distributions. TLS features are excluded in the `embedded` profile because cross-compiling OpenSSL is difficult.

### Docker multi-platform build

To build multi-platform Docker images from macOS:
```sh
docker buildx create --name cross-builder --driver docker-container --use
docker buildx build --platform linux/arm64,linux/arm/v7,linux/amd64 -t rathole:multi --push .
```

Requires Docker Desktop with QEMU emulation for non-native architectures (slower on Intel Mac, native on Apple Silicon for arm64).
