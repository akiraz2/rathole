# AGENTS.md

Краткие правила для AI-агентов, работающих в этом репозитории. Перед любыми изменениями убедитесь, что выполняете базовые требования из этого файла.

## Что это за проект

`rathole` — обратный прокси для NAT traversal на Rust (асинхронный, на `tokio`). Один и тот же бинарник работает как **сервер** или как **клиент** — режим определяется флагом CLI (`-s`/`-c`) либо содержимым TOML-конфига (см. `examples/minimal/`).

Точки входа: `src/main.rs` → `src/lib.rs::run` → `src/{client,server}.rs`. Транспорты живут в `src/transport/` (`tcp`, `tls` через `native-tls`/`rustls`, `noise`, `websocket`). Парсинг и валидация конфига — в `src/config.rs`, hot-reload — в `src/config_watcher.rs`, протокол по управляющему каналу — в `src/protocol.rs`.

## Toolchain и форматирование

- Rust зафиксирован в `rust-toolchain` (сейчас `1.71.0`). **Не повышать** без явного запроса — иначе ломается воспроизводимость CI.
- Edition 2021. Перед коммитом обязательно `cargo fmt` — учитывайте `.rustfmt.toml` (`imports_granularity = "module"`: импорты группируются по модулю, не по элементу).
- Не использовать API, требующее более нового Rust, чем зафиксированный в `rust-toolchain`.

## Cargo features — критично

Список и состав фич — в `[features]` файла `Cargo.toml`. Дефолт: `server, client, native-tls, noise, websocket-native-tls, hot-reload`.

Жёсткое правило: **`native-tls`/`websocket-native-tls` и `rustls`/`websocket-rustls` взаимоисключающие** — одновременно не включать, иначе compile error. CI это проверяет через `cargo hack` (см. ниже).

При добавлении нового опционального кода:
- Заворачивайте его в `#[cfg(feature = "...")]`.
- Если фича отсутствует во время выполнения — используйте `crate::helper::feature_not_compile("name")` (см. примеры в `src/lib.rs`).
- Добавляйте новую фичу в `[features]` `Cargo.toml` и (если она подразумевает зависимость) в `--mutually-exclusive-features` в `.github/workflows/rust.yml`, если она конфликтует с другими.

## Команды, которые должны проходить локально

Перед тем как считать задачу законченной, выполните то же, что делает CI (`.github/workflows/rust.yml`):

```sh
cargo fmt --all -- --check
cargo clippy -- -D warnings
cargo build
cargo test --verbose
cargo test --verbose --no-default-features \
  --features server,client,rustls,noise,websocket-rustls,hot-reload
```

И матрица фич (требуется `cargo install cargo-hack`):

```sh
cargo hack check --feature-powerset --no-dev-deps \
  --mutually-exclusive-features default,native-tls,websocket-native-tls,rustls,websocket-rustls
```

Сборки должны проходить на Linux (gnu), Windows (msvc), macOS (x86_64 и aarch64) — не вносите платформо-специфичный код без `#[cfg(target_os = ...)]`.

## Тесты

- Юнит-тесты живут рядом с кодом в `#[cfg(test)] mod tests`.
- Интеграционные тесты — `tests/integration_test.rs` + общая инфраструктура в `tests/common/`. Они поднимают echo/pingpong-серверы и прогоняют рабочие конфиги из `tests/for_tcp/` и `tests/for_udp/` для каждого транспорта.
- Тесты конфигов: `tests/config_test/{valid_config,invalid_config}/`. Любой новый конфигурационный параметр в `src/config.rs` должен сопровождаться примером в `valid_config/` и (где уместно) кейсом в `invalid_config/`.
- Включить подробные логи теста: `RUST_LOG=debug cargo test -- --nocapture`.
- Когда добавляете новый транспорт — добавьте парные TOML-файлы в `tests/for_tcp/` и `tests/for_udp/` и подключите их в `integration_test.rs` под соответствующим `#[cfg(feature = ...)]`.

## Конфигурация и протокол — что надо знать

- Используется `serde` с `#[serde(deny_unknown_fields)]` — добавление поля в TOML без объявления его в структуре приводит к ошибке валидации. Это намеренно, не отключайте.
- Секреты (`token`, `default_token`) хранятся в `MaskedString` (`src/config.rs`) — её `Debug` печатает `MASKED`. Никогда не печатайте секреты через `{:?}` напрямую и не превращайте `MaskedString` в `String` для логов.
- Версия wire-протокола — `CURRENT_PROTO_VERSION` в `src/protocol.rs`. Любое breaking-change в `Hello`/`Auth`/`ControlChannelCmd`/`DataChannelCmd` требует bump версии и аккуратной обратной совместимости — это сетевой протокол, существующие клиенты в дикой природе.
- Имена сервисов в логах хешируются (см. `protocol::digest`) — не «исправляйте» это на читаемые строки.

## Hot-reload

Включается фичей `hot-reload`. `ConfigWatcherHandle` (`src/config_watcher.rs`) различает `General` и сервисные изменения. Изменение глобальных полей (`bind_addr`, `transport`, токен по умолчанию и т.п.) приводит к **полному рестарту** инстанса; добавление/удаление/правка services — к точечным апдейтам. Если меняете `Config`-структуры, проверьте, в какую категорию попадает поле в `config_watcher.rs`, иначе hot-reload будет пропускать изменения.

## Стиль логов

Используется `tracing`. Уровни: `error!` — невосстановимое, `warn!` — деградация, `info!` — события lifecycle (старт/стоп/connect), `debug!` — детали для отладки, `trace!` — каждое сообщение протокола. Не логируйте на `info` в горячем пути на каждое соединение/пакет.

## Зависимости

Не добавляйте зависимости без необходимости — у проекта таргет на embedded-сборки (фича `embedded`, минимальный профиль `[profile.minimal]`). Если зависимость нужна только под фичу — помечайте её `optional = true` и подключайте через `[features]`. Не обновляйте мажорные версии существующих крейтов «заодно».

## Docker и релизы

`Dockerfile` принимает build-arg `FEATURES` (`docker build --build-arg FEATURES=client,noise .`). Финальный образ — distroless, бинарь запускается от UID 1000 — не используйте пути, требующие root.

Релизный workflow — `.github/workflows/release.yml`. Версия в `Cargo.toml` (`version = ...`) — единственный источник истины для релиза, тег должен ей соответствовать.

## Чего не делать

- Не коммитить `target/`, `Cargo.lock` для бинарного крейта **коммитим** (он уже в репо — это намеренно).
- Не добавлять `unsafe` без обсуждения.
- Не менять лицензию или заголовки `LICENSE` (Apache-2.0).
- Не править README напрямую под мелкие изменения кода — README ориентирован на конечного пользователя, технические детали идут в `docs/`.
- Не вводить блокирующий I/O в async-коде — всё на `tokio`, используйте его примитивы.
