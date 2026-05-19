# Rathole

Rust reverse proxy для NAT traversal. В DomKloud он нужен как туннель между облаком и домашним устройством, но это отдельный Rust-проект, не Go/Laravel/Flutter код.

## Как работать

- Пиши по-русски и объясняй просто.
- Rust зафиксирован в `rust-toolchain` (`1.71.0`). Не поднимай версию без прямого запроса.
- Не меняй сетевой протокол, TLS feature matrix или релизный pipeline «заодно».
- `Cargo.lock` здесь коммитится намеренно.
- Не добавляй `unsafe` и тяжёлые зависимости без необходимости.

## Быстрый вход в код

- `src/main.rs` -> `src/lib.rs::run` - entrypoint.
- `src/client.rs`, `src/server.rs` - два режима одного бинарника.
- `src/config.rs` - TOML-конфиг, `MaskedString`, `serde(deny_unknown_fields)`.
- `src/config_watcher.rs` - hot-reload.
- `src/protocol.rs` - wire protocol и `CURRENT_PROTO_VERSION`.
- `src/transport/` - tcp/tls/noise/websocket.
- `src/api.rs` - runtime REST API feature.
- `tests/integration_test.rs`, `tests/common/`, `tests/for_tcp/`, `tests/for_udp/` - интеграционные сценарии.

## Опасные места

- `native-tls`/`websocket-native-tls` и `rustls`/`websocket-rustls` взаимоисключающие.
- При добавлении config-поля обновляй structs в `config.rs`, valid/invalid config tests и логику hot-reload, если поле влияет на runtime.
- Секреты должны оставаться в `MaskedString`; не печатай token/default_token в логах.
- Breaking change в `Hello`, `Auth`, `ControlChannelCmd`, `DataChannelCmd` требует осознанного bump protocol version.
- В async-код не добавлять блокирующий I/O.

## Проверки

```bash
cargo fmt --all -- --check
cargo clippy -- -D warnings
cargo build
cargo test --verbose
cargo test --verbose --no-default-features \
  --features server,client,rustls,noise,websocket-rustls,hot-reload
```

Feature matrix проверяется через `cargo hack`:

```bash
cargo hack check --feature-powerset --no-dev-deps \
  --mutually-exclusive-features default,native-tls,websocket-native-tls,rustls,websocket-rustls
```
