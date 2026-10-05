# Прикладные домены Rust

Справочник по основным областям применения Rust (состояние на октябрь 2026, стабильный Rust 1.90+, edition 2024): для каждого домена — архитектурные и производительные правила, рекомендуемые крейты (в скобках — актуальная мажорная/минорная линия на октябрь 2026) и проверенные ресурсы. Открывайте нужный раздел, когда задача касается конкретного домена; общие идиомы и тулчейн описаны в [idioms-patterns.md](idioms-patterns.md) и [toolchain.md](toolchain.md).

- [1. Web-бэкенд и сервисы](#1-web-бэкенд-и-сервисы)
- [2. CLI-утилиты и TUI](#2-cli-утилиты-и-tui)
- [3. Библиотеки и публичные крейты](#3-библиотеки-и-публичные-крейты)
- [4. Embedded и no_std](#4-embedded-и-no_std)
- [5. WebAssembly и фронтенд](#5-webassembly-и-фронтенд)
- [6. Interop и FFI](#6-interop-и-ffi)
- [7. Данные, ML и численные методы](#7-данные-ml-и-численные-методы)
- [8. Desktop и GUI](#8-desktop-и-gui)
- [9. Разработка игр](#9-разработка-игр)
- [10. Системное программирование, сеть, ОС](#10-системное-программирование-сеть-ос)

---

## 1. Web-бэкенд и сервисы

### Ключевые выводы
- По умолчанию берите axum поверх tokio: он построен на `tower::Service`, поэтому таймауты, сжатие, CORS, трассировка и лимиты подключаются готовыми слоями из `tower-http`. actix-web — зрелая и быстрая альтернатива со своей экосистемой middleware; на ней написана книга Zero To Production.
- Держите хендлеры тонкими: извлечь данные → вызвать доменный сервис → преобразовать результат в ответ. Доменный слой не должен зависеть от типов axum/actix; репозитории описывайте трейтами, чтобы тестировать логику без HTTP и без БД.
- Ошибки: доменные ошибки — `enum` на `thiserror`; на границе HTTP один тип `AppError` с `impl IntoResponse` (или `ResponseError` в actix), который сопоставляет варианты со статус-кодами. Внутренние детали логируйте через `tracing`, клиенту отдавайте обезличенное сообщение. `anyhow` уместен только как «прочая внутренняя ошибка» → 500.
- БД: `sqlx` даёт проверку SQL на этапе компиляции (`query!`/`query_as!`); для CI без живой базы используйте офлайн-режим (`cargo sqlx prepare`, каталог `.sqlx` в репозитории), для интеграционных тестов — `#[sqlx::test]` (отдельная БД на тест). Diesel — типобезопасный DSL (асинхронно через `diesel-async`), SeaORM — динамическая ORM. Пул соединений кладите в `State`, не открывайте соединение на запрос.
- Не блокируйте исполнитель: CPU-тяжёлое (хеширование паролей `argon2`, сжатие, парсинг больших файлов) — в `tokio::task::spawn_blocking` или rayon; не держите `std::sync::Mutex` через `.await`. Ставьте `TimeoutLayer` и лимит размера тела запроса на весь роутер.
- Наблюдаемость с первого дня: `tracing` + `tracing-subscriber` (`EnvFilter`, JSON-формат в проде), `#[instrument]` на сервисных функциях, `TraceLayer` из `tower-http`. Экспорт в OpenTelemetry — через `tracing-opentelemetry` + `opentelemetry-otlp`; версии всех `opentelemetry*`-крейтов держите синхронными, они меняются согласованно.
- Конфигурация: слои «файл → переменные окружения» десериализуются в типизированную структуру (`config` или `figment`), валидация при старте, секреты — в `secrecy::SecretString`, чтобы не попадали в `Debug`-логи.
- Graceful shutdown: `axum::serve(listener, app).with_graceful_shutdown(signal)`, где `signal` ждёт `ctrl_c` и SIGTERM (контейнеры шлют именно его). Фоновые задачи останавливайте через `CancellationToken` и дожидайтесь через `TaskTracker` (`tokio-util`). Для тестов поднимайте приложение на `127.0.0.1:0` и бейте реальным HTTP-клиентом.

### Крейты
- `axum` (0.8) — роутер и экстракторы на tower/hyper; `axum-extra` — куки, typed headers и прочее.
- `tower` (0.5), `tower-http` (0.7) — абстракция `Service` и готовые HTTP-middleware.
- `actix-web` (4.x) — альтернативный зрелый фреймворк.
- `tokio` (1.x), `hyper` (1.x), `reqwest` (0.13) — рантайм, HTTP-ядро, HTTP-клиент.
- `sqlx` (0.9) + `sqlx-cli` — асинхронный SQL с compile-time проверкой и миграциями.
- `diesel` (2.3) + `diesel-async` (0.9) — типобезопасный query builder.
- `sea-orm` (2.0) — асинхронная динамическая ORM.
- `serde` / `serde_json` (1.x) — сериализация.
- `tracing` (0.1), `tracing-subscriber` (0.3), `tracing-opentelemetry` (0.34), `opentelemetry` / `opentelemetry-otlp` (0.33) — логи, спаны, экспорт.
- `config` (0.15), `figment` (0.10; последний релиз 2024, но стабилен) — слоистая конфигурация.
- `thiserror` (2.x), `anyhow` (1.x) — типизированные и «прикладные» ошибки.
- `secrecy` (0.10), `argon2` (0.6) — секреты и хеширование паролей.
- `utoipa` (6.x) — генерация OpenAPI; `garde` / `validator` — валидация входных DTO.
- `tonic` / `prost` (0.14) — gRPC и Protobuf.
- `tokio-util` (0.7) — `CancellationToken`, `TaskTracker`, кодеки.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Zero To Production In Rust](https://www.zero2prod.com/) | Luca Palmieri, 2022 (регулярно обновляется) | Полный путь API-сервиса: actix-web, sqlx, тесты, телеметрия, CI/CD, аутентификация. [Порт на axum](https://github.com/mattiapenati/zero2prod) от сообщества |
| [Код книги](https://github.com/LukeMathWalker/zero-to-production) | Luca Palmieri, живой | Эталонная структура сервиса по главам |
| [Error Handling In Rust — A Deep Dive](https://www.lpalmieri.com/posts/error-handling-rust/) | Luca Palmieri, 2021 | Как проектировать ошибки для сервиса и отображать их в HTTP |
| [axum examples](https://github.com/tokio-rs/axum/tree/main/examples) | tokio-rs, живой | Готовые паттерны: state, ошибки, graceful shutdown, тесты, WebSocket |
| [Tokio tutorial](https://tokio.rs/tokio/tutorial) | Tokio team, живой | База асинхронного рантайма |
| [Graceful Shutdown](https://tokio.rs/tokio/topics/shutdown) | Tokio team, живой | Сигналы → `CancellationToken` → `TaskTracker` |
| [Inventing the Service trait](https://tokio.rs/blog/2021-05-14-inventing-the-service-trait) | Tokio blog, 2021 | Почему middleware в tower устроены именно так |
| [Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/) | Alice Ryhl, 2021 | Паттерн актора для разделяемого состояния без мьютексов |
| [sqlx](https://github.com/transact-rs/sqlx) | sqlx maintainers, живой | Compile-time запросы, офлайн-режим, миграции, `#[sqlx::test]` |
| [OpenTelemetry Rust](https://opentelemetry.io/docs/languages/rust/) | OpenTelemetry, живой | Настройка трассировки и экспорта OTLP |

---

## 2. CLI-утилиты и TUI

### Ключевые выводы
- Аргументы — `clap` с derive-API (`#[derive(Parser)]`, подкоманды через `enum` + `#[derive(Subcommand)]`); автодополнение и man-страницы генерируйте `clap_complete` и `clap_mangen` в `build.rs` или отдельной `xtask`-командой. Для крошечных бинарников без зависимостей — `lexopt`.
- Ошибки: в приложении `anyhow` с `.context("что делали")` на каждом шаге ввода-вывода; `miette` — когда нужна диагностика с подсветкой фрагментов исходника (компиляторы, линтеры, конфиги); `color-eyre` — цветные отчёты с бэктрейсом. Различайте коды выхода (`std::process::ExitCode`), а не только 0/1.
- stdout — для данных, stderr — для логов, прогресса и ошибок. Проверяйте `std::io::IsTerminal`, чтобы отключать цвет и прогресс-бары при выводе в пайп; уважайте `NO_COLOR` (это делают `anstream`/`owo-colors`).
- Большой вывод: заблокируйте `stdout().lock()` и оберните в `BufWriter`. Rust игнорирует SIGPIPE, поэтому `println!` паникует при закрытом пайпе (`app | head`); пишите через `writeln!` и обрабатывайте `ErrorKind::BrokenPipe` как штатное завершение.
- TUI: `ratatui` в immediate-режиме (каждый кадр перерисовывается из состояния приложения) с бэкендом `crossterm`. Используйте `ratatui::init()`/`restore()`: они ставят panic hook и возвращают терминал в нормальный режим при панике. Состояние и логика — отдельно от отрисовки, это упрощает тесты.
- Пути: работайте с `PathBuf`/`OsStr`, не со `String` — имена файлов не обязаны быть UTF-8. Если утилита осознанно требует UTF-8, используйте `camino::Utf8PathBuf`. Каталоги конфигов и кешей берите из `dirs`/`directories`/`etcetera`, не хардкодьте `~/.config` и `/`.
- Тесты: `assert_cmd` запускает собранный бинарник и проверяет код выхода и вывод; `trycmd` хранит сценарии как снапшоты (`.toml`/`.md`) и удобен для документации; `assert_fs`/`tempfile` дают временные каталоги; `insta` — для снапшотов структурированного вывода.
- Дистрибуция: `cargo-dist` собирает релизные архивы и инсталляторы под платформы в CI, `cargo-binstall` ставит готовые бинарники; статические Linux-сборки — target `x86_64-unknown-linux-musl`.

### Крейты
- `clap` (4.x), `clap_complete`, `clap_mangen` — парсинг аргументов, автодополнение, man.
- `lexopt` (0.3) — минималистичный парсер без макросов.
- `anyhow` (1.x), `miette` (7.x), `color-eyre` (0.6) — ошибки для приложений.
- `indicatif` (0.18) — прогресс-бары и спиннеры (рисуют в stderr).
- `ratatui` (0.30) + `crossterm` (0.29) — TUI.
- `anstream` (1.x), `owo-colors` (4.x) — цвет с учётом TTY и `NO_COLOR`.
- `dialoguer` (0.12) — интерактивные промпты.
- `human-panic` (2.x) — дружелюбное сообщение о панике для конечных пользователей.
- `assert_cmd` (2.x), `trycmd` (1.x), `snapbox`, `assert_fs`, `tempfile`, `insta` — тестирование.
- `camino` (1.x), `dirs` (7.x), `directories` (6.x), `etcetera` (0.11) — пути и платформенные каталоги.
- `cargo-dist` (0.32), `cargo-binstall` — сборка и установка релизов.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Command Line Applications in Rust](https://rust-cli.github.io/book/) | Rust CLI WG, живой | Короткий официальный путеводитель: аргументы, ошибки, вывод, тесты, пакетирование |
| [Command-Line Rust](https://www.oreilly.com/library/view/command-line-rust/9781098109424/) | Ken Youens-Clark, 2022 (обновлено в 2024 под clap 4) | Клоны Unix-утилит с тестами; [код](https://github.com/kyclark/command-line-rust) |
| [clap derive tutorial](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html) | clap-rs, живой | Каноничный разбор derive-API |
| [Rust CLI recommendations](https://rust-cli-recommendations.sunshowers.io/) | Rain (sunshowers), живой | Мнения практика: структура, версии, конфиги, цвета |
| [Ratatui](https://ratatui.rs/) | Ratatui team, живой | Туториалы, шаблоны приложений, паттерны архитектуры TUI |
| [ripgrep](https://github.com/BurntSushi/ripgrep) | Andrew Gallant, живой | Эталонный промышленный CLI: обработка ошибок, вывод, кроссплатформенность |
| [cross](https://github.com/cross-rs/cross) | cross-rs, живой (последний релиз 2023) | Кросс-компиляция через контейнеры |

---

## 3. Библиотеки и публичные крейты

### Ключевые выводы
- Следуйте Rust API Guidelines: стандартные трейты (`Debug`, `Clone`, `PartialEq`, `Send`/`Sync` там, где возможно), конверсии через `From`/`TryFrom`, предсказуемые имена (`as_`/`to_`/`into_`).
- Публичный API эволюционирует безопасно: `#[non_exhaustive]` на публичных `enum` и структурах, sealed-трейты для трейтов, которые не должны реализовываться снаружи; поля структур — приватные с конструкторами/билдерами.
- В публичном API — собственный тип ошибки (`thiserror`), не `anyhow::Error` и не `Box<dyn Error>`; типы публичных зависимостей становятся частью вашего API, их мажорный апгрейд — ваш мажорный апгрейд.
- SemVer проверяйте автоматически: `cargo-semver-checks` в CI перед `cargo publish`; `cargo-public-api` показывает дифф публичного API в ревью.
- Features должны быть аддитивными (включение не ломает код); не делайте feature, отключающую функциональность, — используйте `std` по умолчанию + `default-features = false` для no_std. Матрицу проверяйте `cargo hack --feature-powerset`.
- MSRV: объявите `rust-version` в `Cargo.toml` и проверяйте его отдельной CI-задачей; MSRV-aware resolver (стабилен с Rust 1.84, по умолчанию с `resolver = "3"`/edition 2024) подбирает зависимости, совместимые с вашим MSRV.
- Документация — часть API: `//!` с примером на уровне крейта, примеры как doctest, `#![warn(missing_docs)]`, для docs.rs — `[package.metadata.docs.rs] all-features = true`.
- Подробности по тулчейну (clippy, CI, профили) и идиомам — в [toolchain.md](toolchain.md) и [idioms-patterns.md](idioms-patterns.md).

### Крейты
- `thiserror` (2.x) — типы ошибок библиотеки.
- `cargo-semver-checks` (0.51) — линтер нарушений SemVer.
- `cargo-public-api` (0.52) — дифф публичного API.
- `cargo-hack` (0.6) — проверка комбинаций features.
- `cargo-msrv` (0.19) — поиск и проверка MSRV.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) | Rust libs team, живой | Чек-лист дизайна публичного API |
| [SemVer Compatibility](https://doc.rust-lang.org/cargo/reference/semver.html) | Cargo book, живой | Что считается breaking change в Rust |
| [Features](https://doc.rust-lang.org/cargo/reference/features.html) | Cargo book, живой | Аддитивность, `dep:`, взаимодействие с резолвером |
| [rust-version](https://doc.rust-lang.org/cargo/reference/rust-version.html) | Cargo book, живой | Объявление MSRV; см. также [Rust 1.84](https://blog.rust-lang.org/2025/01/09/Rust-1.84.0/) про MSRV-aware resolver |
| [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks) | Predrag Gruevski, живой | Автоматическая проверка SemVer |
| [Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/) | Microsoft, 2025 | Промышленные правила для библиотек и сервисов |
| [Effective Rust](https://effective-rust.com/) | David Drysdale, 2024 | Главы про API, зависимости, features, документацию |

---

## 4. Embedded и no_std

### Ключевые выводы
- Слои: PAC (сгенерирован `svd2rust` из SVD) → HAL (безопасные драйверы периферии) → BSP (конкретная плата). Драйверы внешних устройств пишите обобщёнными по трейтам `embedded-hal` 1.0 / `embedded-hal-async` — так они работают с любым HAL. Разделяемые шины SPI/I2C — через `embedded-hal-bus`.
- Выбор модели исполнения: Embassy — async-исполнитель без кучи (задачи статически аллоцированы), HAL для STM32, nRF, RP2040/RP235x, `embassy-time` для таймеров; RTIC 2 — задачи на аппаратных приоритетах прерываний с compile-time-гарантией отсутствия дедлоков (SRP), удобен для жёсткого реального времени. ESP32 — `esp-hal` (1.x).
- Память: без кучи по умолчанию; коллекции фиксированной ёмкости — `heapless`; `'static`-объекты для задач — `static_cell`; кросс-платформенные критические секции — `critical-section`. `alloc` подключайте осознанно и с явным аллокатором.
- Отладка и прошивка: `probe-rs` (`probe-rs run` как runner в `.cargo/config.toml`) вместо связки OpenOCD+GDB; логирование — `defmt` (форматирование откладывается на хост, бинарник почти не растёт) + `panic-probe`.
- Размер и детерминизм: `opt-level = "s"`/`"z"`, `lto = true`, `codegen-units = 1`, `panic = "abort"`; избегайте `core::fmt` в горячих путях — он заметно раздувает прошивку.
- Логику, не зависящую от железа (протоколы, парсеры, конечные автоматы), выносите в отдельный `no_std`-крейт и тестируйте `cargo test` на хосте; на целевом устройстве — `defmt-test`/`embedded-test`. Сериализация для каналов связи — `postcard`.

### Крейты
- `embedded-hal` / `embedded-hal-async` (1.0), `embedded-hal-bus` (0.3), `embedded-io` (0.7) — трейты драйверов и ввода-вывода.
- `embassy-executor` (0.10), `embassy-time` (0.5), `embassy-stm32` / `embassy-nrf` / `embassy-rp` — async-фреймворк и HAL.
- `rtic` (2.x) — фреймворк реального времени на прерываниях.
- `esp-hal` (1.x) — HAL для ESP32-семейства.
- `cortex-m-rt` (0.7) — рантайм и вектор прерываний Cortex-M.
- `probe-rs-tools` (0.32) — прошивка, отладка, RTT.
- `defmt` (1.x) — компактное логирование.
- `heapless` (0.9), `static_cell` (2.x), `critical-section` (1.x), `postcard` (1.x) — память, синхронизация, сериализация.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The Embedded Rust Book](https://docs.rust-embedded.org/book/) | Rust Embedded WG, живой | Базовый курс: no_std, периферия, прерывания, конкурентность |
| [Discovery (micro:bit v2)](https://docs.rust-embedded.org/discovery-mb2/) | Rust Embedded WG, живой | Практика на дешёвой плате для новичков |
| [The Embedonomicon](https://docs.rust-embedded.org/embedonomicon/) | Rust Embedded WG, живой | Как устроены рантайм, линкер-скрипты, вектор прерываний |
| [embedded-hal v1.0](https://blog.rust-embedded.org/embedded-hal-v1/) | Rust Embedded WG, 2024 | Что изменилось в стабильных трейтах и как мигрировать |
| [Embassy Book](https://embassy.dev/book/) | Embassy project, живой | Async на микроконтроллерах, исполнитель, HAL |
| [RTIC](https://rtic.rs/) | RTIC team, живой | Модель задач и ресурсов RTIC 2 |
| [probe-rs](https://probe.rs/) | probe-rs team, живой | Прошивка, отладка, интеграция с IDE |
| [defmt book](https://defmt.ferrous-systems.com/) | Ferrous Systems, живой | Отложенное форматирование логов |
| [Comprehensive Rust: Bare Metal](https://google.github.io/comprehensive-rust/bare-metal.html) | Google, живой | Сжатый курс bare-metal Rust |
| [The Rust on ESP Book](https://docs.esp-rs.org/book/) | esp-rs, живой | Экосистема ESP32 |
| [Awesome Embedded Rust](https://github.com/rust-embedded/awesome-embedded-rust) | Rust Embedded WG, живой | Каталог HAL, драйверов и инструментов |

---

## 5. WebAssembly и фронтенд

### Ключевые выводы
- Статус инструментов: организация `rustwasm` расформирована (архив с сентября 2025). `wasm-bindgen` передан в новую организацию [wasm-bindgen](https://github.com/wasm-bindgen) и активно поддерживается; `wasm-pack` также переехал туда и выпускает релизы (0.15, май 2026), но для приложений обычно удобнее Trunk или CLI фреймворка. `gloo` и `twiggy` переданы своим мейнтейнерам. Книга «Rust and WebAssembly» архивирована.
- Фреймворк: Leptos (0.8) — fine-grained реактивность, SSR, server functions, сборка `cargo-leptos`; Dioxus (0.7) — один код для web/desktop/mobile, hot-patching Rust-кода, CLI `dx`; Yew (0.23) — зрелый VDOM-фреймворк в стиле React. Для чистого CSR-приложения — Trunk.
- Граница JS↔Wasm дорогая: делайте крупные вызовы, передавайте массивы через типизированные массивы/линейную память, а не поэлементно; строки при пересечении перекодируются (UTF-8 ↔ UTF-16), держите их на одной стороне.
- Размер бинарника: `opt-level = "z"` или `"s"`, `lto = true`, `codegen-units = 1`, `panic = "abort"`, `strip = true`, затем `wasm-opt -Oz` (binaryen). Ищите раздутие `twiggy`; избегайте `format!`-тяжёлого кода и лишних serde-форматов. Разделение бандла по маршрутам есть в Dioxus 0.7.
- `wasm32-unknown-unknown` не имеет ОС: нет файловой системы, потоков по умолчанию, системного RNG — `getrandom` требует явного включения JS-бэкенда (feature `wasm_js`; детали зависят от версии крейта).
- Серверный Wasm: `wasm32-wasip2` — Tier 2 (с Rust 1.82), сразу собирает компонент Component Model; API WASI — крейт `wasip2`. `cargo-component` (экспериментальный) нужен только для собственных WIT-интерфейсов, для WASI-only хватает штатного таргета. WASI 0.3 с нативным async выпущен в июне 2026; `wasm32-wasip3` пока Tier 3 (std собирается из исходников). Хост для встраивания — `wasmtime`.

### Крейты
- `wasm-bindgen` (0.2), `wasm-bindgen-futures`, `js-sys` / `web-sys` (0.3) — привязки к JS и Web API.
- `wasm-pack` (0.15) — сборка npm-пакетов из Rust (живой после переезда).
- `trunk` (0.21; 0.22 в RC) — сборщик Wasm-приложений.
- `leptos` (0.8), `dioxus` (0.7; 0.8 в альфе), `yew` (0.23) — UI-фреймворки.
- `gloo` (0.12) — удобные обёртки над Web API.
- `wit-bindgen` (0.62), `wasip2` (2.x), `wasmtime` (49.x) — Component Model, WASI, рантайм.
- `cargo-component` (0.21, экспериментальный) — компоненты с пользовательскими WIT-мирами.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Sunsetting the rustwasm GitHub org](https://blog.rust-lang.org/inside-rust/2025/07/21/sunsetting-the-rustwasm-github-org/) | Inside Rust blog, 2025 | Что куда переехало и что архивировано |
| [wasm-bindgen Guide](https://wasm-bindgen.github.io/wasm-bindgen/) | wasm-bindgen org, живой | Актуальная документация по привязкам |
| [wasm-pack](https://github.com/wasm-bindgen/wasm-pack) | wasm-bindgen org, живой | Сборка и публикация npm-пакетов |
| [Rust and WebAssembly book](https://rustwasm.github.io/docs/book/) | Rust Wasm WG, архив 2025 | [устарело] — концепции верны, инструменты смотрите в wasm-bindgen Guide |
| [Leptos Book](https://book.leptos.dev/) | Leptos team, живой | Реактивность, SSR, server functions |
| [Dioxus 0.7 release](https://dioxuslabs.com/blog/release-070/) | DioxusLabs, 2025 | Hot-patching, Dioxus Native, fullstack на axum, code splitting |
| [Yew docs](https://yew.rs/) | Yew team, живой | VDOM-фреймворк в стиле React |
| [Trunk](https://github.com/trunk-rs/trunk) | trunk-rs, живой | Сборка и dev-сервер Wasm-приложений |
| [Component Model: Rust](https://component-model.bytecodealliance.org/language-support/creating-runnable-components/rust.html) | Bytecode Alliance, живой | Компоненты на Rust, WIT, `wasm32-wasip2` |
| [WASI 0.3](https://wasi.dev/releases/wasi-p3) | WASI.dev, 2026 | Нативный async в Component Model |
| [wasm32-wasip2](https://doc.rust-lang.org/rustc/platform-support/wasm32-wasip2.html) / [wasm32-wasip3](https://doc.rust-lang.org/rustc/platform-support/wasm32-wasip3.html) | rustc book, живой | Статус и требования таргетов |
| [min-sized-rust](https://github.com/johnthagen/min-sized-rust) | John Hagen, живой | Все приёмы уменьшения бинарника |
| [twiggy](https://github.com/AlexEne/twiggy) | twiggy maintainers, живой | Профилировщик размера Wasm |

---

## 6. Interop и FFI

### Ключевые выводы
- C → Rust: генерируйте привязки `bindgen` (в `build.rs` или заранее, с коммитом сгенерированного файла) в отдельный `*-sys`-крейт с ключом `links`; поверх него — безопасный крейт-обёртка с RAII и `Result`. Пользователи никогда не должны вызывать `unsafe` напрямую.
- Rust → C: `extern "C" fn` + `#[unsafe(no_mangle)]` (в edition 2024 атрибут требует `unsafe(...)`), типы `#[repr(C)]`, заголовок генерирует `cbindgen`. Сложные объекты отдавайте как непрозрачные указатели с парой функций `*_new`/`*_free`; документируйте, кто владеет памятью, и освобождайте её той же стороной, что выделила.
- Паники не должны пересекать границу: паника из `extern "C"` приводит к abort (с Rust 1.81). Если раскрутка через границу нужна (C++ исключения, `longjmp`), используйте `extern "C-unwind"` (стабилен с Rust 1.71); иначе ловите `catch_unwind` и возвращайте код ошибки. В edition 2024 блоки импорта пишутся `unsafe extern "C" { ... }`.
- C++: `cxx` — безопасный мост с общими типами (`String`, `Vec`, `unique_ptr`) в обе стороны, без ручного `unsafe`; `autocxx` автоматизирует генерацию поверх cxx, но развивается медленнее.
- Python: PyO3 + maturin. abi3-колёса (одно на все версии CPython) сокращают матрицу сборки, но недоступны для free-threaded Python. С PyO3 0.28 модули по умолчанию объявляются совместимыми с free-threading; в 0.26 `with_gil`/`allow_threads` переименованы в `Python::attach`/`detach` — отпускайте интерпретатор (`detach`) на долгих вычислениях.
- Node.js: napi-rs 3 генерирует TypeScript-типы, публикует бинарники по платформам отдельными npm-пакетами и умеет WASI-фолбэк. Мобильные платформы: UniFFI генерирует Kotlin/Swift/Python (Ruby — с ограничениями) из одного Rust-ядра; для прямого Android-доступа — крейт `jni`.
- Проектируйте границу крупнозернистой: несколько вызовов с пакетами данных лучше тысяч мелких; конвертируйте типы на границе и держите ядро без FFI-типов. FFI-слой гоняйте под Miri (где возможно) и санитайзерами.

### Крейты
- `bindgen` (0.73) — C/C++ заголовки → Rust.
- `cbindgen` (0.29) — Rust → C/C++ заголовки.
- `cxx` (1.0), `autocxx` (0.30) — двусторонний мост с C++.
- `pyo3` (0.29), `maturin` (1.x) — расширения Python и сборка колёс.
- `napi` / `napi-derive` (3.x) — нативные аддоны Node.js.
- `uniffi` (0.32) — привязки для Kotlin, Swift, Python.
- `jni` (0.22) — Java/Android через JNI.
- `diplomat` (0.16) — мультиязычные FFI-привязки по единому описанию.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The Rustonomicon: FFI](https://doc.rust-lang.org/nomicon/ffi.html) | Rust project, живой | Правила безопасности FFI, колбэки, владение |
| [RFC 2945: C-unwind ABI](https://rust-lang.github.io/rfcs/2945-c-unwind-abi.html) | Rust project, 2020; стабилизация в [Rust 1.71](https://blog.rust-lang.org/2023/07/13/Rust-1.71.0/) | Семантика раскрутки через FFI |
| [Edition 2024: unsafe extern](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-extern.html) / [unsafe attributes](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html) | Edition Guide, 2025 | Новый синтаксис FFI-объявлений |
| [bindgen User Guide](https://rust-lang.github.io/rust-bindgen/) | rust-lang, живой | Настройка генерации, allowlist, `-sys`-крейты |
| [cbindgen docs](https://github.com/mozilla/cbindgen/blob/main/docs.md) | Mozilla, живой | Конфигурация генерации заголовков |
| [CXX](https://cxx.rs/) | David Tolnay, живой | Безопасный мост Rust↔C++ |
| [PyO3 user guide](https://pyo3.rs/) | PyO3 team, живой | Модули, классы, GIL/free-threading ([раздел](https://pyo3.rs/latest/free-threading.html)) |
| [Maturin](https://www.maturin.rs/) | PyO3 team, живой | Сборка и публикация колёс |
| [NAPI-RS](https://napi.rs/) | napi-rs team, живой | Аддоны Node.js, кросс-платформенная публикация |
| [UniFFI user guide](https://mozilla.github.io/uniffi-rs/) | Mozilla, живой | Общее Rust-ядро для мобильных приложений |
| [Rust FFI Omnibus](https://jakegoulding.com/rust-ffi-omnibus/) | Jake Goulding, 2015+ | Минимальные примеры FFI для многих языков |

---

## 7. Данные, ML и численные методы

### Ключевые выводы
- Табличные данные: Polars (ленивый API `LazyFrame` с оптимизатором запросов) или DataFusion (встраиваемый SQL/DataFrame-движок на Arrow). Rust-API Polars живёт в линии 0.x и меняется между минорными версиями — закрепляйте версию; DataFusion и `arrow` выпускают мажоры часто, держите их версии согласованными (используйте реэкспорт `datafusion::arrow`).
- Колоночные данные быстрее построчных: работайте с Arrow-массивами и векторизованными выражениями, избегайте аллокации на строку и `Vec<struct>` для миллионов записей.
- Массивы и линейная алгебра: `ndarray` — n-мерные массивы в стиле NumPy; `nalgebra` — малые матрицы фиксированного размера, геометрия, трансформации; `faer` — высокопроизводительная плотная линейная алгебра на больших матрицах.
- CPU-параллелизм — `rayon` (`par_iter`): почти бесплатный переход от последовательного кода. Не вызывайте rayon прямо из async-задачи (блокирует воркер tokio) — оборачивайте в `spawn_blocking` или передавайте результат через канал; избегайте вложенных пулов с переподпиской ядер.
- ML: обучать и запускать модели целиком на Rust — `burn` (бэкенды CUDA, ROCm, Metal, Vulkan, WebGPU, LibTorch, CPU; импорт ONNX и safetensors); лёгкий инференс и готовые порты LLM/vision-моделей — `candle` (CPU/CUDA/Metal/Wasm); модели, обученные в Python, в продакшене — через ONNX Runtime (`ort`, линия 2.0 пока в RC).
- Частый путь — Rust как ускоритель для Python: ядро на Rust + PyO3/maturin (см. раздел 6); так устроены Polars и многие пакеты ML-экосистемы.

### Крейты
- `polars` (0.55) — DataFrame с ленивыми запросами.
- `arrow` (60.x), `datafusion` (55.x) — колоночный формат и SQL-движок Apache.
- `ndarray` (0.17), `nalgebra` (0.35), `faer` (0.24) — массивы и линейная алгебра.
- `rayon` (1.x) — data-parallel итераторы.
- `burn` (0.21) — обучение и инференс, мультибэкенд.
- `candle-core` (0.11) — минималистичный инференс от Hugging Face.
- `ort` (2.0-rc) — ONNX Runtime; `tch` (0.26) — привязки к LibTorch.
- `linfa` (0.8) — классическое ML (кластеризация, регрессии).
- `cudarc` (0.19) — низкоуровневый доступ к CUDA.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Polars user guide](https://docs.pola.rs/) | Polars team, живой | Ленивые запросы, выражения, оптимизации |
| [Polars Rust API](https://docs.rs/polars/latest/polars/) | Polars team, живой | Rust-специфика и feature-флаги |
| [Apache DataFusion](https://datafusion.apache.org/) | Apache, живой | Встраиваемый SQL-движок, расширения |
| [Apache Arrow Rust](https://arrow.apache.org/rust/) | Apache, живой | Колоночные массивы, Parquet |
| [ndarray docs](https://docs.rs/ndarray/latest/ndarray/) | rust-ndarray, живой | Включает раздел для пользователей NumPy |
| [nalgebra](https://nalgebra.rs/) | Dimforge, живой | Линейная алгебра и геометрия |
| [The Burn Book](https://burn.dev/books/burn/) | Tracel AI, живой | Обучение моделей на Rust, бэкенды |
| [Candle](https://github.com/huggingface/candle) | Hugging Face, живой | Примеры инференса LLM и vision-моделей; [книга](https://huggingface.github.io/candle/) |
| [ort](https://ort.pyke.io/) | pyke, живой | ONNX Runtime из Rust |
| [Are We Learning Yet?](https://www.arewelearningyet.com/) | сообщество, живой | Каталог ML-крейтов |

---

## 8. Desktop и GUI

### Ключевые выводы
- Выбор по задаче. Tauri 2 — веб-UI в системном webview + Rust-бэкенд, маленькие бинарники, десктоп и iOS/Android. egui — immediate mode, лучший выбор для инструментов, отладочных панелей и оверлеев в играх. iced — Elm-архитектура (state → message → update → view) для полноценных приложений. Slint — декларативный DSL `.slint`, от микроконтроллеров до десктопа и мобильных.
- Tauri: каждая команда и плагин доступны фронтенду только через явно выданные capabilities/permissions — выдавайте минимум. Webview различаются по ОС (WebView2, WKWebView, WebKitGTK) — тестируйте на каждой платформе. Тяжёлую работу выполняйте в async-командах, не в UI-потоке.
- egui перерисовывает UI каждый кадр из вашего состояния: держите состояние в своих структурах, не создавайте тяжёлые объекты в замыканиях отрисовки; нативный вид и сложные раскладки — не его сильная сторона. `eframe` запускает одно приложение нативно и в браузере.
- iced и gpui — pre-1.0 с регулярными breaking changes; закрепляйте версию. gpui (фреймворк редактора Zed) развивается внутри репозитория Zed, версия на crates.io (0.2.2, октябрь 2025) отстаёт от репозитория — берите его, только если готовы к частым изменениям.
- Slint распространяется под GPL, бесплатной royalty-free лицензией для desktop/mobile/web и коммерческой — проверьте условия до выбора. Для встраиваемых устройств лицензирование отдельное.
- Независимо от фреймворка выносите логику приложения в UI-независимый крейт: так её можно тестировать без окна и при необходимости сменить UI.

### Крейты
- `tauri` (2.x; 3.0 в альфе) — webview-приложения с Rust-бэкендом.
- `egui` / `eframe` (0.36) — immediate-mode GUI.
- `iced` (0.14) — Elm-архитектура, кроссплатформенный рендер.
- `slint` (1.x) — декларативный UI с отдельным DSL.
- `gpui` (0.2, pre-1.0) — GPU-ускоренный UI от Zed.
- `winit` (0.30; 0.31 в бете), `wgpu` (30.x) — окна и графика для собственных решений.
- `xilem` (0.4), `vello` (0.11) — экспериментальный реактивный UI и GPU-рендерер 2D от Linebender.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Tauri 2](https://v2.tauri.app/) | Tauri team, живой | Архитектура, capabilities, мобильные таргеты |
| [egui](https://github.com/emilk/egui) / [демо](https://www.egui.rs/) | Emil Ernerfeldt, живой | Immediate-mode GUI, интерактивное демо всех виджетов |
| [iced book](https://book.iced.rs/) | iced team, живой | Elm-архитектура на Rust |
| [Slint](https://slint.dev/) | SixtyFPS GmbH, живой | DSL, платформы, лицензии |
| [GPUI](https://www.gpui.rs/) / [README в Zed](https://github.com/zed-industries/zed/tree/main/crates/gpui) | Zed Industries, живой | Статус pre-1.0, платформенные требования |
| [Are we GUI yet?](https://areweguiyet.com/) | сообщество, живой | Обзор GUI-экосистемы |

---

## 9. Разработка игр

### Ключевые выводы
- Движок по задаче. Bevy (0.19, июнь 2026; новый мажор примерно каждые 3–4 месяца с миграционными гайдами) — ECS-first, всё в коде, сцены BSN. Godot через gdext (крейт `godot` 0.5) — редактор, сцены, анимация и UI в Godot, логика на Rust. Fyrox (1.0, март 2026) — Unity-подобный движок с собственным редактором. macroquad — простые 2D-игры, прототипы и джемы.
- Отделяйте ядро правил/симуляции от презентации: чистый крейт без зависимости от движка, API вида `Action → validate(&State) → apply → Vec<Event>`; движок только отправляет действия и проигрывает события (анимации, звук). Это даёт headless-тесты, фаззинг, серверную авторитетность, ботов и реплеи.
- Детерминизм: вся случайность — через сидированный RNG, хранимый в состоянии (`rand_chacha`); не зависьте от порядка итерации `HashMap` (он рандомизирован) — используйте `BTreeMap`/`IndexMap` или сортировку; фиксированный шаг симуляции. Реплей = сид + начальное состояние + журнал действий; при serde-сериализуемых `State`/`Action`/`Event` бесплатно получаются сохранения, синхронизация по сети и golden-тесты.
- Идентичность объектов — индексы и generational-ключи (`slotmap`), а не `Rc<RefCell<_>>`: ключ удалённого объекта больше не валиден (нет ABA), и исчезают runtime-паники заимствований. Ссылки на объекты движка (например, `Gd<T>` в gdext) держите только в слое-мосте, отображая `EntityId → узел`.
- ECS оправдан для большого числа однородных сущностей в реальном времени. Для пошаговых игр и небольших состояний типизированные `Vec`/slotmap + `enum` проще, быстрее компилируются и сериализуются.
- Контент — данными (RON/JSON/TOML + дерево `enum`-эффектов), а не кодом с замыканиями: баланс меняется без перекомпиляции, данные сериализуются и сравниваются.
- Граница с движком (gdext): один `cdylib`-крейт-мост, `reloadable = true` в `.gdextension` для hot reload; крупные редкие вызовы с пакетами событий вместо множества мелких; типы движка (`Gd<T>`, `Variant`) не проникают в ядро. Доступ к освобождённому `Gd`-объекту паникует. Закрепляйте точную версию `godot` и читайте миграционный гайд перед обновлением — API быстро меняется.
- Критика и смягчение. LogLog Games (2024): Rust тормозит итерацию геймплея, слабый hot reload, нет рефлексии, незрелый GUI, паники `RefCell`, навязанный ECS. Barrett (2024): «Rust — для движка, не для игры». Практический ответ: Rust для систем, симуляции и правил, где корректность окупается; быстро меняющийся геймплей и UI — в редакторе/скриптах/данных. Для скорости сборки в dev-профиле оптимизируйте зависимости (`[profile.dev.package."*"] opt-level = 2`) и используйте feature `bevy/dynamic_linking`.

### Крейты
- `bevy` (0.19; 0.20 в RC), `bevy_ecs` — движок и его ECS отдельно.
- `godot` (0.5) — gdext, привязки к Godot 4.
- `fyrox` (1.x) — движок с редактором.
- `macroquad` (0.4) — простой 2D/3D-фреймворк.
- `hecs` (0.11) — минималистичный ECS без планировщика; `flecs_ecs` (0.2) — привязки к Flecs с отношениями.
- `slotmap` (1.x) — generational-ключи: `SlotMap`, `HopSlotMap`, `DenseSlotMap`, `SecondaryMap`.
- `rand` (0.10), `rand_chacha` (0.10) — детерминированный сидированный RNG.
- `ron` (0.12), `serde` — данные контента и сохранения.
- `rapier3d` (0.36), `avian3d` (0.7) — физика (avian — ECS-нативная для Bevy).
- `wgpu` (30.x), `winit` (0.30) — графика и окна для собственного движка.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Bevy](https://bevy.org/) / [Bevy 0.19](https://bevy.org/news/bevy-0-19/) | Bevy Foundation, 2026 | Quick Start, примеры, release notes и миграции |
| [Tainted Coders: Bevy](https://taintedcoders.com/) | Tainted Coders, обновлено 2026 | Самые актуальные подробные гайды по архитектуре Bevy |
| [Bevy Cheatbook](https://bevy-cheatbook.github.io/) | Ida Iyes, до ~2024 | [устарело] — официально не поддерживается; концепции полезны, API сверяйте с Tainted Coders |
| [This Week in Bevy](https://thisweekinbevy.com/) | Chris Biscardi, еженедельно | Новости экосистемы Bevy |
| [godot-rust book](https://godot-rust.github.io/book/) | godot-rust team, живой | Установка, `Gd<T>`, регистрация классов, миграции до v0.5 |
| [gdext: Hello World](https://godot-rust.github.io/book/intro/hello-world.html) | godot-rust team, живой | Раскладка `godot/` + `rust/`, `cdylib`, hot reload |
| [gdnative book: Game architecture](https://godot-rust.github.io/gdnative-book/overview/architecture.html) | godot-rust team, ~2021 | Три архитектуры Rust+Godot; вариант «игра на Rust + Godot как I/O» — самый тестируемый. Книга для Godot 3, но разбор компромиссов актуален |
| [Fyrox](https://fyrox.rs/) / [Fyrox Book](https://fyrox-book.github.io/) | Fyrox team, 2026 | Движок 1.0 с редактором |
| [macroquad](https://macroquad.rs/) | Fedor Logachev, живой | Минимальный порог входа в 2D |
| [Using Rust for Game Development](https://kyren.github.io/2018/09/14/rustconf-talk.html) | Catherine West, 2018 | Откуда в Rust-геймдеве generational indices и ECS вместо графов объектов |
| [slotmap](https://docs.rs/slotmap/latest/slotmap/) | Orson Peters, живой | Выбор между вариантами SlotMap по доступу/итерации |
| [Leaving Rust gamedev after 3 years](https://loglog.games/blog/leaving-rust-gamedev/) | LogLog Games, 2024 | Главная критика Rust для геймплейного кода |
| [Rust is for the Engine, Not the Game](https://barretts.club/posts/rust-for-the-engine/) | Barrett, 2024 | Аргумент за разделение: Rust для движка, динамический слой для геймплея |
| [Game Programming Patterns](https://gameprogrammingpatterns.com/) | Robert Nystrom, 2014 | Command, Event Queue, State, Type Object — словарь для действий, событий и данных |
| [Data-Oriented Design](https://www.dataorienteddesign.com/dodbook/) | Richard Fabian, 2018 | Основы DOD: таблицы, обработка по наличию |
| [Are We Game Yet?](https://arewegameyet.rs/) | Rust GameDev WG, живой | Каталог движков и крейтов |

---

## 10. Системное программирование, сеть, ОС

### Ключевые выводы
- tokio — рантайм по умолчанию. Многопоточный планировщик с work-stealing требует от задач `Send + 'static`; не блокируйте воркеры (`spawn_blocking` для синхронного I/O и CPU), не держите синхронный мьютекс через `.await`, используйте ограниченные каналы для backpressure и акторы вместо разделяемого `Arc<Mutex<_>>` в сложных случаях.
- TLS — `rustls` 0.23 с подключаемыми криптопровайдерами (по умолчанию `aws-lc-rs`, альтернатива — `ring`); интеграция с tokio — `tokio-rustls`. QUIC/HTTP/3 — `quinn` (чистый Rust поверх rustls) или `s2n-quic` (AWS).
- io_uring только по данным профилирования: для большинства сетевых сервисов tokio на epoll достаточно. Безопасный API io_uring требует, чтобы буферы принадлежали ядру/рантайму (отменённый future не может освобождать буфер, в который ещё пишет ядро) — отсюда отличающиеся от `AsyncRead` API. Docker (с 25.0) по умолчанию блокирует io_uring seccomp-профилем, многие песочницы тоже — предусмотрите фолбэк.
- Выбор io_uring-рантайма: `tokio-uring` фактически застыл (последний релиз 0.5 в мае 2024); `monoio` и `glommio` — thread-per-core, задачи без `Send`, релизы редкие; `compio` — активно развивающийся thread-per-core рантайм с бэкендами io_uring/IOCP/polling, то есть кроссплатформенный.
- Низкоуровневые системные вызовы: `rustix` (безопасные обёртки, может работать без libc на Linux) или `nix` (обёртки над libc); тонкая настройка сокетов — `socket2`; событийный цикл без рантайма — `mio`.
- Rust в ядре Linux больше не эксперимент (решение Maintainers Summit, декабрь 2025); в mainline есть драйверы Nova (GPU), Android Binder, Tyr. Код ядра — без std, со своими API аллокации (ошибка выделения возвращается, а не паникует) и pinned-инициализацией; начинайте с документации ядра и rust-for-linux.com.

### Крейты
- `tokio` (1.x), `tokio-util` (0.7), `bytes` (1.x), `mio` (1.x) — асинхронный I/O.
- `rustls` (0.23), `tokio-rustls` (0.26), `aws-lc-rs` (1.x) — TLS и криптография.
- `quinn` (0.11), `s2n-quic` (1.x) — QUIC.
- `hickory-resolver` (0.26) — DNS-резолвер на Rust.
- `io-uring` (0.7) — низкоуровневые привязки к io_uring.
- `compio` (0.19), `monoio` (0.2), `glommio` (0.9), `tokio-uring` (0.5, застыл) — completion-based рантаймы.
- `rustix` (1.x), `nix` (0.31), `socket2` (0.6) — системные вызовы и сокеты.

### Ресурсы
| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Tokio tutorial](https://tokio.rs/tokio/tutorial) | Tokio team, живой | Задачи, каналы, разделяемое состояние, I/O |
| [Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/) | Alice Ryhl, 2021 | Структурирование конкурентного кода без мьютексов |
| [rustls manual](https://docs.rs/rustls/latest/rustls/manual/index.html) | rustls team, живой | Дизайн, криптопровайдеры, FAQ |
| [Quinn book](https://quinn-rs.github.io/quinn/) | quinn team, живой | QUIC на Rust: соединения, потоки, сертификаты |
| [Notes on io-uring](https://without.boats/blog/io-uring/) | withoutboats, 2020 | Почему безопасный io_uring требует владения буферами |
| [compio](https://github.com/compio-rs/compio) | compio team, живой | Кроссплатформенный completion-based рантайм |
| [monoio](https://github.com/monoio-rs/monoio) / [glommio](https://github.com/DataDog/glommio) | ByteDance / Datadog, живые | Thread-per-core модели и их ограничения |
| [Rust for Linux](https://rust-for-linux.com/) | Rust for Linux, живой | Статус, драйверы, ссылки на документацию |
| [Rust in the kernel docs](https://docs.kernel.org/rust/index.html) | Linux kernel, живой | Официальная сборка и правила кода |
| [The (successful) end of the kernel Rust experiment](https://lwn.net/Articles/1049831/) | LWN, 2025 | Снятие статуса эксперимента |
