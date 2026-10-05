---
name: rust-knowledge
description: Knowledge base and workflow for writing, reviewing and optimizing Rust code in any domain — libraries, CLI tools, web backends, async services, embedded/no_std, WebAssembly, FFI, data/ML, GUI, games. Use when writing or reviewing Rust, structuring a crate or Cargo workspace, choosing crates or cargo tooling (clippy, nextest, cargo-deny, Miri, criterion, samply), benchmarking, profiling or optimizing Rust performance, working with async/concurrency/unsafe, or picking Rust books and articles. Includes a curated library of books, articles and patterns with links, a toolchain guide, and a development pipeline with benchmarks, code review and evidence-based optimization.
---

# Rust: библиотека знаний и пайплайн

Скилл помогает писать, проверять и ускорять Rust-код в любой предметной области. Внутри три вещи:

1. **Библиотека знаний.** Книги, статьи, доклады и паттерны по категориям. Каждый источник дан со ссылкой, автором, годом и пояснением, зачем его читать.
2. **Тулчейн.** Какие инструменты ставить, какие команды запускать и в каком порядке.
3. **Пайплайн.** Порядок работы: тесты до кода, проверки качества, бенчмарки, ревью, поиск решений в библиотеке и повторный замер.

Состояние знаний — октябрь 2026: стабильный Rust 1.99, edition 2024.

## Куда смотреть

| Задача | Файл |
| --- | --- |
| Новая фича, рефакторинг или оптимизация: порядок работы | [references/pipeline.md](references/pipeline.md) |
| Какие инструменты поставить, команды, CI, `[workspace.lints]`, профили сборки | [references/toolchain.md](references/toolchain.md) |
| Код медленный: найти технику по симптому, профилирование, бенчмарки | [references/performance.md](references/performance.md) — начните с таблицы «Симптом → техника → источник» |
| Как выразить идею типами, выбор между generics, `dyn` и `enum`, ошибки, чеклист ревью | [references/idioms-patterns.md](references/idioms-patterns.md) |
| Структура крейтов и workspace, слои, тесты, feature flags | [references/architecture.md](references/architecture.md) |
| Потоки, async, блокировки, каналы, атомики, `unsafe`, FFI | [references/concurrency-async-unsafe.md](references/concurrency-async-unsafe.md) |
| Особенности домена: web, CLI, библиотеки, embedded, WASM, interop, данные, GUI, игры, системный код | [references/domains.md](references/domains.md) |
| Что почитать, маршрут обучения по уровням | [references/books.md](references/books.md) |

Каждый справочник начинается с оглавления. Читайте только нужный раздел, а не файл целиком.

## Как работать

**Для задачи больше одной функции** идите по [пайплайну](references/pipeline.md):

1. Опрос.
2. План и спецификация.
3. Подзадачи.
4. Тесты до кода.
5. Изоляция в worktree.
6. Бенчмарки.
7. Базовый замер и профиль.
8. Ревью.
9. Поиск решений в библиотеке.
10. Оптимизация.
11. Повторный замер.

Для мелкой правки хватает короткого режима из того же файла: тест, код, проверки, ревью.

**Перед тем как объявить работу готовой**, прогоните минимальные проверки и покажите их вывод:

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo nextest run --workspace --all-features   # или cargo test
cargo test --doc --workspace
```

Если в изменении есть `unsafe`, добавьте `cargo +nightly miri test`. Для публикуемой библиотеки добавьте `cargo semver-checks`. Если установлен cargo-deny, добавьте `cargo deny check`. Инструмент может быть не установлен: тогда скажите об этом и предложите команду установки из [toolchain.md](references/toolchain.md), а не пропускайте шаг молча.

**При оптимизации:**

- Сначала замер, потом правка. Без базовой линии нельзя сказать, что код стал быстрее.
- Ищите технику по симптому в [performance.md](references/performance.md) и указывайте источник рядом с правкой.
- Принимайте правку, только если бенчмарк показал значимый выигрыш, а тесты остались зелёными.

**При ревью** используйте чеклисты:

- [idioms-patterns.md](references/idioms-patterns.md) — для любого кода;
- [concurrency-async-unsafe.md](references/concurrency-async-unsafe.md) — для конкурентного и `unsafe`-кода.

## Правила использования библиотеки

- **Ссылайтесь на источник.** Рекомендуя паттерн или оптимизацию, давайте ссылку из справочника. Тогда пользователь может проверить совет, а решение не выглядит выдуманным.
- **Учитывайте пометки.** `[устарело]` значит, что у материала есть замена, и она указана рядом. `[не проверено]` значит, что утверждение взято из одного вторичного источника; его стоит перепроверить, прежде чем строить на нём решение.
- **Проверяйте версии.** Версии крейтов и статус стабилизаций указаны на октябрь 2026. Если проект использует другие версии или с тех пор вышли новые, сверьтесь с crates.io, docs.rs и release notes Rust.
- **Говорите о пробелах прямо.** Если в библиотеке нет ответа, так и скажите, а затем ищите в официальной документации. Не выдавайте догадку за материал библиотеки.
- **Следуйте соглашениям проекта.** Если в проекте уже приняты стиль, крейты или архитектура, советы библиотеки применяются в их рамках, а не поверх.
