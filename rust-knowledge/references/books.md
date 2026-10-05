# Книги по Rust: каталог и маршрут чтения

Аннотированный каталог книг по Rust для тех, кто пишет быстрый, идиоматичный и хорошо устроенный код. Здесь отмечены обязательные книги, полка «по ситуации» и бесплатные онлайн-книги. Указано, какие издания обновлены под edition 2024, а какие старше и что в них по-прежнему верно. В конце дан маршрут чтения по уровням. Состояние на октябрь 2026 года: стабильный Rust 1.90+, edition 2024 стабильна с Rust 1.85 (февраль 2025).

- [Как выбирать книгу](#как-выбирать-книгу)
- [Must-read: восемь книг](#must-read-восемь-книг)
- [Бесплатные онлайн-книги и официальная документация](#бесплатные-онлайн-книги-и-официальная-документация)
- [Полка «по ситуации»](#полка-по-ситуации)
- [Edition 2024: что обновлено, а что старше](#edition-2024-что-обновлено-а-что-старше)
- [Маршрут чтения по уровням](#маршрут-чтения-по-уровням)
- [Каталоги и навигаторы](#каталоги-и-навигаторы)

## Как выбирать книгу

### Ключевые выводы

- Большая часть нужного бесплатна: Rust Performance Book, Rust API Guidelines, Effective Rust, Rust Atomics and Locks и Rustonomicon доступны онлайн. Из платных стоит покупать в первую очередь Rust for Rustaceans и Programming Rust 3e.
- Больше всего пользы на страницу для производительности дают Performance Book и Rust Atomics and Locks. Rustonomicon и Rust Reference нужны, как только появляется unsafe или код зависит от layout.
- Книга «до edition 2024» редко ошибается в идиомах и производительности: изменения edition невелики. Стареет в первую очередь экосистема: API крейтов (actix и axum, версии tokio, sqlx) и async-возможности, стабилизированные после 2021 года.
- Async-главы любой книги до 2024 года дополняйте свежими источниками: в них нет async fn и RPITIT в трейтах (1.75), async-замыканий и `AsyncFn*` (1.85).
- Свежие книги по async читайте вместе с документацией Tokio, а не вместо неё.
- Каждое классическое эссе или книгу до 2021 года полезно читать в паре со свежим текстом: Effective Rust, corrode.dev или [Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/).

## Must-read: восемь книг

### Ключевые выводы

- Rust for Rustaceans остаётся главной «второй-третьей» книгой. Второго издания нет, поэтому изменения edition 2024 применяйте сами.
- Effective Rust — самая свежая книга жанра «как писать хорошо» (2024). Её 35 пунктов удобно использовать как чеклист на ревью.
- Performance Book — единая точка входа по скорости. Читайте целиком, она короткая.
- API Guidelines — чеклист для любого публичного API, включая внутренние крейты workspace.
- Programming Rust 3e — подробный справочник, подтверждённо обновлённый под edition 2024.
- Rustonomicon читайте до первого блока `unsafe`, а не после.
- Zero To Production — лучший разбор архитектуры сервиса целиком, хотя книга на actix-web и 2022 года.

| Ресурс | Автор, год | Издание, доступ | Уровень | Зачем читать: главы для производительности, паттернов, архитектуры |
|---|---|---|---|---|
| [Rust for Rustaceans](https://nostarch.com/rust-rustaceans) | Jon Gjengset, 2021 | No Starch, 1-е издание (единственное, [Goodreads](https://www.goodreads.com/work/editions/91291890-rust-for-rustaceans)); платно | продвинутый | Гл. 2 Types: layout, alignment, `repr`, DST, wide pointers, generics против trait objects и цена мономорфизации. Гл. 3 Designing Interfaces: «неудивительные», гибкие, очевидные, ограниченные API. Гл. 4 Error Handling. Гл. 5 Project Structure: фичи, workspaces, MSRV. Гл. 6 Testing: ловушки бенчмарков, `black_box`. Гл. 9 Unsafe, 10 Concurrency, 11 FFI, 12 `no_std`. Гл. 8 Async устарела частично |
| [Effective Rust](https://effective-rust.com/) | David Drysdale, апрель 2024 | O'Reilly ([Amazon](https://www.amazon.com/Effective-Rust-Specific-Ways-Improve/dp/1098151402)); бесплатно онлайн | средний | Шесть глав: Types, Traits, Concepts, Dependencies, Tooling, Beyond Standard Rust. Ключевые пункты: [Item 6 newtype](https://effective-rust.com/newtype.html), [Item 7 builders](https://effective-rust.com/builders.html), [Item 8 references и interior mutability](https://effective-rust.com/references.html), [Item 9 iterators](https://effective-rust.com/iterators.html), [Item 11 RAII](https://effective-rust.com/raii.html), [Item 12 generics против trait objects](https://effective-rust.com/generics.html). Глава Dependencies: semver и feature flags. Tooling: Clippy, fuzzing |
| [The Rust Performance Book](https://nnethercote.github.io/perf-book/) | Nicholas Nethercote и др., с 2020, живой | бесплатно | средний | Почти всё про скорость: Benchmarking, Build Configuration (LTO, `codegen-units`, PGO, аллокатор), Profiling, Inlining, Hashing, Heap Allocations (`with_capacity`, `SmallVec`, `Cow`, переиспользование), Type Sizes, Standard Library Types, Iterators, Bounds Checks, I/O, Wrapper Types, Machine Code, Parallelism, Compile Times |
| [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html) | Rust libs team, ~2017–2019 | бесплатно; дата последнего обновления не указана | средний | Именование `as_`/`to_`/`into_`, interoperability (общие трейты, `From`/`TryFrom`), predictability, flexibility, type safety (newtype, builder), dependability, debuggability, [future proofing](https://rust-lang.github.io/api-guidelines/future-proofing.html) (sealed traits, приватные поля). Основа для архитектуры библиотек и модулей |
| [Rust Atomics and Locks](https://mara.nl/atomics/) | Mara Bos, январь 2023 | O'Reilly; бесплатно онлайн (старый адрес marabos.nl перенаправляет на mara.nl) | продвинутый | Канонический текст про memory ordering. Атомики, spinlock, каналы и `Arc` своими руками, OS-примитивы (futex) для mutex, condvar, rwlock. Материал не зависит от edition |
| [Programming Rust, 3rd ed.](https://www.oreilly.com/library/view/programming-rust-3rd/9781098176228) | Jim Blandy, Jason Orendorff, Leonora F. S. Tindall, октябрь 2026 | O'Reilly, 748 стр., «fully updated for Rust's 2024 edition»; платно. Вышла ли печатная версия, не подтверждено | средний → продвинутый | Самый подробный «системный» справочник: ownership, borrowing, lifetimes, итераторы, коллекции, замыкания, макросы, конкурентность, async, unsafe и FFI, представление в памяти |
| [The Rustonomicon](https://doc.rust-lang.org/nomicon/) | Rust project, живой | бесплатно | продвинутый | Data layout, ownership и lifetimes изнутри, variance, drop check, неинициализированная память, `Vec` с нуля, FFI, атомики. Нужна перед любым unsafe и низкоуровневой оптимизацией |
| [Zero To Production In Rust](https://www.zero2prod.com/) | Luca Palmieri, 2022 (завершена в марте 2022) | самиздат, ~600 стр., 11 глав; платно | средний | Архитектура бэкенда целиком на примере API email-рассылки: модульная структура, black-box интеграционные тесты, доменные инварианты через типы («parse, don't validate»), обработка ошибок, tracing-телеметрия, CI/CD. На actix-web, есть порт на axum ([mattiapenati/zero2prod](https://github.com/mattiapenati/zero2prod)). Ревизии после 2022 года не найдено |

## Бесплатные онлайн-книги и официальная документация

### Ключевые выводы

- Из must-read бесплатны Performance Book, API Guidelines, Effective Rust, Rust Atomics and Locks и Rustonomicon. Вместе они закрывают производительность, идиомы, конкурентность и unsafe.
- Rust Reference используйте как справочник по `repr`, layout и undefined behavior, а не как книгу для чтения подряд.
- Async Book переписывается, авторы сами называют её «work in progress with much missing». Не опирайтесь на главы, не прошедшие переписывание.
- Живые документы не имеют «издания»: их актуальность зависит от сопровождения, а не от даты печати.
- Rust Design Patterns — единственный канонический источник с отдельным разделом антипаттернов. Глубина у страниц разная.

| Ресурс | Автор, год | Издание, доступ | Уровень | Зачем читать |
|---|---|---|---|---|
| [The Rust Programming Language](https://doc.rust-lang.org/book/) | Steve Klabnik, Carol Nichols, Chris Krycho, живой ([rust-lang/book](https://github.com/rust-lang/book)) | бесплатно; печатное 3e — см. полку «по ситуации» | начальный → средний | База. Для дальнейшей работы важны главы о трейтах и generics, умных указателях, конкурентности, async и unsafe. [Гл. 7](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html): пакеты, крейты, модули, видимость |
| [Rust by Example](https://doc.rust-lang.org/rust-by-example/) | Rust project, живой | бесплатно | начальный | Справочник по синтаксису и std в примерах. Не про производительность и архитектуру |
| [The Rust Reference](https://doc.rust-lang.org/reference/) | Rust project, живой | бесплатно | продвинутый | Нормативная семантика: type layout, атрибуты `repr`, behavior considered undefined, различия edition |
| [Rust Edition Guide](https://doc.rust-lang.org/edition-guide/) | Rust project, живой | бесплатно | средний | Что поменялось в edition 2024. Обязательно к прочтению вместе с любой книгой до 2025 года |
| [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/) | rust-lang, переписывается | бесплатно | средний | Futures, pinning, executors, async/await. Главы до переписывания [устарело]: дополняйте Tokio tutorial и документацией |
| [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) | rust-unofficial, обновлено 2026-09-21 | бесплатно | средний | Идиомы ([аргументы через заимствованные типы](https://rust-unofficial.github.io/patterns/idioms/coercion-arguments.html), [`mem::take`/`replace`](https://rust-unofficial.github.io/patterns/idioms/mem-replace.html), конструкторы, `Default`), паттерны (builder, newtype, RAII guards, strategy, visitor, fold, разбиение структуры ради независимых заимствований, FFI, [свой трейт для сложных bounds](https://rust-unofficial.github.io/patterns/patterns/structural/trait-for-bounds.html)), антипаттерны ([clone ради borrowck](https://rust-unofficial.github.io/patterns/anti_patterns/borrow_clone.html), [`#![deny(warnings)]`](https://rust-unofficial.github.io/patterns/anti_patterns/deny-warnings.html), [Deref-полиморфизм](https://rust-unofficial.github.io/patterns/anti_patterns/deref.html)) |
| [Tour of Rust's Standard Library Traits](https://github.com/pretzelhammer/rust-blog/blob/master/posts/tour-of-rusts-standard-library-traits.md) | pretzelhammer | бесплатно | средний | Исчерпывающий разбор `Deref`, `AsRef`/`Borrow`, `From`/`Into`, `Iterator`, `Send`/`Sync`. В том же блоге — «Common Rust Lifetime Misconceptions» |
| [Learning Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/) | Alexis Beingessner | бесплатно | средний | Почему связные структуры в Rust даются тяжело: `Box`/`Rc`/`RefCell`, сырые указатели, Miri, stacked borrows |
| [The Little Book of Rust Macros](https://veykril.github.io/tlborm/) | Daniel Keep, обновил Lukas Wirth | бесплатно | средний → продвинутый | Механика `macro_rules!`, гигиена, TT munchers, internal rules, push-down accumulation, введение в proc-macro |
| [Comprehensive Rust](https://google.github.io/comprehensive-rust/) | команда Google Android | бесплатно | начальный → средний | Многодневный курс для онбординга команды с углублениями в Android, Chromium, bare-metal, конкурентность. Мелковат для оптимизации |
| [Cargo Book: Features](https://doc.rust-lang.org/cargo/reference/features.html) / [Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) | Cargo team, живой | бесплатно | средний | Нормативный источник по фичам, workspaces, профилям. См. `architecture.md` |
| [Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/) | Microsoft, 2025 | бесплатно | средний | Не книга, а свод правил, с которыми согласится большинство разработчиков с опытом 3+ года. Есть сжатая версия для ИИ-ассистентов |

Также полезны без ссылок здесь: список lint Clippy, Rust Embedded Book и Rust Compiler Development Guide (для внутренностей компилятора).

## Полка «по ситуации»

### Ключевые выводы

- Берите эти книги под задачу, а не подряд: web, async, FFI-миграция, CLI, системное программирование, геймдев.
- Idiomatic Rust добротная, но менее каноничная, чем Rust for Rustaceans и Effective Rust.
- Async Rust получила смешанные отзывы. Используйте её как обзор, а детали сверяйте с документацией Tokio.
- Rust Under the Hood — ниша для интуиции про zero-cost: какой ассемблер генерируют конструкции языка.
- Книги 2021–2022 годов по прикладным темам (web, системы, CLI, геймдев) ценны подходом, но конкретные крейты в них могли смениться.

| Ресурс | Автор, год | Издание, доступ | Уровень | Зачем читать |
|---|---|---|---|---|
| [The Rust Programming Language, 3rd ed.](https://nostarch.com/rust-programming-language-3rd-edition) | Klabnik, Nichols, Krycho, март 2026 | No Starch, 624 стр., «built on the Rust 2024 Edition» ([Penguin Random House](https://www.penguinrandomhouse.com/books/790517/the-rust-programming-language-3rd-edition-by-steve-klabnik-carol-nichols-and-chris-krycho-with-contributions-from-the-rust-community/)); платно, онлайн бесплатно | начальный → средний | База на edition 2024: новая полная глава про async, Miri для анализа unsafe, современный tooling. Не книга про оптимизацию |
| [Idiomatic Rust: Code like a Rustacean](https://www.manning.com/books/idiomatic-rust) | Brenden Matthews, октябрь 2024 | Manning; платно ([интервью SE Radio 659](https://se-radio.net/2025/03/se-radio-659-brenden-matthews-on-idiomatic-rust/)) | средний | Fluent-интерфейсы, builder, неизменяемые структуры данных, функциональные паттерны, generics и трейты, антипаттерны |
| Code Like a Pro in Rust ([каталог Manning](https://www.manning.com/catalog/programming-languages-and-styles/system-programming/rust)) | Brenden Matthews, 2024 | Manning; платно | средний | Tooling, управление проектом, тестирование, async, главы по оптимизации |
| [Async Rust](https://www.oreilly.com/library/view/async-rust/9781098149086/) | Maxwell Flitton, Caroline Morton, 2024 | O'Reilly; платно | средний | Futures, executors, Tokio, async-серверы, потоки и каналы, ошибки в async, акторы. Отзывы около 3.5–3.8/5 [не проверено] ([pythonlib](https://pythonlib.ru/en/book1382)) |
| [Rust Under the Hood](https://www.amazon.com/Rust-Under-Hood-internals-generated/dp/B0D7FQB3DH) | Sandeep и Deepa Ahluwalia, июль 2024 | самиздат, 315 стр.; платно | продвинутый | Сгенерированный ассемблер для enum, строк, dispatch, рекурсии, замыканий, async/await. Отзывов мало |
| [Rust Web Programming, 3rd ed.](https://www.packtpub.com/en-us/product/rust-web-programming-9781835887769) | Maxwell Flitton, январь 2026 | Packt, 674 стр.; платно | средний | Actix, Axum, Rocket, Hyper; монолит против микросервисов и «nanoservices»; auth, БД, деплой в AWS/Terraform, тестирование; глава с паттернами для web. Ориентация на edition 2024 не подтверждена |
| [Refactoring to Rust](https://www.manning.com/books/refactoring-to-rust) | Lily Mara, Joel Holmes, 2025 | Manning; платно | средний | Поэтапная замена горячего кода на других языках через FFI, WebAssembly, Python-расширения |
| Rust in Action | Tim McNamara, сентябрь 2021 | Manning, 456 стр.; платно | средний | Системная интуиция: память, файлы, сеть, время, процессы, ядро. Не книга про идиомы |
| Hands-on Rust | Herbert Wolverson, 2021 | Pragmatic Bookshelf; платно | начальный → средний | Roguelike на ECS (Legion): интуиция data-oriented и ECS-архитектуры. Идеи важнее конкретного ECS-крейта |
| Command-Line Rust | Ken Youens-Clark, 2022 | O'Reilly; платно | средний | Клоны Unix-утилит с тестами: дисциплина тестирования и структура небольших программ |
| Rust Servers, Services, and Apps; Rust Web Development; Embedded Software with Rust ([каталог Manning](https://www.manning.com/catalog/programming-languages-and-styles/system-programming/rust)) | Eshwarla 2023; Gruber 2022; Cabanis, MEAP (выход ожидался в 2026) | Manning; платно | средний | Прикладные книги под web-сервисы и embedded |
| Programming Rust, 2nd ed. ([O'Reilly](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/)) | Blandy, Orendorff, Tindall, июнь 2021 | O'Reilly | средний | [устарело]: заменена 3-м изданием (2026), обновлённым под edition 2024 |

## Edition 2024: что обновлено, а что старше

### Ключевые выводы

- Подтверждённо обновлены под edition 2024 только две книги: TRPL 3e (март 2026) и Programming Rust 3e (октябрь 2026).
- Книги 2024 года (Effective Rust, Idiomatic Rust, Async Rust, Rust Under the Hood) написаны против edition 2021, потому что edition 2024 стабилизировали лишь в феврале 2025 года.
- Читая книгу до 2025 года, применяйте поправки edition 2024 сами: новые правила захвата lifetimes в RPIT, `unsafe extern`, ссылки на `static mut` теперь ошибка по умолчанию, `IntoIterator` для boxed slices.
- В async-главах старых книг замените `Box<dyn Fn() -> Pin<Box<dyn Future>>>` и `#[async_trait]` (где не нужен `dyn`) на нативный async fn в трейтах и `F: AsyncFn()`.
- Старые примеры могут не скомпилироваться на текущем тулчейне, это прямо отмечают обзоры 2026 года ([computingforgeeks](https://computingforgeeks.com/best-rust-programming-books-to-read/)).
- Живые документы (Performance Book, Rustonomicon, Reference, Design Patterns) обновляются непрерывно, а для Effective Rust онлайн и API Guidelines обновление под edition 2024 не подтверждено.

| Ресурс | Автор, год | Статус edition | Что по-прежнему верно | Что дополнить свежим |
|---|---|---|---|---|
| [TRPL 3e](https://nostarch.com/rust-programming-language-3rd-edition) | Klabnik, Nichols, Krycho, 2026 | edition 2024 | Всё | — |
| [Programming Rust 3e](https://www.oreilly.com/library/view/programming-rust-3rd/9781098176228) | Blandy, Orendorff, Tindall, 2026 | edition 2024 | Всё | — |
| [Rust for Rustaceans](https://nostarch.com/rust-rustaceans) | Gjengset, 2021 | до 2024 | Семантика, layout, дизайн интерфейсов, unsafe, тестирование, структура проекта: edition 2024 мало что поменяла | Гл. 8 Async: AFIT и RPITIT (1.75), async-замыкания (1.85); поправки edition 2024 |
| [Effective Rust](https://effective-rust.com/) | Drysdale, 2024 | edition 2021 | Все 35 пунктов про типы, трейты, зависимости, tooling | Поправки edition 2024; `[workspace.lints]` вместо `#![deny(...)]` в корне крейта |
| [Rust Atomics and Locks](https://mara.nl/atomics/) | Bos, 2023 | до 2024, не зависит от edition | Memory ordering и примитивы целиком | Strict provenance (1.84) для работы с указателями |
| [Zero To Production](https://www.zero2prod.com/) | Palmieri, 2022 | до 2024 | Структура сервиса, тестовая стратегия, ошибки, телеметрия, «parse, don't validate» | Версии крейтов; порт на axum |
| Idiomatic Rust, Async Rust, Rust Under the Hood | 2024 | edition 2021 | Паттерны и интуиция | Async-возможности 1.85+, поправки edition 2024 |
| Rust in Action, Hands-on Rust, Command-Line Rust | 2021–2022 | до 2024 | Подход и системная интуиция | Конкретные крейты и их API |
| [Async Book](https://rust-lang.github.io/async-book/) | rust-lang | переписывается | Переписанные главы | Непереписанные главы [устарело] |

## Маршрут чтения по уровням

### Ключевые выводы

- Не перепрыгивайте ступени: Rust for Rustaceans без уверенного владения трейтами и lifetimes читается тяжело.
- На каждом уровне сочетайте книгу-основу с чеклистом (API Guidelines, Effective Rust), который можно применять к своему коду сразу.
- Performance Book читайте только после идиоматичности: большая часть выигрыша в скорости приходит от устранения лишней работы, а не от микротюнинга.
- Специализацию выбирайте по задаче, параллельно с продвинутым уровнем.

| Ступень | Что читать | Зачем |
|---|---|---|
| 1. Начальный: фундамент | [TRPL](https://doc.rust-lang.org/book/) (3e или онлайн), [Rust by Example](https://doc.rust-lang.org/rust-by-example/) как справочник, затем [Tour of std traits](https://github.com/pretzelhammer/rust-blog/blob/master/posts/tour-of-rusts-standard-library-traits.md) и [Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/); для команды — [Comprehensive Rust](https://google.github.io/comprehensive-rust/) | Ownership, трейты, умные указатели, конкурентность. Понять, почему графы объектов в Rust даются тяжело |
| 2. Средний: идиоматичность | [Effective Rust](https://effective-rust.com/) целиком, [API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html) как чеклист, [Rust Design Patterns](https://rust-unofficial.github.io/patterns/), затем эссе: [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/), [Typestate](https://cliffle.com/blog/rust-typestate/), [Error Handling In Rust](https://www.lpalmieri.com/posts/error-handling-rust/), [Modular Errors](https://sabrinajewson.org/blog/errors), [Practical Rust API Design](https://corrode.dev/blog/practical-rust-api-design/) (2026) | Писать код, который выглядит и ведёт себя как в хороших крейтах. Состояние в типах, ошибки по слоям |
| 3. Продвинутый: глубина | [Rust for Rustaceans](https://nostarch.com/rust-rustaceans) (гл. 2, 3, 5, 6 в первую очередь), [Programming Rust 3e](https://www.oreilly.com/library/view/programming-rust-3rd/9781098176228) как справочник, [Rust Reference](https://doc.rust-lang.org/reference/) по мере надобности | Layout, цена абстракций, дизайн интерфейсов, структура проекта |
| 4. Продвинутый: производительность и сборка | [Performance Book](https://nnethercote.github.io/perf-book/) от начала до конца, затем статьи о bounds checks, сборке и SIMD (см. `performance.md`) | Порядок: измерить, устранить работу, сократить аллокации, затем флаги сборки |
| 5. Продвинутый: конкурентность и unsafe | [Rust Atomics and Locks](https://mara.nl/atomics/), [Rustonomicon](https://doc.rust-lang.org/nomicon/) до первого unsafe | Memory ordering, корректные примитивы, инварианты unsafe |
| Специализация: бэкенд | [Zero To Production](https://www.zero2prod.com/), [Rust Web Programming 3e](https://www.packtpub.com/en-us/product/rust-web-programming-9781835887769) | Архитектура сервиса, тесты, телеметрия |
| Специализация: async | [Async Book](https://rust-lang.github.io/async-book/) (переписанные главы), [Async Rust](https://www.oreilly.com/library/view/async-rust/9781098149086/) + документация Tokio | Futures, executors, акторы, отмена |
| Специализация: FFI и миграция | [Refactoring to Rust](https://www.manning.com/books/refactoring-to-rust), гл. 11 Rust for Rustaceans, FFI-главы Rustonomicon | Встраивание Rust в существующий код |
| Специализация: макросы | [The Little Book of Rust Macros](https://veykril.github.io/tlborm/), гл. 7 Rust for Rustaceans | `macro_rules!` и proc-macro |
| Специализация: низкий уровень | [Rust Under the Hood](https://www.amazon.com/Rust-Under-Hood-internals-generated/dp/B0D7FQB3DH), Rust in Action, гл. 12 `no_std` Rust for Rustaceans | Ассемблер, системные интерфейсы, embedded |
| Специализация: CLI, геймдев | Command-Line Rust; Hands-on Rust | Тестовая дисциплина малых программ; data-oriented и ECS |
| Архитектура (параллельно со ступенями 3–5) | См. `architecture.md`: серия matklad «One Hundred Thousand Lines of Rust», architecture doc rust-analyzer, hexarch | Устройство кодовой базы и workspace |

## Каталоги и навигаторы

### Ключевые выводы

- Для поиска книги под узкую тему начинайте с Little Book of Rust Books: он делит книги на официальные, неофициальные и прикладные.
- Списки «лучших книг» проверяйте по странице издателя: даты и издания в агрегаторах часто ошибаются.

| Ресурс | Автор, год | Зачем читать |
|---|---|---|
| [Little Book of Rust Books](https://lborb.github.io/book/) | сообщество, живой | Каталог официальных, неофициальных и прикладных книг |
| [Learning Material for Idiomatic Rust](https://corrode.dev/blog/idiomatic-rust-resources/) | Matthias Endler, 2024 | Мета-список ресурсов по идиоматичному Rust |
| [Best Rust programming books](https://computingforgeeks.com/best-rust-programming-books-to-read/) | computingforgeeks, 2026 | Обзор книг с пометками об edition. Составлен до выхода Programming Rust 3e; дату TRPL 3e указывает неверно (издатель: март 2026) |
| [Каталог Rust-книг Manning](https://www.manning.com/catalog/programming-languages-and-styles/system-programming/rust) | Manning | Текущие и готовящиеся (MEAP) книги издательства |
