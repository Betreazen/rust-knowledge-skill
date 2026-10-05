# Инструментарий Rust: rustup, cargo и cargo-подкоманды

Справочник по инструментам вокруг компилятора по состоянию на октябрь 2026 года. Актуальный stable — 1.99 (1 октября 2026), edition 2024, следующий релиз 1.100 ожидается 12 ноября 2026. Для каждого инструмента указано, что он делает, как его поставить, какие команды нужны чаще всего, где он стоит в рабочем цикле и какие у него ограничения по платформам и nightly. В конце собраны минимальный набор, порядок шагов CI, шаблоны `[workspace.lints]` и профилей, а также ссылки на документацию. В скобках у инструментов стоит последняя версия на crates.io на октябрь 2026.

- [0. Как ставить инструменты](#0-как-ставить-инструменты)
- [1. База: rustup, cargo, rustfmt, clippy, rust-analyzer](#1-база-rustup-cargo-rustfmt-clippy-rust-analyzer)
- [2. Быстрый цикл обратной связи](#2-быстрый-цикл-обратной-связи)
- [3. Тестирование и корректность](#3-тестирование-и-корректность)
- [4. Покрытие](#4-покрытие)
- [5. Зависимости и supply chain](#5-зависимости-и-supply-chain)
- [6. Бенчмарки и профилирование](#6-бенчмарки-и-профилирование)
- [7. Скорость сборки и релиз](#7-скорость-сборки-и-релиз)
- [8. Понимание кода](#8-понимание-кода)
- [9. Документация](#9-документация)
- [10. Минимальный набор и инструменты по ситуации](#10-минимальный-набор-и-инструменты-по-ситуации)
- [11. Порядок CI-пайплайна](#11-порядок-ci-пайплайна)
- [12. Шаблоны: `[workspace.lints]` и профили](#12-шаблоны-workspacelints-и-профили)
- [13. Ресурсы](#13-ресурсы)

## 0. Как ставить инструменты

**Ключевые выводы**
- Для установки из исходников используйте `cargo install --locked <crate>`: флаг `--locked` берёт проверенный авторами `Cargo.lock`, и без него сборка может сломаться на свежей версии зависимости.
- `cargo binstall <crate>` скачивает готовый бинарник, что в разы быстрее компиляции. Если готового бинарника нет, он откатывается на `cargo install`.
- В GitHub Actions ставьте инструменты через `taiki-e/install-action` (`with: tool: cargo-nextest,cargo-llvm-cov`). Action проверяет SHA256 и выдерживает паузу в несколько дней перед установкой свежих версий, если версия не закреплена.
- Тулчейн в CI ставьте явно через `rustup toolchain install` или `dtolnay/rust-toolchain`. Начиная с rustup 1.30 команды `rustup` больше не ставят тулчейн неявно. Прокси `cargo` и `rustc` эту возможность сохраняют, поэтому не полагайтесь на `rustup show` как на установщик.
- Для кеширования `target/` и `~/.cargo` в CI используйте `Swatinem/rust-cache`.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [cargo-binstall](https://github.com/cargo-bins/cargo-binstall) | cargo-bins, 2026 (1.25) | Установка готовых бинарников cargo-подкоманд |
| [taiki-e/install-action](https://github.com/taiki-e/install-action) | Taiki Endo, 2026 | Установка инструментов в GitHub Actions с проверкой контрольных сумм |
| [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache) | Arpad Borsos, 2026 | Кеш сборки в GitHub Actions |
| [Rustup 1.30: implicit installation](https://blog.rust-lang.org/inside-rust/2026/07/03/rustup-update-1.30/) | Rustup team, 2026 | Какие вызовы перестают сами ставить тулчейн |

## 1. База: rustup, cargo, rustfmt, clippy, rust-analyzer

**Ключевые выводы**
- Закрепите тулчейн в `rust-toolchain.toml` в корне репозитория: канал, компоненты и таргеты. Тогда локальная сборка, CI и агент работают на одной версии. Nightly указывайте с датой (`nightly-2026-10-01`), иначе сборки не воспроизводятся.
- Указывайте `rust-version` в `Cargo.toml`. Резолвер v3 (по умолчанию в edition 2024) учитывает MSRV при выборе версий зависимостей, а Clippy использует это значение как `msrv` и не предлагает API новее него.
- Задавайте линты в `[workspace.lints]` (Cargo 1.74+), а в крейтах пишите `[lints] workspace = true`. Так проще, чем дублировать `#![deny(...)]` по крейтам. Группы ставьте с `priority = -1`, чтобы точечные настройки их перекрывали.
- `clippy::pedantic` включайте целиком и гасите шумные линты. `clippy::restriction` целиком не включайте: эти линты противоречат друг другу, их выбирают поштучно.
- Форматирование проверяйте через `cargo fmt --check`. rustfmt берёт edition из `Cargo.toml`, поэтому `rustfmt.toml` нужен только для отклонений от стиля по умолчанию.
- rust-analyzer ставьте через rustup или берите встроенный в расширение редактора. Опция `rust-analyzer.cargo.targetDir = true` даёт ему отдельный каталог сборки, и он перестаёт блокировать `cargo build` в терминале.
- `cargo update --breaking` так и не стабилизирован и удаляется в 1.100. Для мажорных обновлений зависимостей используйте `cargo upgrade --incompatible` из cargo-edit.

| Инструмент | Что и зачем | Установка | Главные команды | Оговорки |
|---|---|---|---|---|
| rustup (1.29) | Ставит тулчейны, компоненты и таргеты, переключает версии | `rustup-init` с rustup.rs | `rustup update`; `rustup target add wasm32-unknown-unknown`; `rustup component add rust-src llvm-tools-preview`; `cargo +nightly …` | Приоритет выбора: `+toolchain` → `RUSTUP_TOOLCHAIN` → `rustup override` → `rust-toolchain.toml` → default. В 1.29 загрузки идут параллельно |
| cargo (встроенные) | Сборка, зависимости, публикация | вместе с Rust | `cargo check --all-targets`; `cargo build --timings`; `cargo tree -d` (дубликаты), `-i <crate>` (кто тянет), `-e features`; `cargo info <crate>`; `cargo add/remove`; `cargo update -p <crate> --precise <ver>`; `cargo fix --edition`; `cargo publish --dry-run` | `cargo tree` закрывает большую часть задач старых `cargo-tree`/`cargo-deps` |
| rustfmt | Единый стиль кода | `rustup component add rustfmt` | `cargo fmt`; `cargo fmt --check` (CI) | Стабильны `edition`, `style_edition` (до `"2024"`), `max_width`, `newline_style`, `reorder_imports`, `use_small_heuristics`. `imports_granularity` и `group_imports` нестабильны и работают только с nightly rustfmt |
| clippy (~850 линтов) | Линтер: ошибки, идиомы, производительность | `rustup component add clippy` | `cargo clippy --all-targets --all-features -- -D warnings`; `cargo clippy --fix --allow-dirty` | Группы по умолчанию: `correctness` = deny; `suspicious`, `style`, `complexity`, `perf` = warn; `pedantic`, `restriction`, `nursery`, `cargo` = allow |
| rust-analyzer | LSP: автодополнение, переходы, инлайн-ошибки, рефакторинги | `rustup component add rust-analyzer` или расширение редактора | `rust-analyzer.check.command = "clippy"` даёт Clippy при сохранении | Релизы выходят еженедельно. В больших воркспейсах ограничивайте `cargo.features` и включайте `cargo.targetDir` |

```toml
# rust-toolchain.toml
[toolchain]
channel = "1.99"                      # или "stable"; nightly — только с датой
components = ["rustfmt", "clippy", "rust-analyzer"]
targets = ["wasm32-unknown-unknown"]  # только нужные
profile = "minimal"
```

**Полезные линты вне групп по умолчанию.** Группы проверены по индексу Clippy stable, октябрь 2026.
- Код с `unsafe` (restriction): `undocumented_unsafe_blocks` требует комментарий `// SAFETY:` перед блоком; `multiple_unsafe_ops_per_block` требует одну операцию на блок; `unnecessary_safety_comment`. Линт `missing_safety_doc` (раздел `# Safety` у `unsafe fn`) уже входит в `style` и включён по умолчанию.
- Гигиена (restriction): `allow_attributes_without_reason` требует `#[allow(..., reason = "...")]`; `dbg_macro`; `todo`; `unwrap_used`/`expect_used` для сервисов и библиотек; `print_stdout` для библиотек; `indexing_slicing` там, где паника недопустима; `clone_on_ref_ptr` (`Arc::clone(&x)` вместо `x.clone()`). `module_name_repetitions` теперь тоже в restriction.
- Шум в pedantic, который обычно выключают: `missing_errors_doc`, `missing_panics_doc`, `must_use_candidate`. В nursery есть полезный `redundant_clone`.
- В `clippy.toml` можно ослабить правила для тестов: `allow-unwrap-in-tests`, `allow-expect-in-tests`, `allow-dbg-in-tests`, `allow-print-in-tests`, `allow-panic-in-tests`, `allow-indexing-slicing-in-tests`.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The rustup book: overrides](https://rust-lang.github.io/rustup/overrides.html) | Rust project, 2026 | Формат `rust-toolchain.toml` и порядок приоритетов |
| [Cargo: `[lints]`](https://doc.rust-lang.org/cargo/reference/manifest.html) и [`[workspace.lints]`](https://doc.rust-lang.org/cargo/reference/workspaces.html) | Rust project, 2026 | Настройка линтов, `priority`, наследование |
| [Clippy lint list (stable)](https://rust-lang.github.io/rust-clippy/stable/index.html) | Clippy team, 2026 | Фильтр по группам, описание каждого линта |
| [Clippy lint configuration](https://doc.rust-lang.org/clippy/configuration.html) | Clippy team, 2026 | Ключи `clippy.toml`, `msrv` |
| [rustfmt configuration](https://rust-lang.github.io/rustfmt/) | rustfmt team, 2026 | Какие опции стабильны |
| [rust-analyzer](https://rust-analyzer.github.io/) | rust-analyzer team, 2026 | Установка и конфигурация |
| [Cargo: dependency resolution](https://doc.rust-lang.org/cargo/reference/resolver.html) | Rust project, 2026 | Резолвер v3 и MSRV-aware выбор версий |

## 2. Быстрый цикл обратной связи

**Ключевые выводы**
- Для фоновых проверок используйте bacon вместо cargo-watch. cargo-watch [устарело]: автор перевёл его в режим «life support», а репозиторий заархивирован в январе 2025. Для произвольных команд по изменению файлов подходит `watchexec`.
- В цикле правки запускайте `cargo check`, а не `cargo build`: check пропускает кодогенерацию и линковку. `--all-targets` добавляет тесты, бенчмарки и примеры.
- Если тесты медленные, переходите на `cargo nextest run`: процесс на тест и параллелизм по бинарникам обычно ускоряют прогон.
- В dev-профиле `debug = "line-tables-only"` (или `0`) ускоряет сборку и линковку, а номера строк в бэктрейсах остаются.
- Если debug-сборка слишком медленная в работе (игры, численные задачи), оптимизируйте только зависимости: `[profile.dev.package."*"] opt-level = 2`. Они пересобираются редко, а ваш код остаётся быстро компилируемым.
- Узкие места сборки ищите через `cargo build --timings`: отчёт в HTML показывает граф крейтов и время каждого.

| Инструмент | Что и зачем | Установка | Главные команды | Оговорки |
|---|---|---|---|---|
| bacon (3.26) | TUI в фоне: при сохранении запускает check/clippy/test и показывает первые ошибки | `cargo install --locked bacon` | `bacon`; `bacon clippy`; `bacon test`; `bacon --init` (создаёт `bacon.toml` со своими job'ами) | Linux, macOS, Windows. Лицензия AGPL-3.0 касается только самого инструмента |
| cargo-watch (8.5.3) | [устарело] Перезапуск команд по изменению файлов | — | — | Заменён на bacon или [watchexec](https://github.com/watchexec/watchexec) (`watchexec -e rs -- cargo test`) |
| cargo check | Проверка типов без кодогенерации | встроено | `cargo check --workspace --all-targets` | Ошибки уровня линковки и мономорфизации (часть `const`-вычислений) проявятся только при build |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [bacon](https://github.com/Canop/bacon) | Denys Séguret, 2026 | Job'ы, `bacon.toml`, горячие клавиши |
| [cargo-watch README](https://github.com/watchexec/cargo-watch) | Félix Saparelli, 2025 | Статус «life support» и рекомендованные замены |
| [Tips for Faster Rust Compile Times](https://corrode.dev/blog/tips-for-faster-rust-compile-times/) | Matthias Endler (corrode), обновляется | Практики ускорения цикла правка → проверка |

## 3. Тестирование и корректность

**Ключевые выводы**
- Базовый раннер — `cargo nextest run`. Doctest'ы nextest не запускает, поэтому в CI нужен отдельный шаг `cargo test --doc`.
- Снапшоты (insta) подходят для больших структурированных выводов: AST, JSON, сообщений об ошибках, CLI-вывода. Property-тесты (proptest) подходят для инвариантов вроде «parse(print(x)) == x». Добавляйте оба там, где ручные `assert_eq!` разрастаются.
- Каждый крейт с `unsafe` прогоняйте под Miri (`cargo +nightly miri test`). Это самый дешёвый способ поймать UB, хотя покрывает он только исполненные пути.
- Lock-free код и самописные примитивы синхронизации проверяйте Loom'ом. Он перебирает чередования потоков, а обычные тесты видят одно случайное.
- Fuzzing (cargo-fuzz или afl.rs) нужен для парсеров, декодеров и всего, что принимает недоверенный ввод. Найденные падения сохраняйте как регрессионные тесты.
- cargo-mutants проверяет, что тесты действительно ловят изменения в логике. В CI запускайте его на диффе PR (`--in-diff`), а не на всём коде.
- Kani доказывает отсутствие паник, переполнений и UB для всех входов в ограниченной модели. Применяйте его точечно, к критичным функциям с `unsafe` и арифметикой.

| Инструмент | Что и зачем | Установка | Главные команды | Когда / оговорки |
|---|---|---|---|---|
| cargo-nextest (0.9.146) | Быстрый раннер: процесс на тест, ретраи flaky-тестов, шардирование, JUnit | `cargo install --locked cargo-nextest`, `cargo binstall cargo-nextest`, готовые бинарники | `cargo nextest run`; `--workspace --all-features`; `--profile ci` (ретраи и JUnit в `.config/nextest.toml`); `--partition count:1/4`; `cargo nextest list` | Linux, macOS, Windows. Doctest'ы не поддерживаются, запускайте `cargo test --doc` |
| insta + cargo-insta (1.49) | Снапшот-тесты с интерактивным ревью | `cargo add --dev insta` (+ features `yaml`, `json`, `redactions`); `cargo install --locked cargo-insta` | `assert_snapshot!`, `assert_debug_snapshot!`, `assert_yaml_snapshot!`, inline `@"..."`; `cargo insta test --review`; `cargo insta accept` | При `CI=true` новые снапшоты не записываются и тест падает. Снапшоты коммитьте |
| proptest (1.11) | Property-based тесты со сжатием контрпримера | `cargo add --dev proptest` | `proptest! { #[test] fn t(x in 0..1000u32) { … } }`; стратегии `prop_oneof!`, `any::<T>()` | Коммитьте каталог `proptest-regressions/`: в нём лежат найденные контрпримеры |
| cargo-fuzz (0.13) | Coverage-guided fuzzing на libFuzzer | `cargo install cargo-fuzz` | `cargo fuzz init`; `cargo fuzz add <target>`; `cargo +nightly fuzz run <target>`; `cargo fuzz cmin`; `cargo fuzz coverage` | Нужен nightly (sanitizer). x86-64 Linux/macOS, Apple Silicon, Windows через MSVC ASan. Структурированный ввод — через крейт `arbitrary` |
| afl.rs (`cargo-afl`, 0.18) | Fuzzing на AFL++ | `cargo install cargo-afl` | `cargo afl build`; `cargo afl fuzz -i in -o out target/debug/<bin>` | x86-64 Linux, x86-64 и ARM64 macOS, без Windows. Нужен C-компилятор и make |
| cargo-mutants (27.1) | Мутационное тестирование: находит изменения кода, которые не ловит ни один тест | `cargo install --locked cargo-mutants` | `cargo mutants`; `cargo mutants --in-diff git.diff`; `--test-tool=nextest`; `-f src/x.rs`; `--jobs 4` | Полный прогон стоит сотен сборок. Результаты в `mutants.out/` (`missed.txt`) |
| Miri | Интерпретатор MIR, ловит UB: выход за границы, use-after-free, неинициализированную память, нарушения aliasing (Stacked/Tree Borrows), гонки данных, утечки | `rustup +nightly component add miri` | `cargo +nightly miri test`; `cargo +nightly miri run`; `MIRIFLAGS="-Zmiri-many-seeds"`; `-Zmiri-tree-borrows`; `-Zmiri-disable-isolation` | Только nightly. Медленнее нативного запуска на порядки, поэтому тяжёлые тесты помечайте `#[cfg_attr(miri, ignore)]`. FFI и сеть почти не поддерживаются, лучше всего поддержан Linux |
| Loom (0.7.2) | Перебор чередований потоков по модели памяти C11 | `[target.'cfg(loom)'.dependencies] loom = "0.7"` | `RUSTFLAGS="--cfg loom" cargo test --release`; `loom::model(\|\| …)`; `LOOM_MAX_PREEMPTIONS=2` | В коде подменяйте `std::sync` на `loom::sync` под `cfg(loom)`. `SeqCst` моделируется как `AcqRel`. Годится только для маленьких сценариев. Релизы редкие, API стабилен. Для крупных сценариев есть рандомизированный [shuttle](https://github.com/awslabs/shuttle) |
| Kani (0.68) | Bounded model checking: доказывает отсутствие паник, переполнений и UB для всех входов | `cargo install --locked kani-verifier && cargo kani setup` | `#[kani::proof]` + `kani::any()`; `#[kani::unwind(n)]`; `cargo kani` | Только Linux и macOS, на Windows — через WSL. Время растёт с размером циклов |
| Санитайзеры (ASan, TSan, MSan, LSan) | Инструментирование на уровне LLVM: память, гонки, неинициализированные чтения | nightly + `rustup component add rust-src` | `RUSTFLAGS=-Zsanitizer=address cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu` | Все санитайзеры пока нестабильны. Стабилизация ASan/LSan (флаг `-Csanitize`) и цель 2026 года по MSan/TSan ещё не завершены. ASan работает на Linux, macOS и FreeBSD, а в списке целей unstable book нет Windows MSVC. Нужны для FFI и C-зависимостей, где Miri бессилен |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [nextest](https://nexte.st/) | Rain и nextest-rs, 2026 | Профили, ретраи, шардирование, CI-рецепты |
| [insta quickstart](https://insta.rs/docs/quickstart/) | Armin Ronacher, 2026 | Макросы, ревью, поведение в CI |
| [proptest](https://github.com/proptest-rs/proptest) | proptest-rs, 2026 | Стратегии и сжатие |
| [Rust Fuzz Book: cargo-fuzz](https://rust-fuzz.github.io/book/cargo-fuzz/setup.html), [afl.rs](https://rust-fuzz.github.io/book/afl/setup.html) | rust-fuzz, 2026 | Настройка fuzzing, платформы |
| [cargo-mutants](https://mutants.rs/) | Martin Pool, 2026 | Мутационное тестирование, `--in-diff`, CI |
| [Miri](https://github.com/rust-lang/miri) | Rust project (Ralf Jung и др.), 2026 | Флаги, ограничения, что ловится |
| [Loom](https://github.com/tokio-rs/loom) | Tokio, 2024 | Модель, ограничения, переменные окружения |
| [Kani](https://github.com/model-checking/kani) и [Kani book](https://model-checking.github.io/kani/) | AWS, 2026 | Proof harness'ы, контракты |
| [Sanitizers (Unstable Book)](https://doc.rust-lang.org/beta/unstable-book/compiler-flags/sanitizer.html) | Rust project, 2026 | Флаги и поддерживаемые таргеты |
| [Project goal: sanitizer stabilization](https://rust-lang.github.io/rust-project-goals/2026/stabilization-of-sanitizer-support.html) | Rust project, 2026 | Статус стабилизации |

## 4. Покрытие

**Ключевые выводы**
- По умолчанию используйте cargo-llvm-cov. Он работает на source-based coverage LLVM, даёт точные строки и регионы, поддерживает Linux, macOS и Windows и умеет `cargo llvm-cov nextest`.
- tarpaulin по умолчанию на Linux работает через ptrace (только x86_64). На других ОС он переключается на тот же LLVM-движок, так что для новых проектов преимуществ не даёт.
- Порог задавайте через `--fail-under-lines`. Покрытие — сигнал о непротестированном коде, а не цель сама по себе: 100% строк не гарантирует, что проверено поведение (это проверяет cargo-mutants).
- Branch/MC/DC coverage и покрытие doctest'ов (`--branch`, `--mcdc`, `--doctests`) пока требуют nightly.

| Инструмент | Что и зачем | Установка | Главные команды | Оговорки |
|---|---|---|---|---|
| cargo-llvm-cov (0.9.1) | Покрытие на LLVM instrumentation | `cargo install --locked cargo-llvm-cov` или binstall/Homebrew/Scoop; компонент `llvm-tools-preview` | `cargo llvm-cov --open`; `cargo llvm-cov nextest --lcov --output-path lcov.info`; `--fail-under-lines 80`; `--show-missing-lines`; `--codecov` | Для `--branch`, `--mcdc`, `--doctests` нужен nightly |
| cargo-tarpaulin (0.37) | Альтернативный инструмент покрытия | `cargo install --locked cargo-tarpaulin` | `cargo tarpaulin --out Html`; `--engine llvm` | Бэкенд ptrace работает только на x86_64 Linux |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [cargo-llvm-cov](https://github.com/taiki-e/cargo-llvm-cov) | Taiki Endo, 2026 | Все режимы, интеграция с CI и Codecov |
| [tarpaulin](https://github.com/xd009642/tarpaulin) | Daniel McKenna, 2026 | Если проект уже на нём |

## 5. Зависимости и supply chain

**Ключевые выводы**
- cargo-deny — основной инструмент проверки зависимостей. Он сверяет их с RustSec (advisories), проверяет лицензии, запрещённые крейты и дубликаты версий (bans), а также источники. При нём отдельный `cargo audit` в CI не обязателен.
- `cargo audit bin` проверяет уже собранные бинарники, и лучше всего работает, если сборка шла через `cargo auditable build`.
- Неиспользуемые зависимости ищите cargo-shear: он разбирает исходники парсером rust-analyzer и умеет `--fix`. cargo-machete быстрее, но работает на regex и чаще ошибается. cargo-udeps точнее, но требует nightly и полной сборки.
- Перед каждым релизом библиотеки запускайте cargo-semver-checks. Он сравнивает rustdoc JSON с опубликованной версией и ловит ломающие изменения, которые случайно попали в minor или patch.
- Указанный `rust-version` проверяйте в CI сборкой на этом тулчейне. Определить исходное значение MSRV поможет `cargo msrv find`.
- cargo-vet нужен проектам с формальными требованиями к аудиту зависимостей. Он требует постоянного процесса, а не разового запуска.
- cargo-hakari (workspace-hack) нужен только большим воркспейсам, где разные наборы features заставляют повторно пересобирать одни и те же зависимости.

| Инструмент | Что и зачем | Установка | Главные команды | Оговорки |
|---|---|---|---|---|
| cargo-deny (0.20) | Advisories, лицензии, bans/дубликаты, источники | `cargo install --locked cargo-deny` | `cargo deny init`; `cargo deny check`; `cargo deny check advisories licenses bans sources` | Конфиг в `deny.toml`. Для CI есть `EmbarkStudios/cargo-deny-action` |
| cargo-audit (0.22) | Проверка `Cargo.lock` и бинарников по RustSec | `cargo install cargo-audit` | `cargo audit`; `cargo audit bin <path>`; `--ignore RUSTSEC-…` или `audit.toml` | Без `Cargo.lock` выполняет `cargo update`, а это исполняет код проекта. Чужой проект проверяйте через `--file` |
| cargo-auditable (0.7) | Встраивает список зависимостей в бинарник | `cargo install cargo-auditable` | `cargo auditable build --release` | Работает в паре с `cargo audit bin` и сканерами контейнеров |
| cargo-vet (0.10) | Учёт ручных аудитов крейтов с импортом аудитов других организаций | `cargo install --locked cargo-vet` | `cargo vet init`; `cargo vet`; `cargo vet suggest`; `cargo vet certify` | Существующие зависимости заносятся в exemptions и проходят аудит постепенно |
| cargo-shear (1.14) | Неиспользуемые и не в той секции зависимости, «осиротевшие» файлы | `cargo binstall cargo-shear` / `cargo install cargo-shear` | `cargo shear`; `cargo shear --fix`; `--format=json` | Ложные срабатывания гасите через `[package.metadata.cargo-shear] ignored = [...]`. `--expand` требует nightly |
| cargo-machete (0.9) | Быстрый, но грубый поиск неиспользуемых зависимостей | `cargo install cargo-machete` | `cargo machete`; `--with-metadata`; `--fix` | Ложные срабатывания на переименованных крейтах и макросах |
| cargo-udeps (0.1.61) | Неиспользуемые зависимости по данным компилятора | `cargo install --locked cargo-udeps` | `cargo +nightly udeps --all-targets` | Нужен nightly. Есть сообщения о несовместимости со свежим Cargo [не проверено] |
| cargo-outdated (0.19) | Показывает устаревшие зависимости | `cargo install --locked cargo-outdated` | `cargo outdated -R`; `--workspace`; `--exit-code 1` | Создаёт временный воркспейс и работает медленно |
| cargo-edit (0.13): `cargo upgrade`, `cargo set-version` | Обновляет требования к версиям в `Cargo.toml` | `cargo install cargo-edit` | `cargo upgrade --dry-run`; `cargo upgrade --incompatible`; `cargo set-version --bump minor` | `cargo add` и `cargo rm` давно встроены в Cargo |
| cargo tree | Граф зависимостей | встроено | `cargo tree -d`; `cargo tree -i <crate>`; `cargo tree -e features -i <crate>` | Первый шаг при дубликатах версий и вопросе «откуда эта фича» |
| cargo-semver-checks (0.51) | Линтер SemVer для публичного API | `cargo binstall cargo-semver-checks` / `cargo install --locked cargo-semver-checks` | `cargo semver-checks`; `--baseline-rev main`; `--baseline-version 1.2.0` | Не ловит все нарушения: смену типов полей и параметров, generics и lifetimes, комбинации features. Для CI есть `obi1kenobi/cargo-semver-checks-action@v2` |
| cargo-msrv (0.19) | Поиск и проверка MSRV | `cargo install --locked cargo-msrv` | `cargo msrv find`; `cargo msrv verify`; `cargo msrv list` | Требует rustup и скачивает тулчейны |
| cargo-hakari (0.9.39) | Генерирует workspace-hack крейт, чтобы зависимости собирались с единым набором features | `cargo install --locked cargo-hakari` | `cargo hakari init <name>`; `cargo hakari generate`; `cargo hakari manage-deps`; `cargo hakari verify` | Только для больших воркспейсов. `Cargo.lock` должен быть закоммичен |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [cargo-deny book](https://embarkstudios.github.io/cargo-deny/) | Embark Studios, 2026 | Конфигурация `deny.toml` |
| [RustSec](https://rustsec.org/) и [cargo-audit](https://github.com/rustsec/rustsec/tree/main/cargo-audit) | RustSec WG, 2026 | База уязвимостей, `audit.toml` |
| [cargo-auditable](https://github.com/rust-secure-code/cargo-auditable) | Rust Secure Code WG, 2026 | Аудит готовых бинарников |
| [cargo-vet book](https://mozilla.github.io/cargo-vet/) | Mozilla, 2026 | Процесс аудита, импорт и exemptions |
| [cargo-shear](https://github.com/Boshen/cargo-shear), [cargo-machete](https://github.com/bnjbvr/cargo-machete), [cargo-udeps](https://github.com/est31/cargo-udeps) | Boshen; B. Bouvier; est31, 2026 | Сравнение подходов к поиску неиспользуемых зависимостей |
| [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks) | Predrag Gruevski, 2026 | Лимиты, baseline-опции, CI |
| [cargo-msrv](https://github.com/foresterre/cargo-msrv) | Martijn Gribnau, 2026 | Поиск MSRV |
| [cargo-hakari](https://docs.rs/cargo-hakari/latest/cargo_hakari/) | guppy-rs, 2026 | Когда нужен workspace-hack |
| [cargo-edit](https://github.com/killercup/cargo-edit), [cargo-outdated](https://github.com/kbknapp/cargo-outdated) | killercup; kbknapp, 2026 | Обновление зависимостей |

## 6. Бенчмарки и профилирование

**Ключевые выводы**
- Сначала измеряйте, потом оптимизируйте. Для времени выполнения подходят criterion или divan, для детерминированных счётчиков инструкций в шумном CI — Gungraun, для целых программ — hyperfine.
- Горячие места ищите сэмплирующим профайлером на release-сборке с символами. samply работает на трёх ОС и открывает результат в Firefox Profiler. На Linux есть ещё `perf` и cargo-flamegraph.
- Для профилирования заведите отдельный профиль `profiling` (наследует release, `debug = "line-tables-only"`), а для лучших стеков добавьте `-C force-frame-pointers=yes`.
- Аллокации измеряйте dhat-rs: он умеет утверждать число аллокаций в тестах. На Linux подходит и heaptrack.
- Перед `unsafe`-оптимизациями проверяйте ассемблер через cargo-show-asm: исчезли ли bounds checks, векторизовался ли цикл.
- Размер бинарника разбирайте cargo-bloat, а раздувание от мономорфизации, которое бьёт по времени компиляции, — cargo-llvm-lines.
- Зависшие или «голодающие» async-задачи на Tokio смотрите через tokio-console.

| Инструмент | Что и зачем | Установка | Главные команды | Оговорки |
|---|---|---|---|---|
| criterion (0.8.2) | Статистические бенчмарки времени с детекцией регрессий и HTML-отчётами | `cargo add --dev criterion`, `[[bench]] harness = false` | `cargo bench`; `cargo bench -- <filter>`; `--save-baseline main` / `--baseline main` | Переехал в `criterion-rs/criterion.rs` с новыми мейнтейнерами. Поддерживает три последних stable |
| divan (0.1.21) | Лёгкие бенчмарки на атрибутах: generic-параметры, `AllocProfiler` | `cargo add --dev divan`, `harness = false` | `#[divan::bench(args = [...])]`; `cargo bench`; `cargo bench -- --test` (прогон без измерений в CI) | Последний релиз — апрель 2025. Учитывайте риск при долгосрочном выборе |
| Gungraun (0.20, бывший iai-callgrind) | Однопроходные детерминированные бенчмарки на Valgrind: инструкции, кеш, аллокации | `cargo add --dev gungraun` + `cargo install gungraun-runner` той же версии + Valgrind | `#[library_benchmark]`; `cargo bench` | Linux, macOS только Intel, без Windows. iai-callgrind заморожен на 0.16.1, исходный `iai` [устарело]. Подходит для регрессионного гейта в CI |
| hyperfine (1.21) | Бенчмарк целых команд CLI | `cargo install --locked hyperfine` или пакетный менеджер | `hyperfine --warmup 3 'old' 'new'`; `--export-markdown` | Измеряет процесс целиком, включая запуск |
| samply (0.13) | Сэмплирующий профайлер, UI в Firefox Profiler | `cargo install --locked samply` или скрипт-установщик | `samply record ./target/profiling/app args` | Linux: нужен `perf_event_paranoid` ≤ 1 (или -1). macOS: не профилирует системные подписанные бинарники. Windows: `-a` для записи всей системы |
| cargo-flamegraph (0.6.14) | Flamegraph в SVG одной командой | `cargo install flamegraph` | `cargo flamegraph --bin app -- args`; `--root`; `--dev` | Linux через perf, macOS через xctrace, Windows через blondie или DTrace. С lld (по умолчанию на x86_64 Linux с 1.90) и mold нужен `-C link-arg=-Wl,--no-rosegment`, иначе стеки неверны |
| perf | Системный профайлер Linux | пакет дистрибутива | `perf record -g --call-graph dwarf ./app`; `perf report`; `perf stat` | Только Linux |
| dhat-rs (0.3.3) | Профиль кучи и тесты на число и пик аллокаций | `cargo add dhat` за feature-флагом | `#[global_allocator] static A: dhat::Alloc`; `dhat::Profiler::new_heap()`; `dhat::assert!` | Автор помечает крейт как экспериментальный с низким приоритетом поддержки. Результат смотрите в DHAT viewer |
| heaptrack | Профиль аллокаций без изменения кода | пакет дистрибутива | `heaptrack ./app`; `heaptrack_gui heaptrack.app.*.zst` | Только Linux |
| cargo-show-asm (0.2.63) | Ассемблер, LLVM IR и MIR конкретной функции | `cargo install cargo-show-asm` | `cargo asm -p crate --lib path::to::fn`; `--llvm`; `--mir`; `--rust`; `--mca` | Windows поддерживается ограниченно. Функция должна реально генерироваться: generic без инстанса не покажется |
| cargo-bloat (0.12.1) | Что занимает место в бинарнике | `cargo install cargo-bloat` | `cargo bloat --release -n 20`; `cargo bloat --release --crates` | ELF, Mach-O, PE. Для WASM используйте twiggy. Последний релиз — 2024 |
| cargo-llvm-lines (0.4.48) | Объём LLVM IR по функциям: раздувание generics | `cargo install cargo-llvm-lines` | `cargo llvm-lines --release \| head -20` | Ищите кандидатов на приём «внутренняя не-generic функция» |
| tokio-console (0.1.14) | «top» для Tokio-задач: опросы, простои, waker'ы | `cargo install --locked tokio-console`; в приложении `console-subscriber` | `console_subscriber::init()`; сборка с `RUSTFLAGS="--cfg tokio_unstable"`; `tokio-console` | Только для Tokio. На Windows запускайте в UTF-8 терминале |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The Rust Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote, обновляется | Главы Benchmarking, Profiling, Build Configuration |
| [criterion.rs](https://github.com/criterion-rs/criterion.rs) | criterion-rs, 2026 | Текущий репозиторий и MSRV |
| [divan](https://github.com/nvzqz/divan) | Nikolai Vazquez, 2025 | API и аллокационный профайлер |
| [Gungraun guide](https://gungraun.github.io/gungraun/latest/html/installation/gungraun.html), [bencher.dev: Gungraun](https://bencher.dev/learn/benchmarking/rust/gungraun/) | Gungraun team; Bencher, 2026 | Установка runner'а, Valgrind, CI |
| [hyperfine](https://github.com/sharkdp/hyperfine) | David Peter, 2026 | Сравнение команд |
| [samply](https://github.com/mstange/samply) | Markus Stange, 2026 | Профилирование на трёх ОС |
| [flamegraph](https://github.com/flamegraph-rs/flamegraph) | flamegraph-rs, 2026 | Особенности платформ, `--no-rosegment` |
| [dhat-rs](https://github.com/nnethercote/dhat-rs), [heaptrack](https://github.com/KDE/heaptrack) | Nethercote; KDE | Профилирование кучи |
| [cargo-show-asm](https://github.com/pacak/cargo-show-asm), [cargo-bloat](https://github.com/RazrFalcon/cargo-bloat), [cargo-llvm-lines](https://github.com/dtolnay/cargo-llvm-lines) | pacak; RazrFalcon; dtolnay | Кодоген, размер, мономорфизация |
| [tokio-console](https://github.com/tokio-rs/console) | Tokio, 2025 | Подключение и интерпретация |

## 7. Скорость сборки и релиз

**Ключевые выводы**
- Первый шаг к ускорению — свежий stable. На x86_64 Linux с 1.90 по умолчанию стоит LLD: инкрементальная линковка ускорилась примерно в 7 раз, полная debug-сборка примерно на 20%. На остальных таргетах (включая aarch64 Linux) LLD пока не по умолчанию, там подключайте mold или wild вручную.
- sccache полезен прежде всего в CI и для команды с общим удалённым кешем. Бинарники, proc-macro, cdylib и инкрементальные сборки он не кеширует, поэтому локально выигрыш часто мал.
- Бэкенд Cranelift ускоряет debug-сборки, но доступен только на nightly. Используйте его для локального цикла, а не для релиза.
- Для максимальной скорости выполнения: `lto`, `codegen-units = 1`, затем PGO через cargo-pgo на репрезентативной нагрузке. cargo-wizard применит готовые пресеты.
- Кросс-компиляцию под Linux-таргеты проще всего делать cargo-zigbuild (без Docker, можно выбрать версию glibc). cross нужен, когда требуется полное окружение и запуск тестов под QEMU.
- Выпуск бинарников (installers, Homebrew, GitHub Releases) автоматизирует dist (бывший cargo-dist), версионирование и публикацию крейтов — cargo-release или release-plz.

| Инструмент | Что и зачем | Установка | Главные команды / конфиг | Оговорки |
|---|---|---|---|---|
| sccache (0.18) | Кеш компиляции: локальный, S3, GCS, Azure, Redis, GitHub Actions | `cargo install sccache --locked`, Homebrew, Scoop, winget | `RUSTC_WRAPPER=sccache` или `[build] rustc-wrapper = "sccache"`; `sccache --show-stats` | Не кеширует bin, dylib, cdylib, proc-macro и инкрементальную компиляцию |
| LLD | Быстрый линкер LLVM | встроен (`rust-lld`) | на x86_64 Linux ничего делать не нужно; отключение — `-C linker-features=-lld` | Ручное `-fuse-ld=lld` на x86_64 Linux с 1.90 [устарело] |
| mold | Самый быстрый на практике линкер для ELF | пакет дистрибутива | `.cargo/config.toml`: `[target.x86_64-unknown-linux-gnu] linker = "clang"`, `rustflags = ["-C", "link-arg=-fuse-ld=mold"]`; или `mold -run cargo build` | Только Linux |
| wild (0.10) | Быстрый линкер на Rust, нацелен на инкрементальную линковку | `cargo install --locked wild-linker` | `rustflags = ["-Clink-arg=-fuse-ld=wild"]` (GCC 16.1+ или clang); либо `linker = "clang"` + `-Clink-arg=--ld-path=wild` | Linux x86-64, AArch64, RISC-V, LoongArch. Инкрементальная линковка ещё не реализована. Перед продакшен-сборками проверяйте результат |
| Cranelift backend | Быстрая кодогенерация для debug | `rustup component add rustc-codegen-cranelift-preview --toolchain nightly` | `.cargo/config.toml`: `[unstable] codegen-backend = true`, `[profile.dev] codegen-backend = "cranelift"` | Только nightly. Linux/macOS/Windows x86_64 и Linux/macOS AArch64. SIMD неполный, по умолчанию `panic=abort` |
| cargo-pgo (0.3) | PGO и BOLT в три шага | `cargo install cargo-pgo` + `rustup component add llvm-tools-preview` | `cargo pgo build` → прогон нагрузки → `cargo pgo optimize build`; `cargo pgo bolt build --with-pgo`; `cargo pgo info` | BOLT экспериментальный и ориентирован на Linux. Нагрузка при профилировании должна быть репрезентативной |
| cargo-wizard (0.2.3) | Пресеты профилей: fast-compile, fast-runtime, min-size | `cargo install cargo-wizard` | `cargo wizard`; `cargo wizard apply fast-runtime release` | Пресеты опиняционные. Меняет `Cargo.toml` и `.cargo/config.toml`, ревьюйте дифф |
| dist (бывший cargo-dist, 0.32–0.33) | Релизный пайплайн бинарников: CI, shell/PowerShell-установщики, Homebrew, npm, MSI | `cargo install cargo-dist` или скрипт-установщик | `dist init`; `dist plan`; `dist build`; `dist generate` | Генерирует собственный CI-workflow. Перед внедрением проверьте активность проекта |
| cargo-release (1.1.6) | Выпуск крейтов: bump версии, тег, `cargo publish`, push | `cargo install cargo-release` | `cargo release patch` (по умолчанию dry-run); `cargo release minor --execute`; `--workspace` | Альтернатива — [release-plz](https://release-plz.dev/): релизный PR из CI с changelog и встроенным semver-checks |
| cargo-zigbuild (0.23) | Кросс-линковка через `zig cc` | `cargo install --locked cargo-zigbuild` (+ Zig) или `pip install cargo-zigbuild` | `cargo zigbuild --target aarch64-unknown-linux-gnu`; `--target x86_64-unknown-linux-gnu.2.17` (версия glibc); `--target universal2-apple-darwin` | Таргеты только Linux и macOS. Нужен `rustup target add` |
| cross (0.2.5) | Кросс-сборка и тесты в контейнерах с QEMU | `cargo install cross --git https://github.com/cross-rs/cross` | `cross build --target aarch64-unknown-linux-gnu`; `cross test --target …` | Нужен Docker 20.10+ или Podman 3.4+. Релиз на crates.io старый (2023), ставьте из git |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Faster linking with LLD in 1.90](https://blog.rust-lang.org/2025/09/01/rust-lld-on-1.90.0-stable) | Rémy Rakic, 2025 | Цифры и отключение LLD |
| [sccache](https://github.com/mozilla/sccache) | Mozilla, 2026 | Бэкенды и ограничения для Rust |
| [mold](https://github.com/rui314/mold), [wild](https://github.com/wild-linker/wild) | Rui Ueyama; David Lattimore, 2026 | Конфигурация альтернативных линкеров |
| [rustc_codegen_cranelift](https://github.com/rust-lang/rustc_codegen_cranelift) | bjorn3, 2026 | Включение и ограничения Cranelift |
| [cargo-pgo](https://github.com/Kobzol/cargo-pgo), [cargo-wizard](https://github.com/Kobzol/cargo-wizard) | Jakub Beránek, 2026 | PGO/BOLT и пресеты профилей |
| [Cargo profiles](https://doc.rust-lang.org/cargo/reference/profiles.html), [Cargo config](https://doc.rust-lang.org/cargo/reference/config.html) | Rust project, 2026 | Все ключи профилей и `.cargo/config.toml` |
| [dist](https://axodotdev.github.io/cargo-dist/) | axo, 2026 | Релиз бинарников |
| [cargo-release](https://github.com/crate-ci/cargo-release), [release-plz](https://release-plz.dev/) | crate-ci (Ed Page); Marco Ieni, 2026 | Версионирование и публикация |
| [cargo-zigbuild](https://github.com/rust-cross/cargo-zigbuild), [cross](https://github.com/cross-rs/cross) | rust-cross; cross-rs, 2026 | Кросс-компиляция |

## 8. Понимание кода

**Ключевые выводы**
- Когда ошибка указывает внутрь derive или `macro_rules!`, раскройте макросы через `cargo expand path::to::item`. Это быстрее, чем гадать, что сгенерировалось.
- В незнакомом крейте начните с `cargo modules structure`: дерево модулей с видимостью сразу показывает публичную поверхность и связи.
- `cargo public-api` выводит публичный API текстом. Его удобно снапшотить в тестах или сравнивать между версиями.
- cargo-geiger показывает статистику `unsafe` по дереву зависимостей. Это вход для аудита, а не вердикт о безопасности. Обновляется он редко (0.13.0, август 2025).

| Инструмент | Что и зачем | Установка | Главные команды | Оговорки |
|---|---|---|---|---|
| cargo-expand (1.0) | Показывает код после раскрытия макросов | `cargo install cargo-expand` | `cargo expand`; `cargo expand path::to::module`; `--lib`, `--test name`; `--ugly` | Использует `-Zunpretty=expanded`, поэтому нужен установленный nightly. Вывод — отладочное средство: гигиена макросов в нём теряется |
| cargo-modules (0.27) | Дерево модулей, граф зависимостей модулей, orphan-файлы | `cargo install cargo-modules` | `cargo modules structure --package x`; `cargo modules dependencies` (DOT); `cargo modules orphans` | Лицензия MPL-2.0 |
| cargo-public-api (0.52) | Список и дифф публичного API | `cargo +stable install cargo-public-api` | `cargo public-api`; `cargo public-api diff latest`; `diff v1.0.0..v1.1.0`; `-s` (без blanket/auto impl) | Для rustdoc JSON нужен nightly (не обязательно активный) |
| cargo-geiger (0.13) | Счётчик `unsafe` в крейте и зависимостях | `cargo install --locked cargo-geiger` | `cargo geiger` | Медленный, редкие релизы. Используйте как статистику |

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [cargo-expand](https://github.com/dtolnay/cargo-expand) | David Tolnay, 2026 | Раскрытие макросов |
| [cargo-modules](https://github.com/regexident/cargo-modules) | Vincent Esche, 2026 | Навигация по структуре крейта |
| [cargo-public-api](https://github.com/cargo-public-api/cargo-public-api) | Martin Nordholts и др., 2026 | Контроль публичного API |
| [cargo-geiger](https://github.com/geiger-rs/cargo-geiger) | geiger-rs, 2025 | Аудит `unsafe` в зависимостях |

## 9. Документация

**Ключевые выводы**
- В библиотеках включайте `missing_docs = "warn"` через `[workspace.lints.rust]` и проверяйте документацию в CI: `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps --all-features`. Битые intra-doc ссылки (`[`Type`]`) тогда ломают сборку.
- Примеры в `///` — это doctest'ы, они компилируются и запускаются через `cargo test --doc`. Для кода, который нельзя запускать, есть `no_run`, для кода, который должен не компилироваться, — `compile_fail`, для псевдокода — `ignore` (с пояснением). Строки с `# ` скрываются из вывода, но компилируются.
- У каждой `pub unsafe fn` пишите раздел `# Safety`, у функций с `Result` — `# Errors`, у функций, способных паниковать, — `# Panics`. Первый проверяет `clippy::missing_safety_doc` (включён по умолчанию), два других — pedantic-линты.
- Сборку на docs.rs настраивайте в `[package.metadata.docs.rs]`. Для пометок «доступно с feature X» добавьте `#![cfg_attr(docsrs, feature(doc_cfg))]`. Отдельный `doc_auto_cfg` удалён и влит в `doc_cfg`, старые рецепты с ним [устарело].

```toml
[package.metadata.docs.rs]
all-features = true                  # или features = ["serde", "tokio"]
rustdoc-args = ["--cfg", "docsrs"]   # включает #[cfg(docsrs)] и doc_cfg
targets = ["x86_64-unknown-linux-gnu"]  # меньше таргетов — быстрее сборка на docs.rs
```

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The rustdoc book](https://doc.rust-lang.org/rustdoc/) | Rust project, 2026 | Атрибуты, линты rustdoc, intra-doc ссылки |
| [Documentation tests](https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html) | Rust project, 2026 | `no_run`, `compile_fail`, скрытые строки |
| [The `#[doc]` attribute](https://doc.rust-lang.org/rustdoc/write-documentation/the-doc-attribute.html) | Rust project, 2026 | `doc(hidden)`, `doc(alias)`, inline |
| [docs.rs metadata](https://docs.rs/about/metadata) | docs.rs team, 2026 | Ключи `[package.metadata.docs.rs]` |
| [RFC 3631: rustdoc cfg handling](https://rust-lang.github.io/rfcs/3631-rustdoc-cfgs-handling.html) | Guillaume Gomez, 2024–2025 | Новая модель `doc_cfg` и `auto_cfg` |

## 10. Минимальный набор и инструменты по ситуации

**Ключевые выводы**
- Минимальный набор для любого проекта — `rust-toolchain.toml`, rustfmt, Clippy с `[workspace.lints]`, rust-analyzer, `cargo nextest` (плюс `cargo test --doc`) и `cargo deny`. Всё это ставится за минуты и окупается с первого PR.
- Для библиотек к этому добавляются cargo-semver-checks, `missing_docs` с `cargo doc -D warnings` и проверка MSRV.
- Для кода с `unsafe` добавляются Miri и `undocumented_unsafe_blocks`, а для собственных примитивов синхронизации — Loom.
- Остальное подключайте под конкретную боль, а не заранее. Каждый инструмент в CI стоит минут на каждом PR.

| Ситуация | Инструменты |
|---|---|
| Всегда | rustup + `rust-toolchain.toml`, rustfmt, Clippy, rust-analyzer, bacon (локально), cargo-nextest, cargo-deny, `cargo tree` |
| Библиотека на crates.io | + cargo-semver-checks (или cargo-public-api), cargo-msrv, docs.rs metadata, cargo-release/release-plz |
| `unsafe`, FFI | + Miri, санитайзеры (nightly, Linux/macOS), `undocumented_unsafe_blocks`, cargo-geiger для зависимостей, Kani для критичных функций |
| Конкурентность, lock-free | + Loom (shuttle для больших сценариев), Miri (гонки данных) |
| Парсеры, недоверенный ввод | + cargo-fuzz или afl.rs, proptest |
| Сложные выводы, CLI, компиляторы | + insta |
| Качество тестов | + cargo-llvm-cov, cargo-mutants (на диффе) |
| Производительность | + criterion или divan, Gungraun в CI, samply/flamegraph, dhat-rs, cargo-show-asm, hyperfine |
| Async на Tokio | + tokio-console |
| Медленная сборка | + `cargo build --timings`, mold или wild (Linux), Cranelift (nightly), sccache (CI), cargo-llvm-lines, cargo-hakari (большие воркспейсы), cargo-shear |
| Релиз бинарников | + dist, cargo-zigbuild или cross, cargo-auditable, cargo-bloat |
| Повышенные требования к supply chain | + cargo-vet, cargo-audit bin |

## 11. Порядок CI-пайплайна

**Ключевые выводы**
- Шаги идут от дешёвых к дорогим и быстро прерываются при ошибке: формат и линты за секунды, тесты за минуты, покрытие и бенчмарки — дольше всего.
- Покрытие и бенчмарки выносите в отдельные job'ы. Бенчмарки по времени на общих раннерах шумные, поэтому как гейт используйте Gungraun (счётчик инструкций), а criterion и divan — для трендов.
- Матрица ОС (Linux, macOS, Windows) нужна только шагу тестов. fmt, clippy, deny и semver-checks достаточно гонять на одной ОС.
- Отдельный job собирает проект на MSRV-тулчейне, а nightly-проверки (Miri, санитайзеры, fuzz-смоук) лучше запускать по расписанию или в отдельном job'е.

Порядок шагов:
1. `cargo fmt --all --check`
2. `cargo clippy --workspace --all-targets --all-features -- -D warnings`
3. `cargo nextest run --workspace --all-features --profile ci` (матрица ОС)
4. `cargo test --workspace --doc`
5. `RUSTDOCFLAGS="-D warnings" cargo doc --workspace --no-deps --all-features`
6. `cargo deny check` (и/или `cargo audit`)
7. Для библиотек: `cargo semver-checks` и сборка на MSRV (`cargo +1.85 check`, где 1.85 — ваш `rust-version`)
8. `cargo llvm-cov nextest --workspace --lcov --output-path lcov.info` (отдельный job)
9. Бенчмарки: Gungraun как гейт, criterion или divan для трендов (отдельный job или по расписанию)
10. По расписанию: `cargo +nightly miri test`, `cargo mutants --in-diff`, fuzz-смоук, `cargo outdated`

```yaml
# Набросок GitHub Actions: только ключевые шаги
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: rustup toolchain install          # читает rust-toolchain.toml
      - uses: Swatinem/rust-cache@v2
      - uses: taiki-e/install-action@v2
        with: { tool: "cargo-nextest,cargo-deny,cargo-semver-checks" }
      - run: cargo fmt --all --check
      - run: cargo clippy --workspace --all-targets --all-features -- -D warnings
      - run: cargo nextest run --workspace --all-features
      - run: cargo test --workspace --doc
      - run: cargo doc --workspace --no-deps --all-features
        env: { RUSTDOCFLAGS: "-D warnings" }
      - run: cargo deny check
      - run: cargo semver-checks              # только для библиотек
```

Версии actions (`@v4`, `@v2`) сверяйте с их README в момент настройки.

## 12. Шаблоны: `[workspace.lints]` и профили

**Ключевые выводы**
- Группы линтов ставьте с `priority = -1`, отдельные линты — с приоритетом 0 по умолчанию. Иначе порядок применения не определён и `allow` может не сработать.
- Используйте `deny`, а не `forbid`, если какому-то модулю понадобится локальный `#[allow(..., reason = "...")]`: `forbid` перекрыть нельзя.
- Крейт с `[lints] workspace = true` не может переопределить отдельный линт в своём `Cargo.toml`. Делайте это атрибутом в `lib.rs` или `main.rs`.
- `panic = "abort"` ставьте, только если вам не нужны `catch_unwind` и разматывание (например, плагины и FFI-границы с восстановлением). Тесты всегда собираются с unwind.
- `bench` по умолчанию наследует `release`. Включайте в нём `debug`, чтобы профайлеры видели символы. На скорость кода это не влияет.

```toml
# Cargo.toml (корень воркспейса)
[workspace.lints.rust]
unsafe_code = "deny"            # крейты с обоснованным unsafe: #![allow(unsafe_code, reason = "...")]
missing_docs = "warn"           # для библиотек; в бинарниках можно убрать
unreachable_pub = "warn"        # pub, который не виден снаружи крейта, — сигнал к pub(crate)

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
missing_errors_doc = "allow"    # шумные pedantic-линты
missing_panics_doc = "allow"
must_use_candidate = "allow"
# Выбранные restriction-линты
undocumented_unsafe_blocks = "warn"
multiple_unsafe_ops_per_block = "warn"
allow_attributes_without_reason = "warn"
dbg_macro = "warn"
todo = "warn"
unwrap_used = "warn"            # с allow-unwrap-in-tests = true в clippy.toml

[workspace.lints.rustdoc]
broken_intra_doc_links = "deny"

# в каждом crates/*/Cargo.toml:
# [lints]
# workspace = true
```

```toml
[profile.dev]
debug = "line-tables-only"      # быстрее сборка и линковка, номера строк в бэктрейсах остаются

[profile.dev.package."*"]
opt-level = 2                   # оптимизировать только зависимости (игры, численный код)

[profile.release]
lto = "thin"                    # "fat" — ещё несколько процентов ценой времени сборки
codegen-units = 1               # лучше оптимизация, медленнее сборка
panic = "abort"                 # меньше и быстрее бинарник; без catch_unwind
# strip = "debuginfo" — по умолчанию в release, если debug не включён

[profile.profiling]             # cargo build --profile profiling
inherits = "release"
debug = "line-tables-only"      # символы и строки для samply/perf/flamegraph

[profile.bench]                 # наследует release
debug = true                    # символы для профайлеров при cargo bench
```

```toml
# clippy.toml
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-dbg-in-tests = true
# msrv берётся из rust-version в Cargo.toml; задавайте здесь, только если они должны различаться
```

## 13. Ресурсы

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The rustup book](https://rust-lang.github.io/rustup/) | Rust project, 2026 | Тулчейны, компоненты, таргеты, overrides |
| [Announcing Rustup 1.29.0](https://blog.rust-lang.org/2026/03/12/Rustup-1.29.0), [1.29.1](https://blog.rust-lang.org/2026/09/01/Rustup-1.29.1/) | Rustup team, 2026 | Параллельные загрузки, депрекация неявной установки |
| [Announcing Rust 1.99.0](https://blog.rust-lang.org/2026/10/01/Rust-1.99.0/) | Rust release team, 2026 | Текущий stable |
| [The Cargo Book: profiles](https://doc.rust-lang.org/cargo/reference/profiles.html) | Rust project, 2026 | Ключи профилей и наследование |
| [The Cargo Book: config](https://doc.rust-lang.org/cargo/reference/config.html) | Rust project, 2026 | `.cargo/config.toml`: линкер, `rustflags`, `rustc-wrapper` |
| [The Cargo Book: `[workspace.lints]`](https://doc.rust-lang.org/cargo/reference/workspaces.html) | Rust project, 2026 | Наследование линтов |
| [`cargo tree`](https://doc.rust-lang.org/cargo/commands/cargo-tree.html), [`cargo update`](https://doc.rust-lang.org/cargo/commands/cargo-update.html) | Rust project, 2026 | Анализ и обновление зависимостей |
| [rustc: allowed-by-default lints](https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html) | Rust project, 2026 | Линты rustc, которые стоит включить (`unreachable_pub`, `missing_docs`) |
| [Clippy lints](https://rust-lang.github.io/rust-clippy/stable/index.html), [Clippy README](https://github.com/rust-lang/rust-clippy) | Clippy team, 2026 | Группы и уровни |
| [rustfmt](https://rust-lang.github.io/rustfmt/) | rustfmt team, 2026 | Опции и их стабильность |
| [rust-analyzer](https://rust-analyzer.github.io/) | rust-analyzer team, 2026 | Настройки LSP |
| [The rustdoc book](https://doc.rust-lang.org/rustdoc/) | Rust project, 2026 | Документация и doctest'ы |
| [The Rust Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote, обновляется | Бенчмарки, профилирование, профили сборки |
| [Tips for Faster Rust Compile Times](https://corrode.dev/blog/tips-for-faster-rust-compile-times/) | Matthias Endler, обновляется | Ускорение сборки и CI |
| [nexte.st](https://nexte.st/) | nextest-rs, 2026 | Раннер тестов |
| [cargo-deny book](https://embarkstudios.github.io/cargo-deny/) | Embark Studios, 2026 | Проверка зависимостей |
| [Rust Fuzz Book](https://rust-fuzz.github.io/book/cargo-fuzz/setup.html) | rust-fuzz, 2026 | Fuzzing |
| [Miri](https://github.com/rust-lang/miri) | Rust project, 2026 | Поиск UB |
