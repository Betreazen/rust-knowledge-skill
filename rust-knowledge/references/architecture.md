# Архитектура Rust-проектов

Как раскладывать код Rust-проекта любого размера: workspace и границы крейтов, модули и видимость, направление зависимостей, ports & adapters, ошибки по слоям, конфигурация, feature flags, тестовая архитектура, влияние разбиения на время сборки и документирование архитектуры. В конце даны эталонные кодовые базы, таблица разногласий между источниками с рекомендацией по умолчанию и чеклист для нового проекта. Состояние на октябрь 2026 года. Самое конкретное руководство по-прежнему серия matklad 2021 года «[One Hundred Thousand Lines of Rust](https://matklad.github.io/2021/09/05/Rust100k.html)». С 2024 по 2026 год разговор сместился от раскладки workspace (вопрос решён) к масштабированию времени компиляции и практикам сопровождения.

- [Workspace и границы крейтов](#workspace-и-границы-крейтов)
- [Модули и видимость](#модули-и-видимость)
- [Направление зависимостей и слои](#направление-зависимостей-и-слои)
- [Ports & adapters (гексагональная архитектура)](#ports--adapters-гексагональная-архитектура)
- [Ошибки по слоям](#ошибки-по-слоям)
- [Конфигурация и глобальное состояние](#конфигурация-и-глобальное-состояние)
- [Feature flags](#feature-flags)
- [Архитектура тестов](#архитектура-тестов)
- [Разбиение на крейты и время сборки](#разбиение-на-крейты-и-время-сборки)
- [ARCHITECTURE.md](#architecturemd)
- [Кодовые базы для изучения](#кодовые-базы-для-изучения)
- [Где источники расходятся](#где-источники-расходятся)
- [Чеклист: структура нового проекта](#чеклист-структура-нового-проекта)

## Workspace и границы крейтов

### Ключевые выводы

- Начинайте с одного пакета: `lib.rs` с логикой и тонкий `main.rs`. Переходите на workspace, когда кода больше ~10k LOC или компиляция начинает мешать.
- Workspace делайте плоским: корневой `Cargo.toml` — виртуальный манифест с `members = ["crates/*"]`, все крейты (включая главный) на одном уровне в `crates/`, имя папки совпадает с именем крейта. Иерархия нужна только после ~100 крейтов (~1M LOC).
- Граф крейтов делайте широким, а не глубоким: общий «словарный» крейт типов → независимые крейты-фичи → листовой бинарник, который всё связывает. Линейные цепочки A→B→C→D сериализуют сборку.
- Бинарники держите тонкими: они только собирают приложение из библиотек. Это упрощает тесты и переиспользование.
- Версии зависимостей, общие метаданные и lint централизуйте через `[workspace.dependencies]`, `[workspace.package]` и `[workspace.lints]` (в крейтах — `foo.workspace = true`). Наследование доступно с Rust 1.64, `[workspace.lints]` — с 1.74.
- Clippy настраивайте таблицей `[workspace.lints]` (например, `pedantic` с отключёнными шумными lint вроде `missing_errors_doc`, `module_name_repetitions`), а не `#![deny(...)]` в корне крейта. `-D warnings` ставьте в CI, а не в код.
- Зависимости — это обязательства: берите немногие, популярные, версии ≥1.0; обновляйте, а не пиньте навсегда.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [One Hundred Thousand Lines of Rust](https://matklad.github.io/2021/09/05/Rust100k.html) | Aleksey Kladov (matklad), 2021 | Индекс серии: ARCHITECTURE.md, Delete Cargo Integration Tests, How to Test, Inline In Rust, Large Rust Workspaces, Fast Rust Builds |
| [Large Rust Workspaces](https://matklad.github.io/2021/08/22/large-rust-workspaces.html) | matklad, 2021 | Плоская раскладка `crates/*` для 10k–1M LOC. Огромные монорепозитории живут по другим правилам |
| [Cargo Book: Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) | Cargo team, живой | Виртуальные манифесты, общий `Cargo.lock` и `target/`, наследование `workspace.*` |
| [Clippy pedantic на уровне workspace](https://coreyja.com/notes/clippy-pedantic-workspace) | coreyja | Как настроить `[workspace.lints]` вместо атрибутов в коде |
| [Long-term Rust Project Maintenance](https://corrode.dev/blog/long-term-rust-maintenance/) | Matthias Endler, 2024 | Зависимости как обязательства, узкая публичная поверхность, `#[non_exhaustive]`, release-plz, ежеквартальные ревью сопровождения |
| [Making Your First Real-World Rust Project a Success](https://corrode.dev/blog/successful-rust-business-adoption-checklist/) | Matthias Endler | Низкая связность и высокая связанность между модулями и крейтами; бизнес-логика отдельно от инфраструктуры |
| [scaling codebase 50k loc → 500k loc](https://users.rust-lang.org/t/soft-question-scaling-codebase-50k-loc-500k-loc/104129) | users.rust-lang.org | Обсуждение сообществом роста кодовой базы [не проверено] |

## Модули и видимость

### Ключевые выводы

- Источник истины — дерево модулей, а не файловая система: каждый модуль объявляется через `mod` в родителе. Элементы по умолчанию приватны.
- Внутри крейта по умолчанию используйте `pub(crate)`. Курированное `pub` оставляйте фасадным крейтам и явным API-границам.
- Модули группируйте по предметной области (фичам), а не по типу кода (`models/`, `utils/`).
- Публичные enum и структуры с расчётом на развитие помечайте `#[non_exhaustive]` и держите поля приватными. Это позволяет добавлять варианты и поля без semver-break.
- Трейт, который нельзя реализовывать снаружи, делайте sealed: тогда в него можно добавлять методы без semver-break.
- Не используйте glob-импорты и самодельные prelude во внутреннем коде: они скрывают, откуда пришло имя.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [The Rust Book, гл. 7](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html) | Rust project, живой | `mod`/`pub`/`use`, `pub(crate)`, паттерн «binary + library crate» |
| [Clear explanation of Rust's module system](https://www.sheshbabu.com/posts/rust-module-system/) | Sheshbabu Chinnakonda, 2020 | Дерево модулей против файловой системы ([обсуждение HN](https://news.ycombinator.com/item?id=23889427)) |
| [Rust API Guidelines: Future proofing](https://rust-lang.github.io/api-guidelines/future-proofing.html) | Rust libs team | Sealed traits, приватные поля, `#[non_exhaustive]` |
| [A definitive guide to sealed traits](https://predr.ag/blog/definitive-guide-to-sealed-traits-in-rust/) | Predrag Gruevski, 2023 | Три уровня запечатывания и частичное запечатывание |
| [Practical Rust API Design](https://corrode.dev/blog/practical-rust-api-design/) | Matthias Endler, 2026 | Инварианты в сигнатурах, пары owned/borrowed хэндлов, `#[non_exhaustive]` с escape-вариантом |
| [Don't Use Preludes And Globs](https://corrode.dev/blog/dont-use-preludes-and-globs/) | Matthias Endler | Почему явные импорты лучше в долгоживущем коде |

## Направление зависимостей и слои

### Ключевые выводы

- Зависимости направляйте внутрь: ядро (домен, типы, алгоритмы) не знает об IO, сериализации, сети, UI и фреймворках. Внешние крейты зависят от ядра, а не наоборот.
- IO, сериализацию, глобальную конфигурацию и установку tracing-subscriber держите только во внешних (верхних) крейтах. В rust-analyzer JSON знает только верхний крейт, а ядро не делает IO.
- Явно называйте API-границы и фиксируйте их инварианты. Пример из rust-analyzer: «Internal `hir-*` crates are not, and will never be, an api boundary».
- Над внутренними крейтами ставьте фасадный крейт (`hir`, `ide` в rust-analyzer): потребители зависят от фасада, внутренности можно менять свободно.
- Ядро может возвращать значение вместе с собранными ошибками (`(T, Vec<Error>)`), а не падать на первой. Это удобно для анализаторов, компиляторов и валидаторов.
- Functional core / imperative shell: чистые операции над данными в ядре, побочные эффекты в оболочке. Так ядро тестируется без моков.
- Для изоляции сбоев в долгоживущем сервисе допустим `catch_unwind` на уровне отдельного запроса, как в rust-analyzer.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [rust-analyzer Architecture](https://rust-analyzer.github.io/book/contributing/architecture.html) | команда rust-analyzer, живой | Эталон слоёв: `syntax` → `base-db` → `hir-*` → фасад `hir` → `ide*` → LSP-сервер. Именованные API-границы, инварианты, отсутствие IO в ядре |
| [Fast Rust Builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html) | matklad, 2021 | Форма графа: словарный крейт → независимые фичи → листовой крейт; serde и proc-macro ближе к краям |
| [Study of std::io::Error](https://matklad.github.io/2020/10/15/study-of-std-io-error.html) | matklad, 2020 | Образец API на границе: kind + opaque repr + payload |

## Ports & adapters (гексагональная архитектура)

### Ключевые выводы

- Порт — это трейт (например, `AuthorRepository`), который скрывает конкретную зависимость вроде `sqlx::SqlitePool`. Адаптеры делятся на входные (HTTP, CLI) и выходные (БД, внешние API).
- Раскладка: `domain` / `inbound` / `outbound` — модулями в одном крейте или отдельными крейтами в workspace.
- Внедряйте зависимости без фреймворка: через параметры конструктора и generics (`AppState<AR: AuthorRepository>`) или `Arc<dyn Trait>`, когда нужна подмена во время выполнения или важна скорость сборки.
- Значения домена делайте валидированными newtype (`AuthorName`): если значение существует, оно корректно («parse, don't validate»).
- Ошибки — исчерпывающий enum на каждую операцию (`CreateAuthorError::Duplicate | Unknown`), который входной адаптер транслирует в HTTP-ответ.
- Подход избыточен для тривиального CRUD, утилит-преобразователей и горячего кода, чувствительного к стоимости преобразований между слоями: он добавляет boilerplate. Сам автор главного гайда об этом предупреждает.
- Применяйте его, когда домен богаче транспорта: много бизнес-правил, несколько входов или хранилищ, долгий срок жизни. Успех зависит от дисциплины команды, а не от раскладки папок.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Master Hexagonal Architecture in Rust](https://www.howtocodeit.com/guides/master-hexagonal-architecture-in-rust) | How To Code It, 2024 (гайд v1.1.7, дописывается) | Главный текст по теме: порты-трейты, DI через generics, newtype, ошибки по операциям, честный раздел о том, когда подход не нужен |
| [howtocodeit/hexarch](https://github.com/howtocodeit/hexarch) | How To Code It, 2024 | Код по веткам: «1-very-bad-app» → «2-slightly-better-app» (слоистая) → «3-simple-service» (гексагональная) |
| [Zero To Production In Rust](https://www.zero2prod.com/) | Luca Palmieri, 2022 | Та же идея на практике: `lib.rs` + тонкий `main.rs`, доменные типы, black-box тесты. Книга на actix-web, есть [порт на axum](https://github.com/mattiapenati/zero2prod) |
| [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) | Alexis King, 2019 | Валидация возвращает тип-доказательство. Примеры на Haskell, идея вне времени |
| [Make Illegal States Unrepresentable](https://corrode.dev/blog/illegal-state/) | Matthias Endler | Enum с данными вместо флагов и `Option` |

## Ошибки по слоям

### Ключевые выводы

- Ошибки служат двум целям: управлению потоком (их матчит вызывающий код) и отчётности (их читает оператор). Проектируйте тип под цель.
- Библиотеки и доменный слой: типизированные enum-ошибки (thiserror или snafu), которые вызывающий код может разобрать. Внешние публичные enum — с `#[non_exhaustive]`.
- Приложение и его край: непрозрачные ошибки с контекстом (anyhow или eyre) для логов и отчёта.
- Отдельный тип ошибки на операцию лучше одного большого enum на весь крейт: большой общий enum заставляет вызывающих обрабатывать невозможные варианты и считается лёгким запахом.
- Трансляция ошибок происходит на границе слоя: домен не знает о HTTP-кодах, входной адаптер переводит доменную ошибку в ответ.
- Panic уместна для багов и нарушенных инвариантов, в тестах и примерах, но не для ожидаемых сбоев ввода или окружения.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Error Handling In Rust: A Deep Dive](https://www.lpalmieri.com/posts/error-handling-rust/) | Luca Palmieri, ~2021 | Ошибки для потока управления и для отчётности; thiserror против anyhow |
| [Modular Errors in Rust](https://sabrinajewson.org/blog/errors) | Sabrina Jewson, 2023 | Тип ошибки на операцию вместо одного enum на крейт |
| [Error handling in iroh](https://www.iroh.computer/blog/error-handling-in-iroh) | n0, 2025 | Ошибки и backtrace в библиотеке на практике, сдвиг к snafu-подобному контексту |
| [Using unwrap() in Rust is Okay](https://burntsushi.net/unwrap/) | Andrew Gallant, 2022 | Когда panic оправдана |
| [Study of std::io::Error](https://matklad.github.io/2020/10/15/study-of-std-io-error.html) | matklad, 2020 | Расширяемая ошибка: kind + opaque repr + payload |

## Конфигурация и глобальное состояние

### Ключевые выводы

- Библиотеки не настраивают глобальное состояние: не ставят tracing-subscriber, логгер, глобальный аллокатор, обработчик panic. Это делает только бинарник.
- Библиотеки эмитят события через `tracing`, а subscriber устанавливает приложение. То же правило, что и для IO: только во внешнем слое.
- Конфигурацию читайте один раз на старте во внешнем слое (файл, переменные окружения), разбирайте в типизированную структуру и передавайте внутрь явно. Ядро не читает переменные окружения само.
- Отсутствующие обязательные параметры и секреты проверяйте при старте и падайте с понятным сообщением.
- Когда поведение должно переключаться без пересборки, используйте runtime-конфигурацию, а не взаимоисключающие cargo-фичи.
- Модуль `configuration` рядом со `startup` и тонким `main.rs`, как в Zero To Production, — разумная отправная точка для сервиса [не проверено: точные имена модулей книги].

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [tracing](https://github.com/tokio-rs/tracing) ([docs](https://docs.rs/tracing)) | tokio-rs | Структурированная span-инструментация; разделение «библиотека эмитит, бинарник подписывается» |
| [Zero To Production In Rust](https://www.zero2prod.com/) | Luca Palmieri, 2022 | Конфигурация, запуск приложения, телеметрия в одном сервисе |
| [rust-analyzer Architecture](https://rust-analyzer.github.io/book/contributing/architecture.html) | команда rust-analyzer | Сериализация и IO только в верхнем крейте |

## Feature flags

### Ключевые выводы

- Фичи должны быть аддитивными: включение фичи не должно отключать функциональность. Cargo унифицирует фичи по всему графу, поэтому любая фича может быть включена за вас.
- Избегайте взаимоисключающих фич. Если без них никак, ставьте `compile_error!`, но лучше разделите пакеты или используйте runtime-конфигурацию.
- Из-за унификации делайте фичу `std` (включает возможности), а не фичу `no_std` (отключает).
- Удаление фичи или перенос публичного кода за фичу — это semver-breaking изменение.
- Resolver 2 не унифицирует фичи для build-dependencies, proc-macro, dev-dependencies и чужих таргетов, но может собрать одну зависимость дважды. Дубли ищите через `cargo tree --duplicates`. В виртуальном манифесте `resolver` задаётся явно.
- Для публикуемой библиотеки удобна схема «фасадный крейт + подкрейты за фичами» (bevy, tokio, polars). Proc-macro выносите в отдельный крейт.
- В большом workspace разные наборы фич у членов приводят к пересборке одной и той же зависимости. Это лечит workspace-hack крейт (cargo-hakari).

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Cargo Book: Features](https://doc.rust-lang.org/cargo/reference/features.html) | Cargo team, живой | Нормативный источник: аддитивность, унификация, resolver 2, semver |
| [cargo-hakari](https://docs.rs/cargo-hakari/latest/cargo_hakari/about/index.html) | guppy-rs | Workspace-hack крейт с единым набором фич; на omicron от Oxide суммарное ускорение до 1.7x (418 с против 717 с), отдельные команды 1.1x–100x ([замеры](https://github.com/guppy-rs/hakari-on-omicron-perf)) |
| [Effective Rust](https://effective-rust.com/) | David Drysdale, 2024 | Глава Dependencies: semver и feature flags |

## Архитектура тестов

### Ключевые выводы

- Каждый файл `tests/*.rs` — отдельный бинарник, который заново линкует библиотеку. Сводите интеграционные тесты в один бинарник `tests/it/main.rs` с подмодулями: у Cargo это сократило время компиляции тестов в 3 раза и артефакты на диске в 5 раз.
- Для внутренних (непубликуемых) крейтов держите тесты в `#[cfg(test)] mod tests;` отдельным файлом (правка теста не пересобирает библиотеку) и ставьте `doctest = false`.
- Тестируйте поведение через стабильную точку входа, а не внутреннее API: data-driven тесты через одну функцию `check(input, expected)`. Правило rust-analyzer: «do not test the API».
- Для объёмного вывода (диагностика, AST, вывод CLI) используйте снапшоты: insta (`cargo insta review`) или более лёгкий expect-test.
- Для инвариантов парсеров и домена добавляйте property-тесты (proptest) с минимизацией контрпримеров.
- Сервисы тестируйте black-box: интеграционный тест поднимает приложение и ходит в него как клиент.
- Сидированный фаззинг с воспроизводимыми запусками хорошо ловит ошибки в движках с большим пространством состояний.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Delete Cargo Integration Tests](https://matklad.github.io/2021/02/27/delete-cargo-integration-tests.html) | matklad, 2021 | Почему один бинарник интеграционных тестов, `doctest = false`, тесты в отдельном файле |
| [How to Test](https://matklad.github.io/2021/05/31/how-to-test.html) | matklad, 2021 | Осознанная стратегия тестирования: data-driven тесты, тестировать фичи, а не реализацию |
| [rust-analyzer Architecture: Testing](https://rust-analyzer.github.io/book/contributing/architecture.html) | команда rust-analyzer | Снапшоты expect-test на границе `ide` |
| [insta](https://insta.rs/) | Armin Ronacher | Снапшот-тесты с ревью изменений |
| [proptest](https://github.com/proptest-rs/proptest) ([книга](https://proptest-rs.github.io/proptest/)) | proptest-rs | Property-based тесты с shrinking |
| [Ruff CONTRIBUTING](https://github.com/astral-sh/ruff/blob/main/CONTRIBUTING.md) | Astral | mdtest: тесты в Markdown со снапшотами, обновление через `MDTEST_UPDATE_SNAPSHOTS=1` |

## Разбиение на крейты и время сборки

### Ключевые выводы

- Крейт — единица параллелизма и инкрементальности компилятора. Один огромный крейт загружает одно ядро; широкий граф загружает все.
- Измеряйте, а не угадывайте: `cargo build --timings` показывает критический путь графа.
- Границы крейтов делайте не-generic: тонкая generic-обёртка вызывает конкретную внутреннюю функцию, `&dyn Fn` вместо `impl Fn`. Иначе мономорфизация переносит компиляцию в каждый зависимый крейт.
- Крейты с proc-macro и `syn` ставьте поздно в графе, serde держите ближе к краям системы.
- Меньше бинарников — меньше линковки: m тестовых бинарников на n крейтов линкуются m×n раз.
- Сгенерированный код дробите агрессивно: Feldera разбила ~100k LOC сгенерированного кода на ~1106 крейтов и ускорила сборку ~15x (с 30 до 2 минут). Рост `codegen-units` не помог. Цена — больше объектных файлов на линковку и повторные вычисления на крейт.
- Обновление тулчейна само по себе ускоряет сборку: с Rust 1.90 на x86_64 Linux по умолчанию используется LLD.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Fast Rust Builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html) | matklad, 2021 | Лучший концептуальный текст: форма графа, не-generic границы, proc-macro у листьев, `--timings` |
| [Cutting down Rust compile times from 30 to 2 minutes with one thousand crates](https://www.feldera.com/blog/cutting-down-rust-compile-times-from-30-to-2-minutes-with-one-thousand-crates) | Gerd Zellweger (Feldera), 2025 | Крайний случай дробления ради параллельного codegen ([HN](https://news.ycombinator.com/item?id=43715235), [lobste.rs](https://lobste.rs/s/b5ocbq/cutting_down_rust_compile_times_with_one)) |
| [Tips For Faster Rust Compile Times](https://corrode.dev/blog/tips-for-faster-rust-compile-times/) | Matthias Endler, живой | Полный чеклист: разбиение workspace, workspace-hack, линкеры, кэш |
| [Tips for Faster CI Builds](https://corrode.dev/blog/tips-for-faster-ci-builds/) | Matthias Endler, живой | Кэш, nextest, сборка в CI |
| [cargo-hakari](https://docs.rs/cargo-hakari/latest/cargo_hakari/about/index.html) | guppy-rs | Убирает пересборки из-за разной унификации фич |
| [Faster linking times with 1.90.0](https://blog.rust-lang.org/2025/09/01/rust-lld-on-1.90.0-stable) | Rémy Rakic, 2025 | LLD по умолчанию на x86_64 Linux; ручной `-fuse-ld=lld` там больше не нужен |

## ARCHITECTURE.md

### Ключевые выводы

- Держите в корне репозитория короткий ARCHITECTURE.md: вид с высоты птичьего полёта и codemap, отвечающий на вопрос «где то, что делает X?».
- Называйте важные файлы, модули и типы, но не ставьте ссылки: они устаревают, а символ находится поиском.
- Явно формулируйте архитектурные инварианты, особенно те, что выражены отсутствием чего-то («ядро не делает IO», «внутренние крейты не являются API-границей»).
- Отмечайте границы слоёв и добавьте раздел о сквозных аспектах (ошибки, логирование, отмена, тесты).
- Не синхронизируйте документ постоянно: перечитывайте и правьте пару раз в год. Codemap заодно показывает, лежат ли связанные вещи рядом.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [ARCHITECTURE.md](https://matklad.github.io/2021/02/06/ARCHITECTURE.md.html) | matklad, 2021 | Исходное описание практики |
| [rust-analyzer Architecture](https://rust-analyzer.github.io/book/contributing/architecture.html) | команда rust-analyzer | Развёрнутый живой пример такого документа |

## Кодовые базы для изучения

### Ключевые выводы

- Видны два стиля. «Много мелких плоских крейтов» (rust-analyzer, ruff, uv, zed) подходит приложениям и инструментам. «Фасад + подкрейты за фичами» (bevy, tokio, polars) подходит публикуемым библиотекам и фреймворкам.
- Изучайте не код целиком, а границы: что лежит в словарном крейте, кто зависит от фасада, где появляется IO.
- Сначала читайте документ об архитектуре или CONTRIBUTING, если он есть, потом дерево `crates/`.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [rust-analyzer](https://github.com/rust-lang/rust-analyzer) | rust-lang | Стек слоёв, фасадные крейты `hir`/`ide`, отмена через unwinding, снапшот-тесты на границе `ide`, сгенерированный код в репозитории |
| [ruff](https://github.com/astral-sh/ruff) | Astral | Плоский `crates/ruff_*` по зонам ответственности (`ruff_python_ast`, `ruff_python_parser`, `ruff_python_semantic`, `ruff_linter`, `ruff_python_formatter`), конвейер tokens → AST → semantic model, отдельные бинарники `ruff_wasm`, `ruff_dev`, `ruff_benchmark` |
| [uv](https://github.com/astral-sh/uv) | Astral | Та же плоская конвенция `crates/uv-*` (резолвер, установщик, кэш, клиент отдельно), обильные insta-снапшоты вывода CLI |
| [ripgrep](https://github.com/BurntSushi/ripgrep) | Andrew Gallant | CLI разложен на переиспользуемые библиотеки (`grep-searcher`, `grep-regex`, `grep-printer`, `ignore`, `globset`); бинарник — тонкий оркестратор |
| [tokio](https://github.com/tokio-rs/tokio) | tokio-rs | Библиотечный workspace (`tokio`, `tokio-macros`, `tokio-util`, `tokio-stream`, `tokio-test`), мелкие аддитивные фичи (`rt`, `net`, `macros`, `full`), proc-macro в отдельном крейте |
| [axum](https://github.com/tokio-rs/axum) | tokio-rs | Минимальное стабильное ядро `axum-core` для экосистемы, `axum-extra` для необязательного, tower `Service` как абстракция-порт |
| [bevy](https://github.com/bevyengine/bevy) | Bevy | Фасадный крейт поверх ~50+ `bevy_*`, фичи включают подсистемы, трейт `Plugin` как механизм композиции, ECS как архитектура |
| [polars](https://github.com/pola-rs/polars) | pola-rs | Слои `polars-core` / `polars-plan` / `polars-expr` / `polars-lazy` / `polars-io`: планирование запросов отделено от исполнения, биндинги на краю, фичи против долгой компиляции |
| [helix](https://github.com/helix-editor/helix) | helix-editor | Functional core / imperative shell: `helix-core` без IO, `helix-view`, `helix-term`, `helix-lsp` |
| [nushell](https://github.com/nushell/nushell) | nushell | Словарный крейт `nu-protocol`, `nu-engine`, `nu-parser`, `nu-command`; граница плагинов через сериализованный протокол |
| [tikv](https://github.com/tikv/tikv) | tikv | Большой распределённый workspace (`components/` вместо `crates/`), `engine_traits` абстрагирует хранилища — ports & adapters в масштабе |
| [deno](https://github.com/denoland/deno) | denoland | Бинарник `cli/`, `runtime/`, extension-крейты `ext/*` поверх V8 |
| [zed](https://github.com/zed-industries/zed) | Zed Industries | Очень большой workspace приложения со своим GPU UI-фреймворком `gpui`; оценки размера расходятся: 170–231 крейт, ~1.3M LOC [не проверено] ([factory.ai](https://factory.ai/open-source-wikis/zed)) |

## Где источники расходятся

### Ключевые выводы

- Почти все разногласия — это компромисс между временем сборки, скоростью выполнения и церемониями. Выбирайте по размеру проекта, а не по моде.
- Если сомневаетесь, берите вариант по умолчанию из таблицы и пересматривайте его, когда появится измеримая проблема.

| Вопрос | Позиция A | Позиция B | Рекомендация по умолчанию |
|---|---|---|---|
| Generics или `dyn` на границах | howtocodeit: порты внедряются через generics ради скорости и ошибок на этапе типов ([гайд](https://www.howtocodeit.com/guides/master-hexagonal-architecture-in-rust)) | matklad: границы крейтов без generics, `&dyn Fn` вместо `impl Fn`, ради времени сборки ([Fast Rust Builds](https://matklad.github.io/2021/09/04/fast-rust-builds.html)) | Generics внутри крейта или ядра приложения; `dyn` или конкретные типы на границах крейтов в большом workspace. В горячих путях — generics или enum dispatch |
| Раскладка интеграционных тестов | Rust Book: файл на тему в `tests/*.rs` | matklad: один бинарник `tests/it/main.rs`, у Cargo это дало 3x ([статья](https://matklad.github.io/2021/02/27/delete-cargo-integration-tests.html)) | Один бинарник на крейт. Zero To Production поступает так же |
| Где живут unit-тесты | corrode: рядом с кодом ([статья](https://corrode.dev/blog/long-term-rust-maintenance/)) | matklad: `mod tests;` в отдельном файле, чтобы правка теста не пересобирала библиотеку | Inline `mod tests { ... }` в небольших крейтах; отдельный файл, когда крейт большой и сборка заметна |
| Сколько крейтов | matklad: плоско, иерархия только после ~100 крейтов | Feldera: 1000+ крейтов ради параллельного codegen ([статья](https://www.feldera.com/blog/cutting-down-rust-compile-times-from-30-to-2-minutes-with-one-thousand-crates)) | Делить по зонам ответственности, когда граница стабильна или сборка упирается в один крейт. Массовое дробление — для сгенерированного кода; следить за линковкой и унификацией фич (hakari) |
| Гексагональная архитектура | corrode: изучать ради низкой связности ([статья](https://corrode.dev/blog/successful-rust-business-adoption-checklist/)) | howtocodeit: избыточна для CRUD и горячего кода | Применять, когда домен богаче транспорта или хранилищ несколько. Для CRUD и утилит — модули `lib.rs` + тонкий `main.rs` без портов |
| Гранулярность ошибок | Один enum ошибок на крейт (частый приём в старых туториалах) | Jewson: тип на операцию ([статья](https://sabrinajewson.org/blog/errors)); iroh: контекст в стиле snafu | Тип на операцию или небольшую группу операций; большой общий enum — лёгкий запах |
| Стиль workspace | «Много мелких плоских крейтов» (ruff, uv, rust-analyzer) | «Фасад + подкрейты за фичами» (bevy, tokio, polars) | Приложения и инструменты — плоско; публикуемые библиотеки — фасад с аддитивными фичами |

## Чеклист: структура нового проекта

### Ключевые выводы

- Начинайте с самого простого уровня и переходите на следующий по сигналу (размер, время сборки, число команд), а не заранее.
- Переход между уровнями дешевле, если с самого начала логика лежит в `lib.rs`, а `main.rs` тонкий.

| Уровень | Когда | Что сделать |
|---|---|---|
| Один крейт | до ~10k LOC, одна команда, сборка не мешает | `lib.rs` с логикой и тонкий `main.rs`. Модули по предметной области, внутри `pub(crate)`. Доменные newtype и ошибки на операцию (thiserror); в `main` — anyhow с контекстом. Интеграционные тесты одним бинарником `tests/it/main.rs`. Конфигурация и tracing-subscriber только в `main`. Lint в `[lints]` в `Cargo.toml` |
| Небольшой workspace | ~10k–100k LOC, 3–15 крейтов; компиляция начала мешать или появился второй бинарник | Виртуальный манифест, `members = ["crates/*"]`, явный `resolver`. Словарный крейт типов, ядро без IO, крейты-адаптеры (БД, HTTP, CLI), тонкие бинарники. `[workspace.dependencies]`, `[workspace.package]`, `[workspace.lints]`. Фичи только аддитивные. Короткий ARCHITECTURE.md. По одному бинарнику интеграционных тестов на крейт. Порты-трейты — только если домен богаче транспорта |
| Большой workspace | >100k LOC, десятки крейтов, несколько команд | Широкий граф по `cargo build --timings`; не-generic границы крейтов; proc-macro и serde ближе к краям. Явные API-границы с инвариантами в ARCHITECTURE.md; фасадные крейты с курированным `pub` и `#[non_exhaustive]`. `doctest = false` и тесты в отдельных файлах на внутренних крейтах; снапшоты и property-тесты. cargo-hakari при пересборках из-за фич; `cargo tree --duplicates` в ревью зависимостей. Иерархию каталогов вводить только после ~100 крейтов |
| Публикуемая библиотека (на любом уровне) | крейт пойдёт на crates.io | Фасадный крейт, proc-macro отдельно, аддитивные фичи (`std`, а не `no_std`), `#[non_exhaustive]` и sealed traits по API Guidelines, типизированные ошибки, MSRV и semver-дисциплина (release-plz) |
